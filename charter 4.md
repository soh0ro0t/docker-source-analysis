
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
|   第一节 |  rootfs|
|   第二节 |  namespace|
|   第三节 |  cgroups|
|   第四节 |  capability|
|   第五节 |  进程环境变量|

##02 内容
###2.1 rootfs

创建“隔离”的系统环境，首先应该构造OCI-rootfs，生成全新的根文件系统，里面包含基本的可执行文件和库文件等。

**2.1.1 执行流程:**  
  
1. 下载镜像，导出，解压生成OCI-rootfs文件目录
1. mount --bind xx/OCI-rootfs xx/OCI-rootfs 
1. mount proc dev等分区到OCI-rootfs下面
1. 创建/dev 目录下的字符设备
1. 创建/proc/self/fd/* 目录下的链接文件
1. pivot_root OCI-rootfs old-root

**2.1.2 挂载rootfs及所需文件系统：**

- mount --bind xx/rootfs xx/rootfs
- mount -t proc proc xx/rootfs/proc
- mount -t tmpfs tmpfs xx/rootfs/dev
- ...... 

下表是挂载完成后的全部节点：

------------------------------
| 设备        | 挂载点           | 文件系统类型  | 参数 | x | x |
|:------------- |:-------------|:-----|:-----|:-----|:-----|
|  /dev/disk/by-uuid/106049e0-381b-4944-972c-49ee8467863e |  / |  ext4 |  ro,relatime,errors=remount-ro,data=ordered |  0 |  0 |
|  proc |  /proc |  proc |  rw,relatime |  0 |  0 |  
|  tmpfs |  /dev |  tmpfs |  rw,nosuid,size=65536k,mode=755 |  0 |  0 |  
|  devpts |  /dev/pts |  devpts |  rw,nosuid,noexec,relatime,gid=5,mode=620,ptmxmode=666 |  0 |  0 |  
|  shm |  /dev/shm |  tmpfs |  rw,nosuid,nodev,noexec,relatime,size=65536k |  0 |  0 |  
|  mqueue |  /dev/mqueue |  mqueue |  rw,nosuid,nodev,noexec,relatime |  0 |  0 |  
|  sysfs |  /sys |  sysfs |  ro,nosuid,nodev,noexec,relatime |  0 |  0 |  
|  tmpfs |  /sys/fs/cgroup |  tmpfs |  ro,nosuid,nodev,noexec,relatime,mode=755 |  0 |  0 |  
|  systemd |  /sys/fs/cgroup/systemd |  cgroup |  ro,nosuid,nodev,noexec,relatime,name=systemd |  0 |  0 |  
|  cgroup |  /sys/fs/cgroup/cpuset |  cgroup |  ro,nosuid,nodev,noexec,relatime,cpuset |  0 |  0 |  
|  cgroup |  /sys/fs/cgroup/cpu |  cgroup |  ro,nosuid,nodev,noexec,relatime,cpu |  0 |  0 |  
|  cgroup |  /sys/fs/cgroup/cpuacct |  cgroup |  ro,nosuid,nodev,noexec,relatime,cpuacct |  0 |  0 |  
|  cgroup |  /sys/fs/cgroup/blkio |  cgroup |  ro,nosuid,nodev,noexec,relatime,blkio |  0 |  0 |  
|  cgroup |  /sys/fs/cgroup/memory |  cgroup |  ro,nosuid,nodev,noexec,relatime,memory |  0 |  0 |  
|  cgroup |  /sys/fs/cgroup/devices |  cgroup |  ro,nosuid,nodev,noexec,relatime,devices |  0 |  0 |  
|  cgroup |  /sys/fs/cgroup/freezer |  cgroup |  ro,nosuid,nodev,noexec,relatime,freezer |  0 |  0 |  
|  cgroup |  /sys/fs/cgroup/net_cls |  cgroup |  ro,nosuid,nodev,noexec,relatime,net_cls |  0 |  0 |  
|  cgroup |  /sys/fs/cgroup/perf_event |  cgroup |  ro,nosuid,nodev,noexec,relatime,perf_event |  0 |  0 |  
|  cgroup |  /sys/fs/cgroup/net_prio |  cgroup |  ro,nosuid,nodev,noexec,relatime,net_prio |  0 |  0 |  
|  cgroup |  /sys/fs/cgroup/hugetlb |  cgroup |  ro,nosuid,nodev,noexec,relatime,hugetlb |  0 |  0 |  
|  devpts |  /dev/console |  devpts |  rw,nosuid,noexec,relatime,gid=5,mode=620,ptmxmode=000 |  0 |  0 |  
|  proc |  /proc/asound |  proc |  ro,nosuid,nodev,noexec,relatime |  0 |  0 |  
|  proc |  /proc/bus |  proc |  ro,nosuid,nodev,noexec,relatime |  0 |  0 |  
|  proc |  /proc/fs |  proc |  ro,nosuid,nodev,noexec,relatime |  0 |  0 |  
|  proc |  /proc/irq |  proc |  ro,nosuid,nodev,noexec,relatime |  0 |  0 |  
|  proc |  /proc/sys |  proc |  ro,nosuid,nodev,noexec,relatime |  0 |  0 |  
|  proc |  /proc/sysrq-trigger |  proc |  ro,nosuid,nodev,noexec,relatime |  0 |  0 |  
|  tmpfs |  /proc/kcore |  tmpfs |  rw,nosuid,size=65536k,mode=755 |  0 |  0 |  
|  tmpfs |  /proc/timer_stats |  tmpfs |  rw,nosuid,size=65536k,mode=755 |  0 |  0 |  
|  tmpfs |  /proc/sched_debug |  tmpfs |  rw,nosuid,size=65536k,mode=755 |  0 |  0 |  

**2.1.3 创建/dev 目录下的字符设备：**

------------------------------
| 设备文件        | 权限           | 属组  |
|:------------- |:-------------|:-----|
|  /dev/null  |  rw  |  root  |
|  /dev/zero  |  rw  |  root  |
|  /dev/tty  |  rw  |  root  |
|  /dev/random  |  rw  |  root  |
|  /dev/urandom  |  rw  |  root  |
|  /dev/fuse  |  rw  |  root  |

**2.1.4 创建/proc/self/fd 目录下的符号链接：**

------------------------------
| 源路径        | 目的路径           |
|:------------- |:-------------|
|  /proc/self/fd  |  /dev/fd  |
|  /proc/self/fd/0   |  /dev/stdin  |
|  /proc/self/fd/1  |  /dev/stdout  |
|  /proc/self/fd/2  |  /dev/stderr  |



**2.1.5 更改根文件系统：**

- mkdir old-root

- pivot_root OCI-rootfs old-root


###2.2 namespace

Linux Namespace是Linux提供的一种**内核级别 系统环境 隔离**的方法。chroot()（通过修改根目录把用户jail到一个特定目录下），chroot提供了一种简单的隔离模式：chroot内部的文件系统无法访问外部的内容。Linux Namespace在此基础上，提供了对UTS、IPC、mount、PID、network、User等的隔离机制,目的是将容器中的进程放置于一个独立的系统环境中，与容器外的一切均隔离。

Linux Namespace 有如下种类，官方文档在这里《Namespace in Operation》


------------------------------
| 分类        | 系统调用参数           | 相关内核版本  |
|:------------- |:-------------|:-----|		
|  Mount |  CLONE_NEWNS	| Linux 2.4.19|
|  UTS | 	CLONE_NEWUTS	| Linux 2.6.19|
|  IPC | CLONE_NEWIPC	| Linux 2.6.19|
|  PID | CLONE_NEWPID	| Linux 2.6.24|
<<<<<<< HEAD
|  Network | CLONE_NEWNET	|始于Linux 2.6.24 完成于 Linux 2.6.29|
|  User	|CLONE_NEWUSER	|始于 Linux 2.6.23 完成于 Linux 3.8)|

**2.2.2 UTS Namespace:**

UTS命名空间能为容器设置新的主机名和域名，本质上就是调用sethostname()函数修改主机名，显示为你所需的任意字符串。它在网络上可被视作一个独立的节点而非宿主机上的一个进程。

调用clone()函数时加上“CLONE_NEWUTS”参数即可，如果不加则修改的是主机的名称，而非容器名称。

[**示例代码**](https://github.com/TheBeeMan/docker-source-analysis/blob/demo/UTS-demo.c)

**2.2.3 PID Namespace:**

系统所有进程是由不同的PID命名空间组成的，这些PID命名空间不是扁平的，而是具有层次结构，形成一个等级体系。最顶层的是系统初期创建的PID命名空间，然后由它创建的新的PID命名空间则是子节点，而原先的PID命名空间就成为了新的PID命名空间的父节点。父节点能够看到子节点中的所有进程，并通过信号等方式对它们进行通信。反过来，子节点的所有进程无权访问父节点命名空间中的任何内容，子节点也无法影响兄弟节点命名空间中的内容。

- **1 每个PID命名空间的第一个进程都为“1”，称为守护进程。**
- **2 低级别的PID命名空间无法影响同级或高级的PID命名空间。**
- **3 在新的PID命名空间中重新挂载“proc”文件系统，能查看其下的所有进程信息。**

[**示例代码**](https://github.com/TheBeeMan/docker-source-analysis/edit/demo/PID-demo.c)
http://lwn.net/Articles/533495/

**2.2.4 IPC Namespace:**

IPC命名空间其实与PID命名空间绑定在一起，只有相同的PID namespace中的进程才能进程通信，常用手段是信号量、消息队列等。

**2.2.5 NET Namespace:**

NET命名空间绑定了特定的网络资源，包括 ... ...，不能获取其他的NET命名空间。

**2.2.6 USER Namespace:**

USER命名空间使进程以特定的用户权限执行容器内进程，通过调用clone()添加"CLONE_NEWUSER"，再设置user-map，将容器内部用户和主机用户进行映射，从而控制执行容器内进程的UID和GID，使之行使很小的权限。值得注意的是，docker主要通过控制capabilities来限制容器内进程的权限，即使不设置user namespace，而是使用主机的root权限执行init进程仍然不会有太多安全问题，因为container启动时的capabilities进行了严格的控制，同时由于其他命名空间的配合，还设置了rootfs，很难发生穿透。除非发生穿透，使用“user namespace”能降低权限，否则用处确实不大。

=======
|  Network |  CLONE_NEWNET	|始于Linux 2.6.24 完成于 Linux 2.6.29|
|  User	|CLONE_NEWUSER	|始于 Linux 2.6.23 完成于 Linux 3.8)|

>>>>>>> origin/master

##03 参考
- 容器运行时 	[opencontainers/runc](https://github.com/opencontainers/runc)
- 虚拟机运行时 [hyperhq/runv](https://github.com/hyperhq/runv)
- OCI测试 [huawei-openlab/oct](https://github.com/huawei-openlab/oct)

