# 第十二章 ChainIndexer 索引流程及实现

# ChainIndexer

本章节将主要介绍在以太坊系统中索引的整个创建流程、关于索引的三种索引器(Bloom、BloomTrie、Cht)的方法实现以及所适用的场景。

> 为什么会存在三种索引器？它们都在什么地方使用呢？
> 整个索引的创建过程是怎么样的？
> 什么时候会触发索引器？以怎样的形式触发呢？

* * *

在`以太坊启动流程`章节中，曾介绍过 Ethereum 以及 LightEthereum 的启动流程，而它们在初始过程中有一个步骤便是初始化索引器。

只不过在`Ethereum`客户端实例化过程，只用到一个索引器`BloomIndexer`, 通过`NewBloomIndexer(chainDb, params.BloomBitsBlocks, params.BloomConfirms)`方法对成员变量 bloomIndexer 进行赋值；

```go
bloomIndexer:   NewBloomIndexer(chainDb, params.BloomBitsBlocks, params.BloomConfirms), 
```

而在`LightEthereum`客户端中，使用了三种索引器:
**BloomIndexer**

```go
bloomIndexer: eth.NewBloomIndexer(chainDb, params.BloomBitsBlocksClient, params.HelperTrieConfirmations), 
```

**ChtIndexer**

```go
leth.chtIndexer = light.NewChtIndexer(chainDb, leth.odr, params.CHTFrequencyClient, params.HelperTrieConfirmations) 
```

**BloomTrieIndexer**

```go
leth.bloomTrieIndexer = light.NewBloomTrieIndexer(chainDb, leth.odr, params.BloomBitsBlocksClient, params.BloomTrieFrequency) 
```

虽然外在看来是三种类型索引器，但其内部构造方式却都是使用了同一个方法，只是具体类型初始化使用不同参数而已。

```go
core.NewChainIndexer(db, table, backend, size, confirms, bloomThrottling, "bloombits") 
```

所以，三个索引器在构建过程中是一样的，可分为以下三个步骤：

**首先** ，根据传递的参数构造 ChainIndex 对象。实例化过程使用到 chaindb、索引库、索引器实现、消息通道、section 大小、区块确认数、消息频率、日志类型。

```go
c := &ChainIndexer{
        chainDb:     chainDb,
        indexDb:     indexDb,
        backend:     backend, 
        update:      make(chan struct{}, 1),
        quit:        make(chan chan error),
        sectionSize: section,
        confirmsReq: confirm,
        throttling:  throttling,
        log:         log.New("type", kind),
} 
```

需要重点说明的三个成员变量`backend`、 `sectionSize`、 `confirmsReq`，backend 参数传递的是三种类型索引器的具体实现，三种索引器均扩展自`ChainIndexerBackend`接口, 分别实现了三个方法：`Reset`、`Process`、`Commit`；

```go
type ChainIndexerBackend interface {
    //重置数据
    Reset(ctx context.Context, section uint64, prevHead common.Hash) error
    //更新区块头数据至 trie 或缓存中
    Process(ctx context.Context, header *types.Header) error
    //将 trie 映射关系或缓存数据刷新至数据库中
    Commit() error
} 
```

`sectionSize`、 `confirmsReq`根据不同的索引实现所起作用也有所不同。

**其后**，调用了 c.loadValidSections()方法

```go
func (c *ChainIndexer) loadValidSections() {
    data, _ := c.indexDb.Get([]byte("count"))
    if len(data) == 8 {
        c.storedSections = binary.BigEndian.Uint64(data)
    }
} 
```

从数据库读取最近验证成功的索引段数据并初始化`c.storedSections`变量。

**最后**，新启动一个线程，执行`c.updateLoop()`方法。在该方法中主要轮询等待 update 消息，也就当一个 section 段中接收到 4096 个区块以及 section 段中最后一个区块也收到 256 个确认后触发此消息。在内部实现逻辑中，首先通过`processSection`生成新 section 中 lasthead 哈希值，即 newHead。

```go
newHead, err := c.processSection(section, oldHead) 
```

而后将 section 及 newHead 值更新至索引库。
在 c.processSection 方法中，主要是通过遍历 section 段的区块并调用索引器的`Process`同步更新索引缓存或 trie 树，遍历完后得到 section 段的最后一个区块 hash 值，最后调用索引器的`commit`方法将索引同步回数据库。

**那么，是哪个地方触发`update`通道消息的呢？**
其实就在初始化 Ethereum 或 LightEthereum 客户端时触发的，因为他们在初始化索引器后，便直接调用索引器启动方法`Start(chain ChainIndexerChain)`，通过监听链上事件，进而触发索引的创建流程。

