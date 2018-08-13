---
title: consul负载均衡的使用
date: 2018-08-08 16:46:47
tags:
---
<h2>什么是命名</h2>

在一个集群中进程和服务可以相互发现，相互通信，并且知道彼此的基本信息。在大规模的服务集群中，机器的上线下线是比较正常的，名字服务提供一个可以集群管理机器、服务等信息的地方。如果你的系统是比较复杂的，需要较强的可扩展性时，服务被频繁替换时，为避免服务中断，名字服务是重要的。

<h2>什么是consul</h2>

  Consul 是一个支持多数据中心分布式高可用的服务发现和配置共享的服务软件,由 HashiCorp 公司用 Go 语言开发, 基于 Mozilla Public License 2.0 的协议进行开源， Consul 支持健康检查,并允许 HTTP 和 DNS 协议调用 API 存储键值对。一致性协议采用 Raft 算法,用来保证服务的高可用，使用 GOSSIP 协议管理成员和广播消息, 并且支持 ACL 访问控制
  
<h2>为什么是consul</h2>

在开源系统中可以做名字服务的项目又很多：Zookeeper，Doozer，Etcd，Consul等，各有优势和缺点
![](./consul.jpg)

<h2>consul的基础架构</h2>

server: 服务端, 保存配置信息, 高可用集群, 在局域网内与本地客户端通讯, 通过广域网与其他数据中心通讯. 每个数据中心的
client: 客户端, 无状态, 将 HTTP 和 DNS 接口请求转发给局域网内的服务端集群。
![](./info.png)

<h2>consul安装</h2>
	
    ```c++
    > wget https://releases.hashicorp.com/consul/1.1.0/consul_1.1.0_linux_amd64.zip
    > unzip consul_1.1.0_linux_amd64.zip
    解压后，可以看到只有一个consul的二进制文件，我们移动到 /usr/local/bin目录下：
    >cp consul /usr/local/bin/
    接着通过执行命令检查consul是否安装成功：
    >consul -h
    Usage: consul [--version] [--help] <command> [<args>]
    Available commands are:
        agent          Runs a Consul agent
        catalog        Interact with the catalog
        event          Fire a new event
        exec           Executes a command on Consul nodes
        force-leave    Forces a member of the cluster to enter the "left" state
        info           Provides debugging information for operators.
        join           Tell Consul agent to join cluster
        keygen         Generates a new encryption key
        keyring        Manages gossip layer encryption keys
        kv             Interact with the key-value store
        leave          Gracefully leaves the Consul cluster and shuts down
        lock           Execute a command holding a lock
        maint          Controls node or service maintenance mode
        members        Lists the members of a Consul cluster
        monitor        Stream logs from a Consul agent
        operator       Provides cluster-level tools for Consul operators
        reload         Triggers the agent to reload configuration files
        rtt            Estimates network round trip time between nodes
        snapshot       Saves, restores and inspects snapshots of Consul server state
        validate       Validate config files/directories
        version        Prints the Consul version
        watch          Watch for changes in Consul
    ```

<h2>使用示例</h2>

	```c++
    sudo consul agent -server -bootstrap-expect 1 -data-dir /data/consul/ -node=s1 -bind=127.0.0.1  -rejoin -config-dir=/etc/consul.d/ -client 0.0.0.0
    ```

