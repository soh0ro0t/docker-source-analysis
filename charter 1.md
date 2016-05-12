
#chapter 1：docker daemon启动过程
------
##00 序
docker daemon的功能是创建守护进程，保障docker服务正常运行。由两部分逻辑组成：第一，创建docker运行环境并启动守护进程；第二，服务docker client，接收和处理请求。

##目录

|序号|标题|
|:-|:-|
|  第一节  | daemon 配置初始化|
|  第二节  | 创建daemon所需的其他服务|
|  第三节  | 创建核心守护进程|
|  第四节  | 创建Middlewares|
|  第五节  | 创建Router|

|章节|标题|
|:-:|:-:|
|   第一章  | docker daemon启动过程|
|   第二章  | router mapping 集合|
|   第四章  | |
|   第五章  | |
|   第六章  | |

##内容
#### [1]. daemon 配置初始化
这部分功能在main.init()函数中实现，作用是初始化docker daemon的参数列表，解析用户启动参数。

**1.1** 首先定位main.NewDaemonCli()，功能是创建daemon支持的配置参数信息，形同“dackerd --help”所示数据。
```c
func NewDaemonCli() *DaemonCli {
	// TODO(tiborvass): remove InstallFlags?
	daemonConfig := new(daemon.Config)
	daemonConfig.LogConfig.Config = make(map[string]string)
	daemonConfig.ClusterOpts = make(map[string]string)

	if runtime.GOOS != "linux" {
		daemonConfig.V2Only = true
	}

	daemonConfig.InstallFlags(flag.CommandLine, presentInHelp)
	configFile := flag.CommandLine.String([]string{daemonConfigFileFlag}, defaultDaemonConfigFile, "Daemon configuration file")
	flag.CommandLine.Require(flag.Exact, 0)

	return &DaemonCli{
		Config:      daemonConfig,
		commonFlags: cliflags.InitCommonFlags(),
		configFile:  configFile,
	}
}
```
**1.2** 定位main.main()，先合并docker daemon的全部配置参数，然后解析用户启动daemon时的命令行参数，若是“--help”则调用flag.Usage()后结束，或是“--version”则调用showVersion()后结束，其他则调用核心函数daemonCli.start()进入主流程。
```
	flag.Merge(flag.CommandLine, daemonCli.commonFlags.FlagSet)
	//set flag Usage
	flag.Usage = func() {
        ......
	}
	//set flag shortUsage
	flag.CommandLine.ShortUsage = func() {
		fmt.Fprint(stderr, "\nUsage:\tdockerd [OPTIONS]\n")
	}
	//analysize cmd option
	if err := flag.CommandLine.ParseFlags(os.Args[1:], false); err != nil {
		os.Exit(1)
	}

	if *flVersion {
		showVersion()
		return
	}

	if *flHelp {
		flag.Usage()
		return
	}

	if !stop {
		err = daemonCli.start()
```
####[2] daemonCli.start()：创建daemon所需的其他服务

**2.1** 创建daemon启动时的配置信息，通过合并用户的命令行参数和配置文件的参数实现，检查配置文件中的参数是否与命令行参数有重合，如有则退出，无则将两者融合成最终的配置参数。

**2.2** 创建apiserver，用于监听并接收客户端请求数据，默认情况下的监听接口是unix:///var/run/docker.sock，如果存在TCP连接则进行严格的TLS检测。

**2.3** 创建registryservice，用于配置注册服务器的信息，包括公有服务器和私有服务器信息。

**2.4** 创建libcontainer，用于管理容器的各项具体操作事例。

```
	cliConfig, err := loadDaemonCliConfig(cli.Config, flags, cli.commonFlags, *cli.configFile)
    ......
	serverConfig := &apiserver.Config{
		Logging:     true,
		SocketGroup: cli.Config.SocketGroup,
		Version:     dockerversion.Version,
		EnableCors:  cli.Config.EnableCors,
		CorsHeaders: cli.Config.CorsHeaders,
	}
    ......
	api := apiserver.New(serverConfig)
	for i := 0; i < len(cli.Config.Hosts); i++ {
		var err error
		if cli.Config.Hosts[i], err = opts.ParseHost(cli.Config.TLS, cli.Config.Hosts[i]); err != nil {
        ......
		logrus.Debugf("Listener created for HTTP on %s (%s)", protoAddrParts[0], protoAddrParts[1])
		api.Accept(protoAddrParts[1], l...)
	}
	......
	registryService := registry.NewService(cli.Config.ServiceOptions)
	containerdRemote, err := libcontainerd.New(cli.getLibcontainerdRoot(), cli.getPlatformRemoteOptions()...)
	if err != nil {
		return err
	}
```
####[3]  daemon.Newdaemon()：创建核心守护进程
-------------------------------------------------------------------------------------
**3.1** 设置MTU的值：setDefaultMtu()
    
    如果config选项中存在此值则无需改动，否则设置为默认值1500。