```go
func (c *ChainIndexer) Start(chain ChainIndexerChain) {
    //ChainIndexerChain 实现有两种: BlockChain、LightChain, 所以在 SubscribeChainHeadEvent 方法具体实现中有所不同。
    events := make(chan ChainHeadEvent, 10)
    sub := chain.SubscribeChainHeadEvent(events)

    //异步线程：用于轮询链上事件消息，触发具体索引器创建流程
    go c.eventLoop(chain.CurrentHeader(), events, sub)
} 
```

此方法中主要完成两件事：
第一、订阅并监听 BlockChain 或 LightChain 上的区块事件消息。另外注意， `chain ChainIndexerChain)`是一个接口类，BlockChain、LightChain 便是它的实现类, 根据客户端(Ethereum/LightEthereum)的不同会传递不同的 ChainIndexerChain 实现

```go
type ChainIndexerChain interface {
    // CurrentHeader retrieves the latest locally known header.
    CurrentHeader() *types.Header

    // SubscribeChainHeadEvent subscribes to new head header notifications.
    SubscribeChainHeadEvent(ch chan<- ChainHeadEvent) event.Subscription
} 
```

第二、启动新线程，轮询链上事件消息，当收到新区块消息时，更新当前索引器中区块头数据，并发送`update`消息触发索引器中的索引创建流程。

```go
c.newHead(header.Number.Uint64(), false) 
```

至此，整个索引的创建流程就结束了。

**那么，三种索引器的具体实现有何不同呢？**
下面我们将分个讲解三种索引器的具体实现逻辑。前面介绍过三种索引器都扩展自一个接口`ChainIndexerBackend`，接口内定义了`Reset`、`Process`、`Commit`三个方法，那么我们也将逐个分析它们三个方法的实现。

### BloomIndexer 源码分析

#### Reset

该方法用于重置当前索引器。重新生成 BitGenerator，section 段容量长度，以前当前 head 区块哈希

```go
func (b *BloomIndexer) Reset(ctx context.Context, section uint64, lastSectionHead common.Hash) error {
    gen, err := bloombits.NewGenerator(uint(b.size))
    b.gen, b.section, b.head = gen, section, common.Hash{}
    return err
} 
```

索引在创建过程中，往往是收集到一定数量的区块后，把它们整合成一个 section 然后统一进行索引创建。之所以重置当前索引器，是因为在处理索引过程可能会碰到索引创建失败的情况，为了保持 section 数据的完整性，所以每次在更新索引之前都会重置索引器数据。

#### Process

该方法主要用于将索引数据更新至当前索引器实例变量

```go
b.gen.AddBloom(uint(header.Number.Uint64()-b.section*b.size), header.Bloom)
b.head = header.Hash() 
```

header.Bloom 是在创建区块时，通过 CreateBloom(receipts)方法将 receipts 执行交易结果中的 logs 转化为 Bloom 对象，最后将 Bloom 实例以 bit 的形式更新至当前索引器的`*bloombits.Generator`变量中。

那么它是如何以 bit 的形式存储更新的呢？
先看一下`Generator`的数据结构

```go
type Generator struct {
    //blooms 字节数据主要用于快速定位某个日志是否存在。布隆过滤器虽然不能百分百确认某个数据确实存在，但可以明确知道它不存在。
    blooms   [types.BloomBitLength][]byte // Rotated blooms for per-bit matching
    sections uint                         // Number of sections to batch together
    nextSec  uint                         // Next section to set when adding a bloom
} 
```

在该结构中，blooms 字段定义了一个 2048 长度的二维字节数组。
在调用`b.gen.AddBloom`存储入一个 Bloom 对象时，便是将 Bloom 实例按位的形式进行标识存储。

```go
//共计循环 2048 次，将 256 字节数据写入 b.blooms 变量中
for i := 0; i < types.BloomBitLength; i++ {
    //定位字节 index。 Bloom 对象共 256 个字节
    bloomByteIndex := types.BloomByteLength - 1 - i/8    
    bloomBitMask := byte(1) << byte(i%8)                 

    //标识字节位是否存在,每循环八次写入一个字节
    if (bloom[bloomByteIndex] & bloomBitMask) != 0 {    
        //用于或运算的形式进行字节赋值
        b.blooms[i][byteIndex] |= bitMask
    }
} 
```

为什么说通过位的形式可以快速判断某项数据是否存在或不存在呢？
举个简单示例
比如: 用长度为 4 的数组，按取模运算方式获取数字所存入的数组位置并标记为 1。
初始状态下，数组内容为: 0000
存储数字 2 时，2%4=2。数据内容为: 0100
存储数字 3 时，3%4=3。数组内容为: 1000
存储数字 6 时，6%4=2。数组内容为: 0100
当我要查询数字 1 存在不存在该数组时，1%4=1 发现数组下标为 1 的位置为 0，则意味着并不存在数字 1；那查询 14 是否存在于数组时，14%4=2，发现数组下标 2 位置为 1，那一定是存在吗? 不一定呢。因为在刚才存储数字 2、6 时也是存在此位置的。
所以说：通过按位检索可以快速判断某个值不存在；但却不可以完全肯定某项值存在。

