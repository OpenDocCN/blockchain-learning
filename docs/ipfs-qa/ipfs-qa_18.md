# 第十八章 【IPFS 一问一答】IPFS 底层架构解析之身份层

# 18 IPFS 底层架构解析之身份层

每个 IPFS 节点都有自己的身份，身份用 NodeID 表示。节点在加入 IPFS 网络前，首先要生成自己的身份。通过 S/Kademlia 静态加密算法产生一个公钥，然后通过 hash 运算到的值就是 NodeID。用 C++语言描述 NodeID 的生成过程：

```go
//设定一个难度系数，即先导 0 的个数
difficulty =< integer parameter >
//初始化一个节点
n= Node{}
//循环运算，直到满足条件 hash（NodeID）运算后的值的先导 0 的个数≥设定的先导 0 个数
do{
//产生公私钥对
n.PubKey, n.PrivKey = PKI.genKeyPair() 
//对公钥进行 hash 运算，得到未验证的 NodeID
n.NodeId = hash(n.PubKey)
//判断 NodeID 的合法性，验证通过，循环结束
p = count_preceding_zero_bits(hash(n.NodeId)) }while(p<difficulty)
```

从上边的过程我们知道，节点会不断的产生公私钥对，直到公钥的 hash 运算值满足要求才会停止。而最后的公钥 hash 运算的值就是本机作为 ipfs 节点的 NodeID。

在终端运行`ipfs id`会显示本机的 ipfs 节点信息，如下：

```go
$ ipfs id
{
    "ID": "QmXdSpUBx9Ut6q8LF8Wyt1Wi2wxmob6qnnT11uV3SmvUP3",
    "PublicKey": "CAASpgIwggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEKAoIBAQCh7fHOV1X0LEsqxxlY+wRRuMGZ+E7sMAWXfMj8NLenv3KpIX8pHy0lk/H6VCCjKB+t4e2Rb+Px9Uwh2YjRlLaaMkBYN27COVBdtL9sH5ZW2BZ6x00Deg+gWoO0xs5rzadtk45vV44RzxYDuGCT0GW79WgTBUi0s1O00LE6p+wOrZdk6UGYjUEzL3nD6CRMPoQbVtV8WGBWFksoyM+bnlyyCcFhw0suN5Yf8OicwGIDznsjniSOrku9QpoFU1B96SKMmfqiXZC+KyzFfKp/U1lFmfp7wlYObUYpMkcWI0/bRzdursQGga7xXyKwQJ5ZGouSANt6cClTMnOUHGLCFzzNAgMBAAE=",
    "Addresses": null,
    "AgentVersion": "go-ipfs/0.4.18/",
    "ProtocolVersion": "ipfs/0.1.0"
} 
```

用户可以通过删除本机的 ipfs 节点信息文件，再初始化（命令是 ipfs init）来重新设定新的身份。但是这样做会损失掉之前身份的所有网络利益。

当 IPFS 网络中的节点首次进行连接时，彼此会互换自己的公钥，然后对对方的公钥进行 hash 运算，即 hash（other.PubKey），如果得到的值等于对方的 NodeID，那么说明对方是合法的 ipfs 节点，彼此互连，反之连接被终止。

这里说的 hash 函数得到的值是个自我描述的值，如`QmXdSpUBx9Ut6q8LF8Wyt1Wi2wxmob6qnnT11uV3SmvUP3`，这个值是经过 Base58 编码后的值。读者可能会注意到，所有的散列都是以“Qm”开头的。这是因为它实际上是一个 multihash，前两个字节用于指定哈希函数和哈希长度。前两个字节的十六进制是 1220，其中 12 表示这是 SHA256 哈希函数，20 代表哈希函数选择 32 字节长度计算。