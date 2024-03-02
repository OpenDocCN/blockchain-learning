# 三、.1 教程 1: 启动本地节点，检查种子节点和加入的 ipfs 集群

启动本地节点，加入到 IPFS 网络中，命令非常简单，`ipfs daemon`，操作如下： 新开启一个终端窗口，输入`ipfs daemon`，得到：

```go
$ ipfs daemon
Initializing daemon...
go-ipfs version: 0.4.18-
Repo version: 7
System version: amd64/darwin
Golang version: go1.11.1
Swarm listening on /ip4/127.0.0.1/tcp/4001
Swarm listening on /ip4/192.168.0.121/tcp/4001
Swarm listening on /ip6/::1/tcp/4001
Swarm listening on /p2p-circuit
Swarm announcing /ip4/127.0.0.1/tcp/4001
Swarm announcing /ip4/192.168.0.121/tcp/4001
Swarm announcing /ip6/::1/tcp/4001
API server listening on /ip4/127.0.0.1/tcp/5001
Gateway (readonly) server listening on /ip4/127.0.0.1/tcp/8080
Daemon is ready 
```

执行命令后，最后出现`Daemon is ready`，就代表节点启动成功了，并已经作为节点加入了 IPFS 网络了。

检查本机的种子节点命令：`ipfs bootstrap` 检查本机加入的集群信息：`ipfs swarm peers`