# 第三十章 【IPFS 一问一答】解析 IPFS Multiformat 之 Multicodec

# 30 【IPFS 一问一答】解析 IPFS Multiformat 之 Multicodec

multicodec 是一种自描述的编解码，它包含一些自我描述的其他格式。多代码标识符是 varint。它其实是个 table，用 1 到 2 个字节定了数据内容的格式，比如用字母 z 表示 base58 btc 编码，0x50 表示 protobuf 等等。

上边提到的 table 在这里可以看到 https://github.com/multiformats/multicodec/blob/master/table.csv

multiodec 识别的一大块数据如下所示：

```go
< multicodec> <encoded-data > 
＃我们可以简写成：
< mc> <data >
```

另一个有用的场景是使用 multicodec 作为访问数据的密钥的一部分，例如：

```go
# suppose we have a value and a key to retrieve it
"<key>" -> <value>

# we can use multicodec with the key to know what codec the value is in
"<mc><key>" -> <value> 
```

值得注意的是，multiodec 与 multihash 和 multiaddr 配合使用效果非常好，我们可以使用多重代码为这些值添加前缀，以告诉它们是什么。