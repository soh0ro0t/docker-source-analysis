# 第一章：docker daemon启动过程分析

------
docker daemon的功能包括两部分：第一，创建docker运行环境；第二，为docker client提供服务，接收并且处理请求。

从代码细节理解，daemon启动过程分为以下步骤：

## 1. daemon的配置初始化
这部分在initservice()函数中实现，即在main.main()运行前已执行，主要功能是初始化docker daemon的配置信息，使得daemonConfig获取相应的属性
值，在dockerd启动时供其使用。

1.1 首先定位到main.main(docker/cmd/dockerd/docker.go:main)，其核心是执行daemonCli.start()函数，daemonCli是main.NewDaemonCli分配得来。

1.2 然后定位到main.NewDaemonCli(docker/cmd/dockerd/docker.go:NewDaemonCli)
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
初始化dockerd的flags，通过docker daemon --help查看各种配置选项就能理解。
