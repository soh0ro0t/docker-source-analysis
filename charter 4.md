
#chapter 4：docker runc执行过程
------
##00 序
**docker runc** 是docker容器套件中真正的启动容器、执行用户指定的命令，采用简单朴实的方式实现了OCF标准，docker后端实际调用它完成指定任务。runc的前身就是docker libcontainer，通过演化后配置成libcontainer的轻型客户端。本质上讲，libcontainer才是docker系容器的核心，通过对namespace、cgroup、capabilities以及rootfs的管理“隔离”出执行环境。不同处是，runC相比docker更加轻量化，去除了docker包含的镜像、Volume等高级功能。显著特点如下：

- **1 实现容器标准化**
- **2 实现容器的最小化单元**
- **3 核心是libcontainer**


##01 目录

|序号|标题|
|:-:|:-:|
|   第一节  | rootfs|
|   第二节  | namespace|
|   第三节  | cgroups|
|   第四节  | capability|
|   第五节  | 进程环境变量|

##02 内容
###2.1.rootfs

创建“隔离”的系统环境，首先应该构造OCI-rootfs，生成全新的根文件系统。

**2.1.1 执行流程:**  
  
1. 下载镜像，导出，解压生成OCI-rootfs文件目录
1.   mount --bind xx/OCI-rootfs xx/OCI-rootfs 
1.   mount proc dev等等分区到OCI-rootfs下面
1.   pivot_root OCI-rootfs old-root

**2.1.2 挂载信息：**

------------------------------
|设备|路径|文件系统类型|参数|_|_|
|:-|:-||:-|:-||:-|:-|
| 设备 | 路径 | 文件系统类型 | 参数 | _ | _ | 
| :- | :- |  | :- | :- |  | :- | :- | 
|  /dev/disk/by-uuid/106049e0-381b-4944-972c-49ee8467863e  |  /  |  ext4  |  ro,relatime,errors=remount-ro,data=ordered  |  0  |  0  |
|  proc  |  /proc  |  proc  |  rw,relatime  |  0  |  0  | 
| tmpfs | /dev | tmpfs | rw,nosuid,size=65536k,mode=755 | 0 | 0 | 
| devpts | /dev/pts | devpts | rw,nosuid,noexec,relatime,gid=5,mode=620,ptmxmode=666 | 0 | 0 | 
| shm | /dev/shm | tmpfs | rw,nosuid,nodev,noexec,relatime,size=65536k | 0 | 0 | 
| mqueue | /dev/mqueue | mqueue | rw,nosuid,nodev,noexec,relatime | 0 | 0 | 
| sysfs | /sys | sysfs | ro,nosuid,nodev,noexec,relatime | 0 | 0 | 
| tmpfs | /sys/fs/cgroup | tmpfs | ro,nosuid,nodev,noexec,relatime,mode=755 | 0 | 0 | 
| systemd | /sys/fs/cgroup/systemd | cgroup | ro,nosuid,nodev,noexec,relatime,name=systemd | 0 | 0 | 
| cgroup | /sys/fs/cgroup/cpuset | cgroup | ro,nosuid,nodev,noexec,relatime,cpuset | 0 | 0 | 
| cgroup | /sys/fs/cgroup/cpu | cgroup | ro,nosuid,nodev,noexec,relatime,cpu | 0 | 0 | 
| cgroup | /sys/fs/cgroup/cpuacct | cgroup | ro,nosuid,nodev,noexec,relatime,cpuacct | 0 | 0 | 
| cgroup | /sys/fs/cgroup/blkio | cgroup | ro,nosuid,nodev,noexec,relatime,blkio | 0 | 0 | 
| cgroup | /sys/fs/cgroup/memory | cgroup | ro,nosuid,nodev,noexec,relatime,memory | 0 | 0 | 
| cgroup | /sys/fs/cgroup/devices | cgroup | ro,nosuid,nodev,noexec,relatime,devices | 0 | 0 | 
| cgroup | /sys/fs/cgroup/freezer | cgroup | ro,nosuid,nodev,noexec,relatime,freezer | 0 | 0 | 
| cgroup | /sys/fs/cgroup/net_cls | cgroup | ro,nosuid,nodev,noexec,relatime,net_cls | 0 | 0 | 
| cgroup | /sys/fs/cgroup/perf_event | cgroup | ro,nosuid,nodev,noexec,relatime,perf_event | 0 | 0 | 
| cgroup | /sys/fs/cgroup/net_prio | cgroup | ro,nosuid,nodev,noexec,relatime,net_prio | 0 | 0 | 
| cgroup | /sys/fs/cgroup/hugetlb | cgroup | ro,nosuid,nodev,noexec,relatime,hugetlb | 0 | 0 | 
| devpts | /dev/console | devpts | rw,nosuid,noexec,relatime,gid=5,mode=620,ptmxmode=000 | 0 | 0 | 
| proc | /proc/asound | proc | ro,nosuid,nodev,noexec,relatime | 0 | 0 | 
| proc | /proc/bus | proc | ro,nosuid,nodev,noexec,relatime | 0 | 0 | 
| proc | /proc/fs | proc | ro,nosuid,nodev,noexec,relatime | 0 | 0 | 
| proc | /proc/irq | proc | ro,nosuid,nodev,noexec,relatime | 0 | 0 | 
| proc | /proc/sys | proc | ro,nosuid,nodev,noexec,relatime | 0 | 0 | 
| proc | /proc/sysrq-trigger | proc | ro,nosuid,nodev,noexec,relatime | 0 | 0 | 
| tmpfs | /proc/kcore | tmpfs | rw,nosuid,size=65536k,mode=755 | 0 | 0 | 
| tmpfs | /proc/timer_stats | tmpfs | rw,nosuid,size=65536k,mode=755 | 0 | 0 | 
| tmpfs | /proc/sched_debug | tmpfs | rw,nosuid,size=65536k,mode=755 | 0 | 0 | 

##03 参考
- 容器运行时 	[opencontainers/runc](https://github.com/opencontainers/runc)
- 虚拟机运行时 [hyperhq/runv](https://github.com/hyperhq/runv)
- OCI测试 [huawei-openlab/oct](https://github.com/huawei-openlab/oct)
