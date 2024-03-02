# 第十九章 【IPFS 一问一答】IPFS 底层架构解析之网络层

# 19 IPFS 底层架构解析之网络层

网路层也就是 IPFS 的 libp2p 模块。该模块下包含：

*   Transports：传输层
*   Discovery：网络发现层
*   Peer Routing: 节点路由
*   NAT Traversal: NAT 穿透层
*   Content Routing: 内容寻址

![](img/a3c30d565e0cf7ca7fecfe2a9adbbaa3.jpg)

网路层的作用是 IPFS 节点使用各种底层网络协议（可配置的）管理与其他节点的连接，以及保证它们之间正常地有规律地通信。该层不光为整个 ipfs 提供了基础的网络设备或者网络能力，还增加了加密传输，网络穿透，多链接混合等等技术。

IPFS 节点与其他节点连接通信的时候，会跨越广域网，IPFS 网络堆栈的特性如下：

1.传输：IPFS 兼容现有的主流传输协议，其中最适合浏览器端使用的 WebRTC Data Channels，低延时 uTP（LEDBAT）传输协议等。 2.可靠性：使用 uTP 和 sctp 来保障，这两种协议可以动态调整网络状态。 3.可连接性：使用 ICE 等 NAT 穿越技术来实现广域网的可连接性。 4.完整性：使用哈希校验检查数据完整性，IPFS 中所有数据块都有唯一的 Hash。 5.可验证性：使用数据发送者的公钥以及 HMAC 消息认证码来检查消息的真实性。

IPFS 可以使用任何网络，它并不依赖于 IP。这就允许 IPFS 可用来覆盖全网络。IPFS 是通过 multiaddr 的格式来表示目标地址和使用的协议，以此来兼容和扩展未来可能出现的其他网络协议：

```go
# an SCTP/IPv4 connection
/ip4/10.20.30.40/sctp/1234/
# an SCTP/IPv4 connection proxied over TCP/IPv4/ip4/5.6.7.8/tcp/5678/ip4/1.2 .3.4/sctp/1234/
```