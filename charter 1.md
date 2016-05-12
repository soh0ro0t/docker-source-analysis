
#chapter 1：docker daemon启动过程
------
docker daemon由两部分逻辑组成：第一，创建docker运行环境并启动；第二，服务docker client，接收和处理客户端请求。具体步骤如下：

#### 1. daemon 配置初始化
这部分功能在main.init()函数中实现，作用是初始化docker daemon的参数列表，解析用户启动参数。

1.1 首先定位main.NewDaemonCli()，功能是创建daemon支持的配置参数信息，形同“dackerd --help”所示数据。
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
<i class="icon-chevron-sign-left"></i>1.2 定位main.main()，先合并docker daemon的全部配置参数，然后解析用户启动daemon时的命令行参数，若是“--help”则调用flag.Usage()后结束，或是“--version”则调用showVersion()后结束，其他则调用核心函数daemonCli.start()进入主流程。
```c
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
####2. daemonCli.start()
2.1 创建daemon启动时的配置信息，通过合并用户的命令行参数和配置文件的参数实现，检查配置文件中的参数是否与命令行参数有重合，如有则退出，无则将两者融合成最终的配置参数。

2.2 创建apiserver，用于监听并接收客户端请求数据，默认情况下的监听接口是unix:///var/run/docker.sock，如果存在TCP连接则进行严格的TLS检测。

2.3 创建registryservice，用于配置注册服务器的信息，包括公有服务器和私有服务器信息。

2.4 创建libcontainer，用于管理容器的各项具体操作事例。
```c
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