#### Commit

该方法主要是为了将 Process 方法中的缓存索引数据持久化到数据库中。

```go
func (b *BloomIndexer) Commit() error {
    batch := b.db.NewBatch()
    for i := 0; i < types.BloomBitLength; i++ {
        bits, err := b.gen.Bitset(uint(i))
        if err != nil {
            return err
        }
        rawdb.WriteBloomBits(batch, uint(i), b.section, b.head, bitutil.CompressBytes(bits))
    }
    return batch.Write()
} 
```

### ChtIndexerBackend

从 ChtIndexerBackend 数据结构可以看出，其内部数据是以 trie 树进行存储的。

```go
type ChtIndexerBackend struct {
    diskdb, trieTable    ethdb.Database
    odr                  OdrBackend
    triedb               *trie.Database
    //params.CHTFrequencyClient,缺省值为 32768 params.HelperTrieConfirmations: 2048
    section, sectionSize uint64
    lastHash             common.Hash
    trie                 *trie.Trie
} 
```

还有一个需要重点关注的参数`OdrBackend`, 后面章节会进行详细介绍。

#### Reset

重置当前索引器成员数据。如果当前 section 为 0 则生成一颗根节点为空值的 trie 树，如果当前 section 大于 0 则从数据库中获取前一个 section 中的根节点数据并以此重新生成一个 trie 树

```go
func (c *ChtIndexerBackend) Reset(ctx context.Context, section uint64, lastSectionHead common.Hash) error {
    var root common.Hash
    if section > 0 {
        root = GetChtRoot(c.diskdb, section-1, lastSectionHead)
    }
    var err error
    c.trie, err = trie.New(root, c.triedb)

    if err != nil && c.odr != nil {
        err = c.fetchMissingNodes(ctx, section, root)
        if err == nil {
            c.trie, err = trie.New(root, c.triedb)
        }
    }

    c.section = section
    return err
} 
```

另外，在`trie.New`过程中如果出现在本地库找不到 trie 树节点的情况，那么会抛出例外，继而调用`c.fetchMissingNodes`发起 ChtRequest 请求去其他节点数据请求。这里的请求处理过程也是和`OdrBackend`相关，统一后续介绍。

#### Process

根据区块头哈希值以及区块高度从数据库查询当前块的总难度值，然后对`ChtNode{hash, td}`进行 RLP 编码，最后更新到 trie 树中。

```go
func (c *ChtIndexerBackend) Process(ctx context.Context, header *types.Header) error {
    hash, num := header.Hash(), header.Number.Uint64()
    c.lastHash = hash

    td := rawdb.ReadTd(c.diskdb, hash, num)
    if td == nil {
        panic(nil)
    }
    var encNumber [8]byte
    binary.BigEndian.PutUint64(encNumber[:], num)
    data, _ := rlp.EncodeToBytes(ChtNode{hash, td})
    c.trie.Update(encNumber[:], data)
    return nil
} 
```

#### Commit

首先通过`c.trie.Commit(nil)`生成当前 trie 树的根哈希值，然后将 trie 树内容进行数据库更新，最后调用方法`StoreChtRoot(c.diskdb, c.section, c.lastHash, root)`创建 cht 索引数据。

```go
func (c *ChtIndexerBackend) Commit() error {
    root, err := c.trie.Commit(nil)
    if err != nil {
        return err
    }
    c.triedb.Commit(root, false)

    if ((c.section+1)*c.sectionSize)%params.CHTFrequencyClient == 0 {
        log.Info("Storing CHT", "section", c.section*c.sectionSize/params.CHTFrequencyClient, "head", fmt.Sprintf("%064x", c.lastHash), "root", fmt.Sprintf("%064x", root))
    }
    StoreChtRoot(c.diskdb, c.section, c.lastHash, root)
    return nil
} 
```

### BloomTrieIndexerBackend

该索引器数据结构区别于其他索引器的地方是：使用 sectionHeads 哈希数组代替其他数据结构中的 lasthead 字段。

```go
type BloomTrieIndexerBackend struct {
    diskdb, trieTable ethdb.Database
    triedb            *trie.Database
    odr               OdrBackend
    section           uint64
    //params.BloomBitsBlocksClient, 缺省值为 32768
    parentSize        uint64
    //params.BloomTrieFrequency, 缺省值为 32768
    size              uint64
    //负载因子
    bloomTrieRatio    uint64
    trie              *trie.Trie
    sectionHeads      []common.Hash
} 
```

