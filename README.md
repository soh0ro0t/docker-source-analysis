#《docker 源码解析》
-----
##00 <i class="icon-chevron-sign-left"></i> 序
-----
docker 是Linux OS虚拟化容器，原生环境是Linux 64位。具有快速开发、运维，以及部署的特点，它使SaaS层服务对系统和中间层的依赖性变弱，用户能够基于docker官方的基础镜像私人定制Dockerfile文件，搭建和配置特定服务，创建过程全都自动化实现。

docker 基于c/s模式，首先启动dockerd的服务端，并监听特定的unix socket或tcp socket，远程客户端发送请求到服务端的网络接口，然后服务端通过router实例对请求事件进行匹配和分发，再进入相应的hander处理流程，最后将执行结果回传给客户端。

##01 <i class="icon-list"></i> 目录
|章节|标题|
|:-:|:-:|
|   第一章  | docker daemon启动过程|
|   第二章  | router mapping 集合|
|   第四章  | |
|   第五章  | |
|   第六章  | |

##02 <i class="icon-desktop"></i> 参考

《第一本Docker书》（James Turnbull 著）
《Docker技术入门与实践》（杨保华 著）
《Docker进阶与实战》（华为Docker实践小组 著）
《Docker源码分析》（孙宏亮 著）

