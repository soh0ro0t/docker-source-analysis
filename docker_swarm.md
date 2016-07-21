
docker swarm环境搭建
------
##00 序
**docker swarm** 是docker生态链中重要一环，它用于搭建基于docker的主机集群，实现远程管理多个节点的基础服务，包括manager、backend、node三个组件构成。

- **1 node是基础节点，运行服务**
- **2 backend是认证端，连接manager和node，是中间层**
- **3 manager是管理端，用于远程使用CLI管理各个node的服务**

##01 手动搭建
**docker swarm** 搭建之前首先需要下载swarm和consul镜像，它们是manager和backend运行的实体对象。

	docker pull index.tenxcloud.com/docker_library/swarm
	docker pull index.tenxcloud.com/sdvdxl/consul
###1.1 node
	1. 运行docker daemon，开放2375端口
   	   sudo docker daemon -H tcp://0.0.0.0:2375 -H unix:///var/run/docker.sock
 	2. 连接节点和认证端
       sudo docker run -d --name=backend000-node000 zkx/swarm join --advertise=192.168.79.184:2375 consul://192.168.79.185:8500     
###1.2 backend
	1. 运行docker daemon
	2. 启动consul容器，开放8500端口
	   sudo docker run -d -p 8500:8500 --name=swarm-backend consul -server -bootstrap	

###1.3 manager
	1. 运行docker daemon
	2. 启动swarm容器，连接manager和认证端
	   sudo docker run -d --name=manager000-backend000 -p 4000:4000 zkx/swarm manage -H :4000 --replication --advertise 192.168.79.181:4000 consul://192.168.79.185:8500

###1.4 错误类型
###1.4.1 提示"Error response from daemon: No elected primary cluster manager"
	这种情况可能是由于backend端在正常连接中突然中断，然后再次启动就会提示上述错误，解决方法就是backend端重新创建和启动基于consul镜像的容器，然后manager端再次restart即可。
###1.4.2 只能创建单节点，无法加入其他节点
	经测试，manager和node之间的docker version必须相同，或者manager的docker version更低，这样才能正常通信，manager与node之间的关系好比是客户端与服务器，而backend的docker version则不影响。

##02 工具搭建
**docker machine** 创建和管理集群，首先要需要开启vm的vt选项，折腾了一下，找到了；然后是运行docker-machine，提示没有安装virtualbox，于是安装这个虚拟机，在虚拟机中安装虚拟机也是醉了。

还有，再次运行docker-machine，提示Boot2Docker ISO未安装，于是下载这个镜像。github上的release版本文件存放在s3 aws的服务器上，国内访问太慢，忍忍也能下，但是花费时间太长，目前没想到办法。

其次，下载完成后，不知道怎么处理这个ISO文件，这要命了，折腾了好久，只看见github boot2docker说是运行“boot2docker init”就创建虚拟机，可是boot2docker不是ISO吗，瞬间懵逼了，后来想明白了是通过boot2docker CLI利用boot2docker ISO创建虚拟机，那么我们刚刚下载的docker-machine呢，难道它们俩都是创建和管理虚拟机的？

于是，再次搜索“boot2docker 与 docker-machine”区别，看到第一条就是docker-machine取代boot2docker的内容，说明确实跟我猜想一致。好，既然知道了两者一样，那就使用docker-machine吧，因为它是用于管理集群。

最后，运行docker-machine -d virtualbox machine000，提示"(machine000) Downloading /root/.docker/machine/cache/boot2docker.iso from https://github.com/boot2docker/boot2docker/releases/download/v1.12.0-rc4/boot2docker.iso",说明只需将下载的文件拷贝到"~/.docker/machine/cache/"目录下即可，拷贝再次运行正确。