参数 bloomTrieRatio 在系统缺省配置的情况下等于 size/parentSize=1, 因为 parentSize= size= 32768。同样，关于 OdrBackend 的内容在也后续章节进行介绍。

#### Reset

重置当前索引器成员数据。如果当前 section 为 0 则生成一颗根节点为空值的 trie 树；如果当前 section 大于 0 则从数据库中获取前一个 section 中的根节点数据并以此重新生成一个 trie 树。

```go
func (b *BloomTrieIndexerBackend) Reset(ctx context.Context, section uint64, lastSectionHead common.Hash) error {
    var root common.Hash
    if section > 0 {
        //查询前缀为"bltIndex-"的数据
        root = GetBloomTrieRoot(b.diskdb, section-1, lastSectionHead)
    }
    var err error
    b.trie, err = trie.New(root, b.triedb)
    if err != nil && b.odr != nil {
        //所发起的 Request 为 BloomRequest
        err = b.fetchMissingNodes(ctx, section, root)
        if err == nil {
            b.trie, err = trie.New(root, b.triedb)
        }
    }
    b.section = section
    return err
} 
```

#### Process

根据当前区块高度值计算在当前 section 中所在位置，当 section 处理的区块数量达到 section 空间长度时，将最后一个区块的区块头哈希值更新到 b.sectionHeads 指定 section 位置

```go
func (b *BloomTrieIndexerBackend) Process(ctx context.Context, header *types.Header) error {
    //取当前 section 段，区块所在位置
    num := header.Number.Uint64() - b.section*b.size
    //将 section 段中最后一个区块的区块哈希更新至 sectionHeads 中，每个 section 长度为 b.parentSize。
    if (num+1)%b.parentSize == 0 {
        b.sectionHeads[num/b.parentSize] = header.Hash()
    }
    return nil
} 
```

#### Commit

根据当前 section 查询 BloomIndexer 写入数据库的索引数据信息，使用`bitutil.CompressBytes(decomp)`方法压缩索引数据，然后将压缩数据更新到当前索引器的 trie 树中，调用`b.trie.Commit(nil)`生成当前树的根哈希值，最后调用`StoreBloomTrieRoot(b.diskdb, b.section, sectionHead, root)`方法生成 BloomTrie 索引。

```go
func (b *BloomTrieIndexerBackend) Commit() error {
    var compSize, decompSize uint64

    for i := uint(0); i < types.BloomBitLength; i++ {
        var encKey [10]byte
        binary.BigEndian.PutUint16(encKey[0:2], uint16(i))//位 index
        binary.BigEndian.PutUint64(encKey[2:10], b.section) //段 id
        var decomp []byte
        for j := uint64(0); j < b.bloomTrieRatio; j++ { //　backend.bloomTrieRatio = size / parentSize，缺省值情况下为１
            data, err := rawdb.ReadBloomBits(b.diskdb, i, b.section*b.bloomTrieRatio+j, b.sectionHeads[j])
            if err != nil {
                return err
            }
            decompData, err2 := bitutil.DecompressBytes(data, int(b.parentSize/8))
            if err2 != nil {
                return err2
            }
            decomp = append(decomp, decompData...)
        }
        comp := bitutil.CompressBytes(decomp)

        decompSize += uint64(len(decomp))
        compSize += uint64(len(comp))
        if len(comp) > 0 {
            b.trie.Update(encKey[:], comp)
        } else {
            b.trie.Delete(encKey[:])
        }
    }
    root, err := b.trie.Commit(nil)
    if err != nil {
        return err
    }
    b.triedb.Commit(root, false)

    sectionHead := b.sectionHeads[b.bloomTrieRatio-1]
    log.Info("Storing bloom trie", "section", b.section, "head", fmt.Sprintf("%064x", sectionHead), "root", fmt.Sprintf("%064x", root), "compression", float64(compSize)/float64(decompSize))
    StoreBloomTrieRoot(b.diskdb, b.section, sectionHead, root)
    return nil
} 
```

* * *

## 总结

这一节我们主要分析了 ChainIndexer 的数据处理流程，明白索引的创建是在监听到新区块产生事件后，统计当前 section 是否已经收满足 section 长度的区块且保证每个区块都已经收到 256 个确认后，通过调用具体的索引器进行缓存或 trie 树更新，最后将索引数据更新至数据库的一系统流程。
另外，我们也学习到了三种索引器的接口定义与具体实现，明白它们所要处理的业务等。

* * *

> 在教程中如出现不易理解或存在错误的问题🐛，欢迎加我微信指正！
> Name: zhangliang | WeChat: rushking2009 | Mail: zhangliang@cldy.org