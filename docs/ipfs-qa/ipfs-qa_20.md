# 第二十章 【IPFS 一问一答】IPFS 底层架构解析之路由层

# 20 IPFS 底层架构解析之路由层

之间讲过分布式哈希表 DHTs，主要运用在这一层。每个节点都会有自己的身份，即 NodeID。根据网络距离和延迟情况生成一个 Routing Table（路由表），根据对等节点拥有的数据生成 DHTs 分布式哈希表，该表的 key 是文件内容的 hash 值，value 是对应存储该数据的节点 NodeID。

节点在加入 ipfs 网络时，会通过种子节点接收到很多其它对等节点信息，本机会连接这些节点，根据 ping 的时间和节点距离形成路由表。路由表有多个 k 桶组成，k 桶是按本机节点的 NodeID 位数分布的，比如： 我的节点是 01011，k 桶的值是 5（k=5），那么 k 桶由近及远的分布是 k1:01010；k2:01000、01001；k3:01110、01111、01100、01101；k4:00 开头的节点。

然后 k 桶中的节点再通过 ping 的延迟时间分为 3 层，三层分别是 20 毫秒、60 毫秒和高延迟。

节点存数据时，直接将数据存在本地节点 ipfs 存储库中。设定数据的 hash 值是 key，同时广播给其它离 key 相近的节点，告诉他们，我保存了 key 的数据。其它节点就会保存这条信息，形成 DHT 表，内容是 key/value 存储，key 是数据 hash 值，value 是存储该信息的节点。

所以当某个节点检索 key 的数据时，首先会在自己的 blockstore 中查找，如果没有将会在离 key 最近的 k 桶中查询，里面的节点会在自己的 DHT 表中查询，如果存在 key 的 value，则反馈回去，如果没有，会给出离 key 最近的节点。

IPFS 的 DSHT 结构会根据所存数据的大小值进行区分：小的值（等于或小于 1KB）直接存储在 DHT 上，更大的值，DHT 只存储值索引，这个索引就是一个对等节点的 NodeId，该对等节点可以提供对该类型值的具体服务，DSHT 的接口位于 libP2P 模块中，如下：

```go
type IPFSRouting interface{
FindPeer (node NodeId)
//gets a particular peer’s network address 
//获取特定的节点网络地址
SetValue (key[]bytes, value []bytes)
 //stores a small meta data value in DHT 
 //通过 DHT 表存储较小的元文件
GetValue (key[]bytes)
//retrieves small meta data value from DHT 
//从 DHT 表中检索元文件
ProvideValue (key Multihash)
//announces this node can serve a large value
//广播该节点可以提供􏹅􏴎􏱖􏰊􏲃􏰋􏱑􏰉􏰊􏰾􏰿􏰉􏰊􏲡􏰐􏱅􏱆􏰣元文件对应的数据
FindValuePeers (key Multihash, min int)
//gets a number of peers serving a large value 
//获取提供数据的节点信息
}
```