参数说明：
* server ：定义agent运行在server模式，如果是client模式则不需要添加这个参数
* bootstrap-expect ：datacenter中期望提供的server节点数目，当该值提供的时候，consul一直等到达到指定sever数目的时候才会引导（启动）整个集群，为了测试演示，我们这里使用1
* data-dir：consul的工作目录，不需要提前创建
* node：节点在集群中的名称，在一个集群中必须是唯一的，默认是该节点的主机名
* bind：该地址用来在集群内部的通讯，集群内的所有节点到地址都必须是可达的，默认是0.0.0.0
* rejoin：使consul忽略先前的离开，在agent再次启动后仍旧尝试加入集群中。也就是说如果不加入这个参数，当前节点一旦退出，下次重启后是不会自动加入到集群中去的，除非是手动触发 consul join xxxx ，所以为了降低重启后对本身服务的影响，这里统一使用 -rejoin参数。
* config-dir：配置文件目录，里面文件统一规定是以.json结尾才会被自动加载并读取服务注册信息的，（p.s该目录要提前创建）
* client：consul服务侦听地址，处于client mode的Consul agent节点比较简单，无状态，仅仅负责将请求转发给Server agent节点
当服务启动后，后台输出类似以下日志信息：

	```c++
    BootstrapExpect is set to 1; this is the same as Bootstrap mode.
    bootstrap = true: do not enable unless necessary
    ==> Starting Consul agent...
    ==> Consul agent running!
               Version: 'v1.2.1'
               Node ID: '6892d3e1-83e1-d222-648a-2fdd7cb1be86'
             Node name: 's1'
            Datacenter: 'dc1' (Segment: '<all>')
                Server: true (Bootstrap: true)
           Client Addr: [0.0.0.0] (HTTP: 8500, HTTPS: -1, DNS: 8600)
          Cluster Addr: 127.0.0.1 (LAN: 8301, WAN: 8302)
               Encrypt: Gossip: false, TLS-Outgoing: false, TLS-Incoming: false

    ==> Log data will now stream in as it occurs:

        2018/08/09 14:59:09 [INFO] raft: Initial configuration (index=1): [{Suffrage:Voter ID:6892d3e1-83e1-d222-648a-2fdd7cb1be86 Address:127.0.0.1:8300}]
        2018/08/09 14:59:09 [INFO] serf: EventMemberJoin: s1.dc1 127.0.0.1
        2018/08/09 14:59:09 [INFO] serf: EventMemberJoin: s1 127.0.0.1
        2018/08/09 14:59:09 [INFO] agent: Started DNS server 0.0.0.0:8600 (udp)
        2018/08/09 14:59:09 [INFO] raft: Node at 127.0.0.1:8300 [Follower] entering Follower state (Leader: "")
        2018/08/09 14:59:09 [INFO] consul: Adding LAN server s1 (Addr: tcp/127.0.0.1:8300) (DC: dc1)
        2018/08/09 14:59:09 [INFO] consul: Handled member-join event for server "s1.dc1" in area "wan"
        2018/08/09 14:59:09 [WARN] agent/proxy: running as root, will not start managed proxies
        2018/08/09 14:59:09 [INFO] agent: Started DNS server 0.0.0.0:8600 (tcp)
        2018/08/09 14:59:09 [INFO] agent: Started HTTP server on [::]:8500 (tcp)
        2018/08/09 14:59:09 [INFO] agent: started state syncer
        2018/08/09 14:59:14 [WARN] raft: Heartbeat timeout from "" reached, starting election
        2018/08/09 14:59:14 [INFO] raft: Node at 127.0.0.1:8300 [Candidate] entering Candidate state in term 2
        2018/08/09 14:59:14 [INFO] raft: Election won. Tally: 1
        2018/08/09 14:59:14 [INFO] raft: Node at 127.0.0.1:8300 [Leader] entering Leader state
        2018/08/09 14:59:14 [INFO] consul: cluster leadership acquired
        2018/08/09 14:59:14 [INFO] consul: New leader elected: s1
        2018/08/09 14:59:14 [INFO] consul: member 's1' joined, marking health alive
        2018/08/09 14:59:14 [INFO] agent: Synced node info
    ==> Newer Consul version available: 1.2.2 (currently running: 1.2.1)
    ```
查看一下集群成员组成信息

	```c++
    consul members
    Node  Address         Status  Type    Build  Protocol  DC   Segment
    s1    127.0.0.1:8301  alive   server  1.2.1  2         dc1  <all>
    或使用api
    >curl http://localhost:8500/v1/status/peers
	["127.0.0.1:8300"]
    ```
    
命令输出信息中可以看到当前的集群中只有Node名称为s1这一个成员，它的状态是alive,代表是正常运行的。dc1是默认的datacenter名称；Type会列出成员的角色，当前是server；Address中 8301 这个端口是agent默认的LAN端口（WAN端口默认是8300）; Protocol 只是consul集群不同agent版本兼容性设计的，不同版本的agent, protocol的值可能不一样，这个我们先不管。

我们可以使用consul api查看当前的leader：

	```c++
    >curl http://localhost:8500/v1/status/leader
	"127.0.0.1:8300
    ```

更详细的节点运行信息可以使用命令 consul info 查阅

在多台机器上类似于上面配置好之后，调用如下命令加入集群：

	```c++
    consul join <ip>
    或者在启动时加入-join=<ip> 参数
    离开集群：
    consul leave
    ```
    