**3.2** 检测网桥和CGROUP驱动：verifyDaemonSettings()
    
    首先，网卡和IP不能同时设置，因为网卡已经设置了IP，不能重复定义；其次，检测控制容器是否设置互联，如需设置容器连通性(不管连通或不连通)，则必须开启iptable功能；再者，对runtimeoptions中的cgroup driver判断，以及croupparent目录和cgroup driver判断；
    
**3.3** 检测是否设置禁用网桥的选项

**3.4** 检测dockerd的启动用户是否root，目前版本dockerd必须以root权限启动。

**3.5** 检测RemappedRoot属性：setupRemappedRoot()
    
    RemappedRoot属性是指映射docker host上的用户到container中作Root用户，该属性由dockerd启动时的"userns-remap"选项指定。例如，"./dockerd --uers-remap 1000:1000"意味着uid和gid为1000的用户会被重新映射，映射记录存储在/etc/subuid和/etc/subgid文件中，dockerd就是在该文件中查找特定用户映射后的结果，比如"thebeeman:100000:65535"，uid 1000对应thebeeman用户，thebeeman hostID范围是[100000,165535]；container ID范围是[0,65535]，其中container Root ID=0，此时host ID=100000。

**3.6** 通过获取rootUID和rootGID
    
    rootUID和rootGID其实就是Host ID，上步已详细讲解，从uidMaps和gidMaps中取出即可。

**3.7** 获取docker ROOT USER的主目录
    
    如果"--users-remap"参数不为空，创建目录"/var/lib/docker/rootUID:rootGID"，并设置config.root为该值；否则为空，则主目录位于"/var/lib/docker/rootUID:rootGID"。不设置该项时主目录就是"/var/lib/docker/rootUID:rootGID"。例如：
    
    启动命令为dockerd --raws-log，客户端执行docker images -a则是获取"/var/lib/docker/"目录下iamges的数据。
    
    启动命令为dockerd --raws-log  --users-remap=hostUID:hostGID, 客户端执行docker images -a则是获取"/var/lib/docker/rootUID:rootGID"目录下iamges的数据。

**3.8** docker 主目录下创建tmp目录
    
    设置环境变量 "TMPDIR=cofig.root/tmp"

**3.9** 初始化AppArmor配置
    
    首先检测系统是否支持apparmor选项，通过读取"/sys/module/apparmor/parameters/enabled"获知。然后，创建默认的AppArmor的配置文件"/etc/apparmor.d/docker"文件，并填充数据在里面。

**3.10** docker 主作目录下创建containers目录
    
    创建containers目录，用于记录和存储容器。

**3.11** 创建存储驱动的目录
     
     先通过"DOCKER_DRIVER"获取，否则通过config.GraphDriver项获得。然后调用graphdriver.New创建aufs目录下子目录diff、layers、mnt，再调用NewFSMetadataStore创建layerdb数据库文件，最后调用NewStoreFromGraphDriver()重新存储layerdb中的镜像层，加载image/aufs/layerdb目录下面的数据。

**3.12** 检测和配置EnableSelinuxSupport
     
     检测是否设置Selinux选项，是则进行系统配置；否则关闭系统配置。

**3.13** 创建LayerDownloader和LayerUploader

**3.14** 创建image/aufs/imagedb目录
    
     调用NewFSStoreBackend创建imagedb，重载imagedb记录的数据。

**3.15** 创建卷驱动目录
    
     创建volumes目录，然后加载卷驱动，具体怎么实现没搞懂。
  
**3.16** 加载信任的key
     
     受信任的key存储在/etc/docker/key.json

**3.17** 创建distribution目录

**3.18** 创建events

**3.19** 创建image/aufs/repositories.json
     
     标签存储仓库，就是docker images的数据。

**3.20** 文件系统中数据迁移

**3.21** 检测和配置advertise

**3.22** 初始化网络控制器

####[4]  创建Middlewares：initMiddlewares()

####[5]  创建router

   router用于匹配事件的处理函数，并进行事件分发。那么初始化router就是创建事件与其处理函数的maps，包括container、images、volume、build、systemrouter等。由于docker是基于网络服务提供对外接口，c/s采用Http协议进行通信，下面枚举客户端请求与处理函数的记录，客户端请求的数据都会经过router处理后分发到相应的处理流程中去。