###2.1 搭建本地集群
首先简述本地集群搭建过程，这种方式下的machine均是主机上的virtualbox虚拟机，它们与主机组成本地的docker集群。

	docker-machine create -d virtualbox hub-token; eval "$(docker-machine env hub-token)"; docker run zkx/swarm create;(获取token的值)
	docker-machine create -d virtualbox --swarm --swarm-master --swarm-discovery token://SWARM_CLUSTER_TOKEN swarm-manager
	docker-machine create -d virtualbox --swarm --swarm-discovery token://SWARM_CLUSTER_TOKEN node000
	docker-machine create -d virtualbox --swarm --swarm-discovery token://SWARM_CLUSTER_TOKEN node001
登录 swarm-manager，能够查看node信息：

	eval "$(docker-machine env --swarm swarm-manager)"
	docker info

注意，目前docker-machine version 0.8.0-rc2，已经默认支持HTTPS协议，swarm cluster中的node和swarm master通过https协议互相访问对方。docker-machine 在第一次运行时在“/root/.docker/”目录下创建cache和certs目录，其中包括自签名的CA证书以及该machine的证书，然后每次创建新的machine时，docker-machine会将certs目录下的“cert.pem”,"key.pem","ca.pem"等公用的公私钥复制到新的machine目录下面，最后还创建新的server.pem和server-key.pem的密钥对。

###2.2 搭建基于Tls协议的C-S

然后搭建基于TLS协议的docker服务，先在本地尝试。目的是使docker daemon和Cli之间使用https协议进行通信，变得更安全可靠。主要分为创建本地CA、服务器证书、客户端证书等步骤，值得注意的是 openssl.cnf文件内容需要添加如下内容：
	
	subjectAltName = @alt_names

	[alt_names]
	DNS.1 = docker.local
	IP.1 = 192.168.99.100
	IP.2 = 127.0.0.1
注：subjectAltName 在此处用于绑定使用Tls协议的 IP地址或者域名。所以，应该加入docker daemon的IP地址。否则，会提示如下错误：

	certificate is valid for xxxx,not for yyyy

####2.2.1  创建 CA
	openssl genrsa -out ca-key.pem 2048
	openssl req -x509 -new -nodes -key /etc/docker/ssl/CA/ca-key.pem -days 10000 -out /etc/docker/ssl/CA/ca.pem -subj '/CN=docker-CA'

####2.2.2  创建服务器证书，请求CA为其签名
	openssl genrsa -out server-key.pem 2048
	openssl req -new  -key /etc/docker/ssl/server/server-key.pem -days 10000 -out /etc/docker/ssl/server/server.csr -subj '/CN=docker-server' -config /usr/lib/ssl/openssl.cnf
	openssl x509 -req -in /etc/docker/ssl/server/server.csr -CA /etc/docker/ssl/CA/ca.pem  -CAkey /etc/docker/ssl/CA/ca-key.pem -CAcreateserial   -out /etc/docker/ssl/server/server.pem -days 365 -extensions v3_req -extfile /usr/lib/ssl/openssl.cnf

####2.2.3  创建客户端证书，请求CA为其签名
	openssl genrsa -out client-key.pem 2048
	openssl req -new  -key /root/.docker/client-key.pem -days 10000 -out /root/.docker/client.csr -subj '/CN=docker-client' -config /usr/lib/ssl/openssl.cnf
	openssl x509 -req -in /root/.docker/client.csr -CA /etc/docker/ssl/CA/ca.pem  -CAkey /etc/docker/ssl/CA/ca-key.pem -CAcreateserial   -out //root/.docker/client.pem -days 365 -extensions v3_req -extfile /usr/lib/ssl/openssl.cnf

####2.2.4  客户端验证
	docker -H  192.168.99.100:2376 --tlsverify --tlscacert=/etc/docker/ssl/CA/ca.pem  --tlskey=/root/.docker/client-key.pem  --tlscert=/root/.docker/client.pem info

###2.1 搭建公网集群
在实际的生产环境中，本地集群的使用非常有限，更多还是由多个云主机组成的集群架构。包括swam-master、node共同组成，swarm-master开放某端口给指定的主机和用户访问(默认是2376)，这种添加远程主机到集群中需要generic驱动，而不是virtualbox驱动。

##03 通信协议
