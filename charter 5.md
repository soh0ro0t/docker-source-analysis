#chapter 5：docker swarm mode身份认证
------
##00 序
**docker swarm mode** 是docker1.12版本中集成的docker swamkit的项目，是docker容器集群编排工具，能创建docker原生集群，实现节点添加、移除、更新，以及任务创建、分发、伸缩扩展、移除等功能，但是由于swarm默认采用了golang tls协议认证，所以安全性很高，同时swarm manager为每个subserver都配置了认证函数，所以想绕过认证实施攻击很困难，本篇主要讲讲身份认证的具体实现：

##01 内容
###1.1 manager
manager包含的subserver：ControlServer，ResourceAllocatorServer，DispatcherServer，CAServer，NodeCAServer，RaftServer，HealthServer，RaftMembershipServer等等。
它还为这些subserver设置了身份认证，通过调用ca.AuthorizeForwardedRoleAndOrg()对应的client的身份，方法就是获取client的tls 证书，判断证书OU域是否为“swarm-manager” or “swarm-worker”？

**1.1.1 subserver注册:**  
  
	baseControlAPI := controlapi.NewServer(m.raftNode.MemoryStore(), m.raftNode, m.config.SecurityConfig.RootCA())
	baseResourceAPI := resourceapi.New(m.raftNode.MemoryStore())
	healthServer := health.NewHealthServer()
	localHealthServer := health.NewHealthServer()

	authenticatedControlAPI := api.NewAuthenticatedWrapperControlServer(baseControlAPI, authorize)
	authenticatedResourceAPI := api.NewAuthenticatedWrapperResourceAllocatorServer(baseResourceAPI, authorize)
	authenticatedDispatcherAPI := api.NewAuthenticatedWrapperDispatcherServer(m.dispatcher, authorize)
	authenticatedCAAPI := api.NewAuthenticatedWrapperCAServer(m.caserver, authorize)
	authenticatedNodeCAAPI := api.NewAuthenticatedWrapperNodeCAServer(m.caserver, authorize)
	authenticatedRaftAPI := api.NewAuthenticatedWrapperRaftServer(m.raftNode, authorize)
	authenticatedHealthAPI := api.NewAuthenticatedWrapperHealthServer(healthServer, authorize)
	authenticatedRaftMembershipAPI := api.NewAuthenticatedWrapperRaftMembershipServer(m.raftNode, authorize)

	proxyDispatcherAPI := api.NewRaftProxyDispatcherServer(authenticatedDispatcherAPI, m.raftNode, ca.WithMetadataForwardTLSInfo)
	proxyCAAPI := api.NewRaftProxyCAServer(authenticatedCAAPI, m.raftNode, ca.WithMetadataForwardTLSInfo)
	proxyNodeCAAPI := api.NewRaftProxyNodeCAServer(authenticatedNodeCAAPI, m.raftNode, ca.WithMetadataForwardTLSInfo)
	proxyRaftMembershipAPI := api.NewRaftProxyRaftMembershipServer(authenticatedRaftMembershipAPI, m.raftNode, ca.WithMetadataForwardTLSInfo)
	proxyResourceAPI := api.NewRaftProxyResourceAllocatorServer(authenticatedResourceAPI, m.raftNode, ca.WithMetadataForwardTLSInfo)


**1.1.2 subserver身份认证：**

	authorize := func(ctx context.Context, roles []string) error {
		var (
			removedNodes []*api.RemovedNode
			clusters     []*api.Cluster
			err          error
		)

		m.raftNode.MemoryStore().View(func(readTx store.ReadTx) {
			clusters, err = store.FindClusters(readTx, store.ByName("default"))

		})

		// Not having a cluster object yet means we can't check
		// the blacklist.
		if err == nil && len(clusters) == 1 {
			removedNodes = clusters[0].RemovedNodes
		}

		// Authorize the remote roles, ensure they can only be forwarded by managers
		// 鉴定证书OU域是否是roles？
		_, err = ca.AuthorizeForwardedRoleAndOrg(ctx, roles, []string{ca.ManagerRole}, m.config.SecurityConfig.ClientTLSCreds.Organization(), removedNodes)
		return err
	}

###1.2 实例论证
例如，dispatcher模块的client发送session()请求到server，请求为：
	go func() {
		client := api.NewDispatcherClient(s.conn)

		stream, err = client.Session(sessionCtx, &api.SessionRequest{
			Description: description,
			SessionID:   s.sessionID,
		})
		if err != nil {
			errChan <- err
			return
		}

		msg, err = stream.Recv()
		errChan <- err
	}()
	
dispatcher模块的server接收后的处理步骤是先调用authorize()验证，然后进入session创建过程。
	
	func (p *authenticatedWrapperDispatcherServer) Session(r *SessionRequest, stream Dispatcher_SessionServer) error {
	
		if err := p.authorize(stream.Context(), []string{"swarm-worker", "swarm-manager"}); err != nil {
			return err
		}
		return p.local.Session(r, stream)
	}
	
验证发送请求的客户端证书是否为"swarm-worker" or "swarm-manager"实现身份认证。

