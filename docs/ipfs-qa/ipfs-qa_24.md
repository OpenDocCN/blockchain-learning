# 第二十四章 【IPFS 一问一答】IPFS 底层架构解析之命名层

# 24 IPFS 底层架构解析之命名层

## 24.1 IPNS：命名以及易变状态

IPFS 对文件或目录文件可以产生一个可寻址的 hash 值，通过该 hash 值可以检索到相应的文件内容。但文件的内容一旦改变，之前的 hash 值将不能检索到最新的内容，而只能检索到历史的内容。这样，对于我们发布可变的内容时，被检索将极不方便。

因此，为了解决这个问题，出现了 IPNS，将某个值锁定到可变内容的最新状态。即，当你重新编辑文件内容后，通过锁定的 IPNS 值，可以访问到最新的内容。

## 24.2 自我认证命名

SFS 自我认证的命名系统给 IPFS 提供了一种在加密分配的全局命名空间中构建可变的自我认证命名的方法。IPFS 的命名方案如下：

*   通过 NodeId = hash(node.PubKey)，生成 IPFS 节点信息。
*   给每个用户分配一个可变的命名空间，由之前生成的节点 ID 信息作为地址名称，在此路径下： /ipns/ 。
*   一个用户可以在此路径下发布一个用自己私钥签名的对象，比如说： /ipns/XLF2ip4ii9x0wejs23HD2swlddVmas8kd0Ax/ 。
*   当其他用户获取对象时，他们可以检测签名是否与公钥和节点信息匹配，从而验证用户发布对象的真实性，达到可变状态的获取。

比如一个 IPNS 的链接：[`ipfs.io/ipns/QmdKXkeEWcuRw9oqBwopKUa8CgK1iBktPGYaMoJ4UNt1MP`](https://ipfs.io/ipns/QmdKXkeEWcuRw9oqBwopKUa8CgK1iBktPGYaMoJ4UNt1MP)

前面部分 https://ipfs.io/ipns/ 是命名空间，后边是节点的身份 NodeID。所有的节点都是在同一个命名空间下进行 IPNS 命名的。

需要注意的是，这块的动态可变内容是通过设置路由函数来控制的，通过这段源码我们也能了解到为什么命名空间是以绑定 NodeId 的形式来挂载的了：`routing.setValue(NodeId,<ns-object-hash>)`

在命名空间中，所发布的数据对象路径名称可以被当做子名称：

```go
/ipns/XLF2ipQ4jD3UdeX5xp1KBgeHRhemUtaA8Vm/
/ipns/XLF2ipQ4jD3UdeX5xp1KBgeHRhemUtaA8Vm/docs
/ipns/XLF2ipQ4jD3UdeX5xp1KBgeHRhemUtaA8Vm/docs/ipfs
```

## 24.3 人类友好命名

IPNS 是以节点的 NodeID 作为锁定 hash 值，对于用户来说，这么一长串字符串，很难记住且看上去很不友好。所以需要通过策略设置域名解析，如域名 chaindesk.com。我们可以通过 ipfs.io/ipns/chaindesk.com 来对数据内容进行访问。

1.对等节点链接

  遵循自验证文件系统（SFS）的设计理念，用户可以将其他用户节点的对象直接链接到自己的命名空间下，这有利于创建一个更信任的网络：      ```go   # Alice links 到 Bob 上 ipfs link /<alice-pk-hash>/friends/bob /<bob-pk-hash></bob-pk-hash></alice-pk-hash>

# Eve links 到 Alice 上

ipfs link /<eve-pk-hash>/friends/alice /</eve-pk-hash>

# Eve 也可以访问 Bob

/<eve-pk-hash>/friends/alice/friends/bob</eve-pk-hash>

# 访问 Verisign 认证域

/<verisign-pk-hash>/foo.com   ``` 2.DNS TXT IPNS 记录</verisign-pk-hash>

  在现有的 DNS 系统中添加 TXT 记录，这样能够通过域名访问 IPFS 网络中的文件对象：

```go
# DNS TXT 记录
ipfs.benet.ai. TXT "ipfs=XLF2ip4jD3U..."
# 表现为符号链接
ln -s /ipns/XLF2ip4jD3U... /ipns/fs.benet.ai

# IPFS 也支持可读标识符 Proquint,可以将二进制编码翻译成可读文件的方法，如下：
# proquint 语句
/ipns/dahih-dolij-sozuk-vosah-luvar-fuluh
# 分解为相应的下面形式
/ipns/KhWwNprxYVxKqpDZ
```

除此之外，IPFS 还提供短地址的命名服务，类似我们现在看到的 DNS 和 WebURL 链接：

```go
# 用户可以从下面获取一个 link
/ipns/shorten.er/foobar
# 然后放到自己的命名空间
/ipns/XLF2ipQ4JD3Udex6xKbgeHrhemUtaA9Vm
```