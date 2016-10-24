
#chapter 6：raft协议在docker swarm mode中的应用
------
##00 序
**raft**：我们先了解Consensus一致性这个概念，它是指多个服务器在状态达成一致，由于在分布式系统中，因为各种意外可能，有的服务器可能会崩溃或变得不可靠，它就不能和其他服务器达成一致状态。这样就需要一种Consensus协议，一致性协议是为了确保容错性，也就是即使系统中有一两个服务器宕机，也不会影响集群的处理过程。

**演示视频**：[raft](http://thesecretlivesofdata.com/raft)

##01 创建raft

	raftNode := raft.NewNode(newNodeOpts)

	opts := []grpc.ServerOption{
		grpc.Creds(config.SecurityConfig.ServerTLSCreds)}

	m := &Manager{
		config:      config,
		listeners:   listeners,
		caserver:    ca.NewServer(raftNode.MemoryStore(), config.SecurityConfig),
		dispatcher:  dispatcher.New(raftNode, dispatcherConfig),
		server:      grpc.NewServer(opts...),
		localserver: grpc.NewServer(opts...),
		raftNode:    raftNode,
		started:     make(chan struct{}),
		stopped:     make(chan struct{}),
	}
	
其中raftNode本身和raftNode.MemoryStore()是主要分析对象。
raftNode包含raftNode、cluster、raftStore等成员，其中raftNode是etcd/raft结构体，它是raft协议真正实现部分：

	type Node struct {
		raftNode raft.Node
		cluster  *membership.Cluster
	
		raftStore           *raft.MemoryStorage
		memoryStore         *store.MemoryStore
		Config              *raft.Config
		opts                NodeOptions
		reqIDGen            *idutil.Generator
		wait                *wait
		wal                 *wal.WAL
		snapshotter         *snap.Snapshotter
		campaignWhenAble    bool
		signalledLeadership uint32
		isMember            uint32
	
		// waitProp waits for all the proposals to be terminated before
		// shutting down the node.
		waitProp sync.WaitGroup
	
		confState     raftpb.ConfState
		appliedIndex  uint64
		snapshotIndex uint64
	
		ticker clock.Ticker
		doneCh chan struct{}
		// removeRaftCh notifies about node deletion from raft cluster
		removeRaftCh        chan struct{}
		removeRaftFunc      func()
		leadershipBroadcast *watch.Queue
	
		// used to coordinate shutdown
		// Lock should be used only in stop(), all other functions should use RLock.
		stopMu sync.RWMutex
		// used for membership management checks
		membershipLock sync.Mutex
	
		snapshotInProgress chan uint64
		asyncTasks         sync.WaitGroup
	
		// stopped chan is used for notifying grpc handlers that raft node going
		// to stop.
		stopped chan struct{}
	
		lastSendToMember map[uint64]chan struct{}
	}

raftNode.MemoryStore()是非常重要的结构，由go-memdb实现，保证了manager下面的subserver可以进行数据共享，它由
tableSecret、tableNetwork、tableNode、tableCluster、tableService、tableTask等多个表单构成，结构定义：
	
	type MemoryStore struct {
		// updateLock must be held during an update transaction.
		updateLock sync.Mutex
	
		memDB *memdb.MemDB
		queue *watch.Queue
	
		proposer state.Proposer
	}

##01 绑定raft

manager中部分subserver的定义与之相关，说明这些server只是将raftnode和MemoryStore外包，真正的数据和处理流程仍在raftnode。

	baseControlAPI := controlapi.NewServer(m.raftNode.MemoryStore(), m.raftNode, m.config.SecurityConfig.RootCA())
	baseResourceAPI := resourceapi.New(m.raftNode.MemoryStore())
	authenticatedRaftAPI := api.NewAuthenticatedWrapperRaftServer(m.raftNode, authorize)
	authenticatedRaftMembershipAPI := api.NewAuthenticatedWrapperRaftMembershipServer(m.raftNode, authorize)
	
	m.replicatedOrchestrator = orchestrator.NewReplicatedOrchestrator(s)
	m.constraintEnforcer = orchestrator.NewConstraintEnforcer(s)
	m.globalOrchestrator = orchestrator.NewGlobalOrchestrator(s)

##02 实例分析

这里AuthenticatedWrapperRaftServer和AuthenticatedWrapperRaftMembershipServer只使用了raftNode，未使用MemoryStore，说明这两个会真正操作到etcd/raft对象，首先不妨查看AuthenticatedWrapperRaftServer结构，所有相关代码如下：

	type authenticatedWrapperRaftServer struct {
		local     RaftServer
		authorize func(context.Context, []string) error
	}

	func NewAuthenticatedWrapperRaftServer(local RaftServer, authorize func(context.Context, []string) error) RaftServer {
		return &authenticatedWrapperRaftServer{
			local:     local,
			authorize: authorize,
		}
	}

	func (p *authenticatedWrapperRaftServer) ProcessRaftMessage(ctx context.Context, r *ProcessRaftMessageRequest) (*ProcessRaftMessageResponse, error) {

		if err := p.authorize(ctx, []string{"swarm-manager"}); err != nil {
			return nil, err
		}
		return p.local.ProcessRaftMessage(ctx, r)
	}

	func (p *authenticatedWrapperRaftServer) ResolveAddress(ctx context.Context, r *ResolveAddressRequest) (*ResolveAddressResponse, error) {

		if err := p.authorize(ctx, []string{"swarm-manager"}); err != nil {
			return nil, err
		}
		return p.local.ResolveAddress(ctx, r)
	}
它由authorize()和RaftServer组成，显然，前者实现身份认证，后者是消息处理接口。由于ProcessRaftMessage和ResolveAddress只有manager才会调用，所以调用p.authorize(ctx, []string{"swarm-manager"})判断发起申请的node的证书OU域是否为“swarm-manager”。如果worker调用该接口则无法通过认证，更进入不了后续处理流程。其中ProcessRaftMessage最终进入消息处理函数etcd/raft.(*raft).Step()。ResolveAddress则最终调用manager/state/raft/membership.(*Cluster).GetMember获取raftID对应的Addr。

其次，查看AuthenticatedWrapperRaftMembershipServer结构，相关代码如下：
	type authenticatedWrapperRaftMembershipServer struct {
		local     RaftMembershipServer
		authorize func(context.Context, []string) error
	}
	
	func NewAuthenticatedWrapperRaftMembershipServer(local RaftMembershipServer, authorize func(context.Context, []string) error) RaftMembershipServer {
		return &authenticatedWrapperRaftMembershipServer{
			local:     local,
			authorize: authorize,
		}
	}
	
	func (p *authenticatedWrapperRaftMembershipServer) Join(ctx context.Context, r *JoinRequest) (*JoinResponse, error) {
	
		if err := p.authorize(ctx, []string{"swarm-manager"}); err != nil {
			return nil, err
		}
		return p.local.Join(ctx, r)
	}
	
	func (p *authenticatedWrapperRaftMembershipServer) Leave(ctx context.Context, r *LeaveRequest) (*LeaveResponse, error) {
	
		if err := p.authorize(ctx, []string{"swarm-manager"}); err != nil {
			return nil, err
		}
		return p.local.Leave(ctx, r)
	}
同理，先认证发起请求的节点身份是否为“swarm-manager”，然后调用Join()或者Leave()，只需研究下Join()，Leave()执行流程相似：
	func (n *Node) Join(ctx context.Context, req *api.JoinRequest) (*api.JoinResponse, error) {
		// 检查远端的证书摘要，是否可以使用CA证书绕过验证？没有检查Role
		nodeInfo, err := ca.RemoteNode(ctx)
		if err != nil {
			return nil, err
		}
	
		fields := logrus.Fields{
			"node.id": nodeInfo.NodeID,
			"method":  "(*Node).Join",
			"raft_id": fmt.Sprintf("%x", n.Config.ID),
		}
		if nodeInfo.ForwardedBy != nil {
			fields["forwarder.id"] = nodeInfo.ForwardedBy.NodeID
		}
	
	
		// Find a unique ID for the joining member.
		var raftID uint64
		for {
			raftID = uint64(rand.Int63()) + 1
			if n.cluster.GetMember(raftID) == nil && !n.cluster.IsIDRemoved(raftID) {
				break
			}
		}
		
		... ...
		
		remoteAddr := req.Addr
	
		// If the joining node sent an address like 0.0.0.0:4242, automatically
		// determine its actual address based on the GRPC connection. This
		// avoids the need for a prospective member to know its own address.
	
		... ...
		
		if err := n.checkHealth(ctx, remoteAddr, 5*time.Second); err != nil {
			return nil, err
		}
		err = n.addMember(ctx, remoteAddr, raftID, nodeInfo.NodeID)
		if err != nil {
			log.WithError(err).Errorf("failed to add member %x", raftID)
			return nil, err
		}
	
		var nodes []*api.RaftMember
		for _, node := range n.cluster.Members() {
			nodes = append(nodes, &api.RaftMember{
				RaftID: node.RaftID,
				NodeID: node.NodeID,
				Addr:   node.Addr,
			})
		}
		log.Debugf("node joined")
	
		return &api.JoinResponse{Members: nodes, RaftID: raftID}, nil
	}

代码分成三部分：

1.获取nodeInfo.NodeID, raftID, remoteAddr

2.回发health请求到node，判断node的health状态是否为“SERVING”

3.添加节点，最终进入etcd/raft.(*raft).step函数处理消息，后又进入manager/state/raft/(*Node).registerNode将其注册到membership.cluster

当然这只是raft部分功能应用，更多的引用在manager/state/raft/raft.go中直接搜索“n.raftNode.”，例如 n.raftNode.Campaign()、n.raftNode.Stop、n.raftNode.Step()等处理函数。