没有-server参数时，会默认以client的角色运行
	```c++
	consul agent  -data-dir /data/consul/ -node=c1 -bind=127.0.0.1  -rejoin -config-dir=/etc/consul.d/ -client 0.0.0.0
    ```
<h2>服务发现示例</h2>

<h4>往consul注册一个测试服务</h4>

建立Consul Cluster目的是实现服务的注册和发现。Consul支持两种服务注册的方式：
* 通过Consul的服务注册HTTP API，由服务自身在启动后调用API注册自己
* 通过在配置文件中定义服务的方式进行注册
Consul文档中建议使用后面一种方式来做服务配置和服务注册，而且配置很简单，只需往之前启动命令中的 --config-dir 指定的目录下新建json格式的注册文件即可，举个例子：

	```c++
    >touch /etc/consul.d/web.json
    以标准json格式填写如下：
    {
      "service": {
        "name": "web",
        "tags": ["master"],
        "address": "127.0.0.1",
        "port": 10000,
        "checks": [  
          {
            "http": "http://localhost:9999/health",
            "interval": "10s"
          }
        ]
      }
    }
    ```
这个配置就是我们在s3节点上配置了一个web服务做，这个服务中我们定义了服务的name、address、port等，还包含一个checks配置用于健康检查，上面我们的示例会每隔10秒进行一次 http://localhost:9999/health 请求作为healthcheck。checks字段并不是必须的

服务定义文件后，我们需要对两者的consul agent进行重启，才能起到服务更新的效果。

为了完整查看集群中所提供的服务，可以通过特定的API：
	
    ```c++
    curl http://localhost:8500/v1/catalog/service/web\?pretty  // web是服务名字
    ```
    
我们再以brpc/example/echo_c+/echo_server为例子：
	```c++
   	> cd ~/brpc/example/echo_c+/
   	> ./echo_server &
   	> cd /etc/consul.d
   	> vim echo_server.json
        {
            "service": {
                "name" : "echo_server",
                "tags" : ["test"],
                "address": "127.0.0.1",
                "port": 8000
            }
        }
    > sudo consul agent -server -bootstrap-expect 1 -data-dir /data/consul/ -node=s1 -bind=127.0.0.1  -rejoin -config-dir=/etc/consul.d/ -client 0.0.0.0 &
    > curl http://localhost:8500/v1/catalog/service/echo_server?pretty
        [
            {
                "ID": "6892d3e1-83e1-d222-648a-2fdd7cb1be86",
                "Node": "s1",
                "Address": "127.0.0.1",
                "Datacenter": "dc1",
                "TaggedAddresses": {
                    "lan": "127.0.0.1",
                    "wan": "127.0.0.1"
                },
                "NodeMeta": {
                    "consul-network-segment": ""
                },
                "ServiceKind": "",
                "ServiceID": "echo_server",
                "ServiceName": "echo_server",
                "ServiceTags": [
                    "test"
                ],
                "ServiceAddress": "127.0.0.1",
                "ServiceMeta": {},
                "ServicePort": 8000,
                "ServiceEnableTagOverride": false,
                "ServiceProxyDestination": "",
                "ServiceConnect": {
                    "Native": false,
                    "Proxy": null
                },
                "CreateIndex": 459,
                "ModifyIndex": 459
            }
        ]

    // 测试
    >./echo_client --server=consul://echo_server --load_balancer=random
    ```

<h4>DNS方式的服务发现</h4>

使用dig命令:

	```c++
    dig @127.0.0.1 -p 8600 web.service.consul SRV
    ```
简答介绍一下dig命令，dig命令是常用的域名查询工具，可以用来测试域名系统工作是否正常。
语法：dig(选项)(参数)
选项：
@<服务器地址>：指定进行域名解析的域名服务器；
-b<ip地址>：当主机具有多个IP地址，指定使用本机的哪个IP地址向域名服务器发送域名查询请求；
-f<文件名称>：指定dig以批处理的方式运行，指定的文件中保存着需要批处理查询的DNS任务信息；
-P：指定域名服务器所使用端口号；
-t<类型>：指定要查询的DNS数据类型；
-x<IP地址>：执行逆向域名查询；
-4：使用IPv4；
-6：使用IPv6；
-h：显示指令帮助信息。
SRV标志，那是因为我们需要的服务信息不仅有ip地址，还需要有端口号