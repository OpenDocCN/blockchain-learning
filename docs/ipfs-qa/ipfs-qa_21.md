# 第二十一章 【IPFS 一问一答】IPFS 底层架构解析之交换层

# 21 IPFS 底层架构解析之交换层

IPFS 的交换层即通过 BitSwap 协议使对等节点之间进行数据交换，类似 BitTorrent 技术，每个对等节点在下载的同时不断向其他对等节点上传已下载的数据。和 BitTorrent 协议不同的是，BitSwap 不局限于一个种子文件中的数据块。

在整个交换层中，就像存在一个市场，IPFS 的所有节点是市场中的角色，他们之间进行数据交换，Filecoin 就是在这一层加了一层激励层。在这个市场中，Bitswap 起到两个主要工作：

*   试图从网络中获取客户端请求的块。
*   将其拥有的 block 发送给其他节点。

## 21.1 BitSwap 协议

  BitSwap 记录了 ipfs 节点的块交换的信息，源码结构如下：      ```go type BitSwap struct { ledgers map[NodeId]Ledger //节点账单 active map[NodeId]Peer //当前已经连接的对等点 need_list []Multihash //此节点需要的块数据校验列表 have_list []Multihash //此节点已收到块数据校验列表 }

```
当节点需要向其他节点请求数据块或者为其他节点提供数据块时，都会发送 BitSwap message 消息，其中主要包含了两部分内容：想要的数据块列表（want_list）以及对应数据块，整个消息都会使用 Protobuf 进行编码： 
```go

message Message { message Wantlist { message Entry { optional string block = 1; optional int32 priority = 2; //设置优先级，默认为 1 optional bool cancel = 3; //是否会撤销条目 } repeated Entry entries = 1; optional bool full = 2; } optional Wantlist wantlist = 1; repeated bytes blocks = 2; }

```
在 BitSwap 系统中，有两个非常重要的模块需求管理器（Want-Manager）和决定引擎（Decision-Engine）：前者会在节点请求数据时在本地返回相应的结果或发出合适的请求，而后者决定如何为其他节点分配资源，当节点接收到包含 want_list 的消息时，消息会被转发到决定引擎，引擎会根据该节点的 BitSwap 账单决定如何处理请求，整个处理流程如下图：
![](http://image.chaindesk.cn/ipfs8%E5%B1%82%E4%B9%8B%E5%9D%97%E4%BA%A4%E6%8D%A2.png/mark)

通过上面的协议流程图，可以看到一次 BitSwap 数据交换的全过程以及对等连接这块的生命周期，在这个生命周期中，对等节点一般要经历四个状态：

- 状态开放（Open）：对等节点间开放待发送 BitSwap 账单状态，直到建立连接。
- 数据发送（Sending）：对等节点间发送 want_list 和数据块。
- 连接关闭（Close）：对等节点发送完数据后断开连接。
- 节点忽略（Ignored）：对等节点因为超时、自动以、信用分过低等因素被忽略。

结合对等节点的源码结构，分析一下 IPFS 节点是如何找到彼此的： 
```go

type Peer struct { nodeid NodeIs ledger Ledger //节点和此对等节点之间的分类账单 last_seen Timestamp //最后收到消息的时间戳 want_list []Multihash //需要的所有块校验 } // 协议接口 interface Peer { open (nodeid:NodeId,ledger:Ledger); send_want_list(want_list:WantList); send_block(block:Block) -> (complete:Bool); close(final:Bool); }

```
 ### 21.1.1 Peer.open(NodeID,Ledger)

  当节点建立连接时，发送方节点会初始化 BitSwap 账单，可能保存一份对等方的账单，也可能将创建一个新的被清零账单，这取决于节点账单一致性问题。之后，发送方节点将发送一个携带账单的 open 信息通知接收方节点，接收方节点收到一个 open 信息后，可以选择是否接受此连接请求。
  如接收方根据本地的账单数据，发现发送方是一个不可信的节点，即传输超时，很低信用分，很大的债务率，接收方可能会通过 ignore_cooldown 忽略这个请求，断开连接，目的是为了防范作弊行为。
  如果连接成功，接收方将利用本地账单来初始化一个 Peer 对象并更新 last_seen 时间戳，然后，它会将接收到的账单与自己的账单进行比较。如果两个账单完全一样，那么这个连接就被 Open，如果账单不完全一致，那么此节点会创建一个新的被清零的账单并发送同步此账单，以此保证之前提到的发送方节点和接收方节点的账单一致性问题。

### 21.1.2 Peer.send_want_list(WantList)

  当连接已经处于 open 状态，发送方节点将会把 want_list 广播给所有连接的接收方节点。与此同时，接收方节点在收到一个 want_list 后，会检查自身是否有接收方想要的数据块，如果有，会使用 BitSwap 策略来发送传输这些数据块。

### 21.1.3 Peer.send_block(Block)

  发送块的方法逻辑很简单，默认发送方节点只传输数据块，接收到所有数据后，接收方节点计算 Multihash 以校验它是否与预期的匹配，然后返回确认。在完成块的传输后，接收方节点将数据块信息从 need_list 移到 have_list，并且接收方和发送方都同步更新他们的账单列表。如果传输验证失败，则发送方可能发生故障或者故意攻击接收方的行为，接收方可以拒绝进一步的交易。

### 21.1.4 Peer.close(Bool)

  对等连接应该在两种情况下关闭:

- silent_want 超时已过，但未收到来自对方的任何消息，节点发出 Peer.close(false)。
- 节点正在退出，BitSwap 正在关闭，在这种情况下，节点发出 Peer.close(true)。

对于 P2P 网络，有一个很重要的问题：如何激励大家分享自己的数据，每一个 P2P 软件都有自己专属的数据分享策略，IPFS 也是如此，其中 BitSwap 的策略体系由信用、策略、账单三部分组成。

## 21.2 BitSwap 信用体系

  BitSwap 协议必须能激励节点去分享数据，IPFS 根据节点之间的数据收发建立了一个信用体系：有借有还，再借不难。

- 发送给其他节点数据可以增加信用值。
- 从其他节点接收数据将降低信用值。

如果一个节点只接收数据不上传数据，信用值会降低而被其他节点忽略掉。这能有效防范一些类似女巫攻击，洪泛攻击的网络攻击。

## 21.3 BitSwap 策略

  有了 BitSwap 信用体系，还可以采取不同的策略来实现，每一种策略都会对系统的整体性能产生不同的影响，策略的目标是：

- 节点数据交换的整体性能和效率最高。
- 阻止空载节点“吃白食”现象，即不能够只下载不上传数据。
- 有效防止一些攻击行为。
- 对信任节点建立宽松机制。

IPFS 在白皮书里提供了一些参考的策略机制，每个节点根据和其他节点的收发数据，计算信用分和负债率（debt ratio,r）: `r = bytes_sent / bytes_recv +1`

数据发送率（P）:
` P(send|r) = 1 - (1/(1+exp(6-3r)))`

根据上面共识，如果 r 大于 2 时，发送率 P(send|r)将变得很小，从而其他节点将不会继续发送数据给自己。

## 21.4 BitSwap 账单

  BitSwap 节点记录下来和其他节点通信的账单（数据收发记录），账单数据结构如下：

  ```go
type Ledger struct {
    Owner NodeId
    partner NodeId
    bytes_sent int
    bytes_recv int
    timestamp Timestamp
}
```

这可以让节点追踪历史记录以避免被篡改。当两个节点之间建立连接的时候，BitSwap 会相互交换账单信息，如果账单不匹配，则直接清楚并重新记账，恶意节点会有意失去这些账单，从而期望清除自己的债务。其他节点会把这些都记录下来，如果总是发生，伙伴节点可以自由的将其视为不当行为，拒绝交易。