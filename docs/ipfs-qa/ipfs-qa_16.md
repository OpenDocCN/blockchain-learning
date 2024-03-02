# 第十六章 【IPFS 一问一答】IPFS 的数据存储是去重的吗？

# 16 IPFS 的数据存储是去重的吗？

在 IPFS 网络中，数据的存储可能是有重复的。这多少与大家认识的 IPFS 去重存储有些相悖。

之前提到过数据在 IPFS 存储是以块的形式存储的。在 ipfs 提供的数据分割方式有很多种。在 ipfs 源码种 core/commands/add.go 代码中描述了切割的方法：

```go
The chunker option, '-s', specifies the chunking strategy that dictates
how to break files into blocks. Blocks with same content can
be deduplicated. The default is a fixed block size of
256 * 1024 bytes, 'size-262144'. Alternatively, you can use the
rabin chunker for content defined chunking by specifying
rabin-[min]-[avg]-[max] (where min/avg/max refer to the resulting
chunk sizes). Using other chunking strategies will produce
different hashes for the same file.

  > ipfs add --chunker=size-2048 ipfs-logo.svg
  added QmafrLBfzRLV4XSH1XcaMMeaXEUhDJjmtDfsYU95TrWG87 ipfs-logo.svg
  > ipfs add --chunker=rabin-512-1024-2048 ipfs-logo.svg
  added Qmf1hDN65tR55Ubh2RN1FPxr69xq3giVBz1KApsresY8Gn ipfs-logo.svg
```

上边的内容描述的是对同一个文件，可以用 3 种方式进行切割。假如一个文件是 chaindesk。

*   1.默认模式，块的大小是 256kb，也就是 256 * 1024 bytes，对应的 size=262144。命令不需要加参数，即 ipfs add chaindesk。
*   2.指定块大小模式。命令是 ipfs add --chunker=size-1000。其中后边的 1000 可以是任意小于 262144 的数。
*   3.rabin 可变块大小切割模式。命令是 ipfs add --chunker=rabin-[min]-[avg]-[max] chaindesk。其中 min，avg，max 的值分别值最小块大小，平均块大小，最大块大小的意思，值在小于 262144 自行设定。如代码中给出的例子 ipfs add --chunker=rabin-512-1024-2048 ipfs-logo.svg

注：大家想详细了解文件切割细节，推荐一个网址 https://baijiahao.baidu.com/s?id=1610602352467634956&wfr=spider&for=pc

通过上边 3 种方式可扩展出很多很多块的大小，所以同一个文件存储在 ipfs 中，因为存储是选用的文件切割方法不同，返回的 hash 值却不一样。

所以说 IPFS 的块存储没有重复的，而 IPFS 块文件拼凑的数据可能有重复的。也就是说同一个文件可以根据不同的文件切割方法在 IPFS 网络中重复的存储多次。

IPFS 官方说的 ipfs 数据存储是去重的，指的是系统中块数据没有重复的，而由块文件拼凑出来的数据是可以重复的。

但不管怎样，大家使用 ipfs 都是采用默认的分割模式，即 blocksize=256kb。所以对应的数据文件在网络中的存储是唯一的去重的。所以当你往 IPFS 中存储网络中已存在的内容的时候，IPFS 会很快告诉你，该数据在网络中之前已经被存好了，您不需要再去存储了。

你想想同样的数据在网络中只被存储一份，全世界将节省多少硬盘存储空间啊。假如一部非常火的电影，大家都习惯性的将该电影存储到自己的电脑 E 盘或其它硬盘存储中，全世界如果有 1 亿的人存储了这个电影，这不是对存储的极大浪费吗？在 ipfs 网络中，该电影只被存储在一个节点中，当有用户需要读取的时候，会产生新的备份。就是谁使用数据，这个数据就会复制到谁那里。这将节省了多少硬盘存储空间啊。

当一个节点加入 IPFS 网络时，这个节点会提供一部分硬盘空间（缺省为 10G，可以配置）给整个网络使用。那么通常情况下，当您在存储文件的时候，您自己提供的这部分硬盘空间总是最快的，因为不需要跨网。当存储完毕后，网络上任意节点都可以访问这个文件。当另一个节点访问的时候，那个节点往往会复制一份您的数据到他的缓存空间。这样整个网络中就有两份拷贝了。试想，当有很多人对这个文件感兴趣，那么网络中的拷贝数会越来越多。

需要提出的是：拷贝一般都是缓存，也就是说是临时存储的。时间一长就被自动删除掉了。这种临时缓存非常好地解决了分布式数据分发的问题，比如说一个社会热点往往呈现出预热期、火热期和退潮期等阶段，利用 IPFS，数据的分布和拷贝数与这些时期是完全匹配的。访问的人越多，拷贝数就越多，但热度下来了，拷贝数就会降下来，从而自然地实现空间利用率和存取效率的平衡。