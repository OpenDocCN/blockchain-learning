# 第二章 牛刀小试-创建最简单的区块链

# 第二章 基本原型(basic-prototype)

## 1\. 课程目标

1.  了解区块链的结构
2.  学会创建一个区块(Block)
3.  学会创建区块链(BlockChain)
4.  学会向一个区块链上添加新的区块

## 2\. 项目代码及效果展示

### 2.1 项目代码结构

![`img.kongyixueyuan.com/01_%E9%A1%B9%E7%9B%AE%E7%BB%93%E6%9E%84.png`](img/afd3d80b834a007e8c6cd7197dd2d796.jpg)

### 2.2 项目运行结果

#### 2.2.1 创建一个区块效果

![`img.kongyixueyuan.com/02_%E8%BF%90%E8%A1%8C%E7%BB%93%E6%9E%9C1.gif`](img/724699776059703a8da0cf39c9dc4698.jpg)

#### 2.2.2 创建创世区块效果

![`img.kongyixueyuan.com/03_%E8%BF%90%E8%A1%8C%E7%BB%93%E6%9E%9C2.gif`](img/4b14909227e5e51e9f2ced67b526dc00.jpg)

#### 2.2.3 创建区块链效果

![`img.kongyixueyuan.com/04_%E8%BF%90%E8%A1%8C%E7%BB%93%E6%9E%9C3.gif`](img/ec485e17a74e294e9ae28dd9a3d10432.jpg)

#### 2.2.4 向一个区块链中添加新区块效果

![`img.kongyixueyuan.com/05_%E8%BF%90%E8%A1%8C%E7%BB%93%E6%9E%9C4.gif`](img/e55563a0f6b7e64dac87957a4cec8584.jpg)

## 3\. 创建项目

### 3.1 创建工程

首先打开 Goland 开发工具

新建工程：`mypublicchain`

创建目录：`day01_01_Base_Prototype`

![`img.kongyixueyuan.com/06_%E5%88%9B%E5%BB%BA%E9%A1%B9%E7%9B%AE.gif`](img/e10fdd7d4231bb1da0fe75239a7ece2f.jpg)

### 3.2 代码实现

#### 3.2.1 创建`Block.go`

在`day01_01_Base_Prototype`目录下，再创建一个目录作为包命名为 BLC。

在该目录下创建一个文件`Block.go`。

![`img.kongyixueyuan.com/07_%E5%88%9B%E5%BB%BA%E6%96%87%E4%BB%B6.gif`](img/fabfdd59ae537f9e536cd10cf35dc5bc.jpg)

在`Block.go`文件中编写代码如下：

```go
package BLC

import (
    "time"
    "strconv"
    "bytes"
    "crypto/sha256"
)

//step1:创建 Block 结构体
type Block struct {
    //字段：
    //高度 Height：其实就是区块的编号，第一个区块叫创世区块，高度为 0
    Height int64
    //上一个区块的哈希值 ProvHash：
    PrevBlockHash []byte
    //交易数据 Data：目前先设计为[]byte,后期是 Transaction
    Data [] byte
    //时间戳 TimeStamp：
    TimeStamp int64
    //哈希值 Hash：32 个的字节，64 个 16 进制数
    Hash []byte
}

//step2：创建新的区块
func NewBlock(data string,provBlockHash []byte,height int64) *Block{
    //创建区块
    block:=&Block{height,provBlockHash,[]byte(data),time.Now().Unix(),nil}
    //设置哈希值
    block.SetHash()
    return block
}

//step3:设置区块的 hash
func (block *Block) SetHash(){
    //1.将高度转为字节数组
    heightBytes:= IntToHex(block.Height)
    //fmt.Println(heightBytes)
    //2.时间戳转为字节数组
    //timeBytes:=IntToHex(block.TimeStamp)
    //转为二进制的字符串
    //fmt.Println(block.TimeStamp)
    //fmt.Printf("%x,%b\n",block.TimeStamp,block.TimeStamp)
    timeString := strconv.FormatInt(block.TimeStamp,2)
    //fmt.Println("timeString:",timeString)
    timeBytes := [] byte(timeString)
    //fmt.Println("timeStamp:",timeBytes)
    //3.拼接所有的属性
    blockBytes:= bytes.Join([][]byte{
        heightBytes,
        block.PrevBlockHash,
        block.Data,
        timeBytes},[]byte{})
    //4.生成哈希值
    hash:=sha256.Sum256(blockBytes)//数组长度 32 位
    block.Hash = hash[:]
}

//step4:创建创世区块：
func CreateGenesisBlock(data string) *Block{
    return NewBlock(data,make([] byte,32,32),0)
}
```

#### 3.2.2 创建`utils.go`

在 BLC 包下，再创建一个 go 文件，命名为：`utils.go`。里面编写所需要的各种工具方法。

编写代码如下：

```go
package BLC

import (
    "bytes"
    "encoding/binary"
    "log"
)

/*
将一个 int64 的整数：转为二进制后，每 8bit 一个 byte。转为[]byte
 */
func IntToHex(num int64) []byte {
    buff := new(bytes.Buffer)
    //将二进制数据写入 w
    //func Write(w io.Writer, order ByteOrder, data interface{}) error
    err := binary.Write(buff, binary.BigEndian, num)
    if err != nil {
        log.Panic(err)
    }
    //转为[]byte 并返回
    return buff.Bytes()
} 
```

#### 3.2.3 创建`BlockChain.go`

在 BLC 包下，再创建一个 go 文件，`BlockChain.go`。

编写如下代码：

```go
package BLC

//step5:创建区块链
type BlockChain struct {
    Blocks []*Block //存储有序的区块
}

//step6：创建区块链，带有创世区块
func CreateBlockChainWithGenesisBlock(data string) *BlockChain{
    //创建创世区块
    genesisBlock := CreateGenesisBlock(data)
    //返回区块链对象
    return &BlockChain{[]*Block{genesisBlock}}
}

//step7：添加一个新的区块，到区块链中
func (bc *BlockChain) AddBlockToBlockChain(data string,height int64,prevHash [] byte){
    //创建新区块
    newBlock := NewBlock(data,prevHash,height)
    //添加到切片中
    bc.Blocks = append(bc.Blocks,newBlock)
} 
```

#### 3.2.4 创建`main.go`

在`day01_01_Base_Prototype`目录下，直接创建`main.go`。

并编写如下代码：

```go
package main

import (
    "./BLC"
    "fmt"
)

func main() {
    //1.测试 Block
    //block:=BLC.NewBlock("I am a block",make([]byte,32,32),1)
    //fmt.Println(block)

    //2.测试创世区块
    //genesisBlock :=BLC.CreateGenesisBlock("Genesis Block..")
    //fmt.Println(genesisBlock)

    //3.测试区块链
    //genesisBlockChain := BLC.CreateBlockChainWithGenesisBlock()
    //fmt.Println(genesisBlockChain)
    //fmt.Println(genesisBlockChain.Blocks)
    //fmt.Println(genesisBlockChain.Blocks[0])

    //4.测试添加新区块
    blockChain:=BLC.CreateBlockChainWithGenesisBlock("Genesis Block..")
    blockChain.AddBlockToBlockChain("Send 100RMB To Wangergou",blockChain.Blocks[len(blockChain.Blocks)-1].Height+1,blockChain.Blocks[len(blockChain.Blocks)-1].Hash)
    blockChain.AddBlockToBlockChain("Send 300RMB To lixiaohua",blockChain.Blocks[len(blockChain.Blocks)-1].Height+1,blockChain.Blocks[len(blockChain.Blocks)-1].Hash)
    blockChain.AddBlockToBlockChain("Send 500RMB To rose",blockChain.Blocks[len(blockChain.Blocks)-1].Height+1,blockChain.Blocks[len(blockChain.Blocks)-1].Hash)

    fmt.Println(blockChain)
} 
```

## 4\. 项目基本原型讲解

### 4.1\. 区块 Block

首先从 “区块” 谈起。在区块链中，真正存储有效信息的是区块（block）。而在比特币中，真正有价值的信息就是交易（transaction）。实际上，交易信息是所有加密货币的价值所在。除此以外，区块还包含了一些技术实现的相关信息，比如版本，当前时间戳和前一个区块的哈希。

#### 4.1.1 区块结构

不过，我们要实现的是一个简化版的区块链，而不是一个像比特币技术规范所描述那样成熟完备的区块链。所以在我们目前的实现中，区块仅包含了部分关键信息，它的数据结构如下(在`Block.go`文件中)：

创建`Block.go`文件，并添加`Block`的结构体，代码如下：

```go
//step1:创建 Block 结构体
type Block struct {
    //字段：
    //高度 Height：其实就是区块的编号，第一个区块叫创世区块，高度为 0
    Height int64
    //上一个区块的哈希值 ProvHash：
    PrevBlockHash []byte
    //交易数据 Data：目前先设计为[]byte,后期是 Transaction
    Data [] byte
    //时间戳 TimeStamp：
    TimeStamp int64
    //哈希值 Hash：32 个的字节，64 个 16 进制数
    Hash []byte
}
```

说明一下：

| 字段 | 解释 |
| --- | --- |
| `Height` | 当前区块的高度，就是区块的编号 |
| `PrevBlockHash` | 前一个块的哈希，即父哈希 |
| `Data` | 区块存储的实际有效信息，也就是交易。随着项目的逐步完善后期会将 Data 修改为 Transaction。 |
| `Timestamp` | 当前时间戳，也就是区块创建的时间 |
| `Hash` | 当前块的哈希 |

> 什么是创世区块？
> 
> 区块链中的第一个区块，也就是高度 Heigth 为 0 的区块，往往叫做创世区块。

#### 4.1.2 对比比特币

我们这里的 `Timestamp`，`PrevBlockHash`, `Hash`，在比特币技术规范中属于区块头（`block header`），区块头是一个单独的数据结构。

![`img.kongyixueyuan.com/08_block.png`](img/f977d41d475dddeaf022091c209169c0.jpg)

完整的比特币的区块头（`block header`）结构如下：

| Field | Purpose | Updated when... | Size (Bytes) |
| --- | --- | --- | --- |
| Version | Block version number | You upgrade the software and it specifies a new version | 4 |
| hashPrevBlock | 256-bit hash of the previous block header | A new block comes in | 32 |
| hashMerkleRoot | 256-bit hash based on all of the transactions in the block | A transaction is accepted | 32 |
| Time | Current timestamp as seconds since 1970-01-01T00:00 UTC | Every few seconds | 4 |
| Bits | Current target in compact format | The difficulty is adjusted | 4 |
| Nonce | 32-bit number (starts at 0) | A hash is tried (increments) | 4 |

而我们的 `Data`, 在比特币中对应的是交易，是另一个单独的数据结构。为了简便起见，目前将这两个数据结构放在了一起。在真正的比特币中，区块的数据结构如下：

| Field | Description | Size |
| --- | --- | --- |
| Magic no | value always 0xD9B4BEF9 | 4 bytes |
| Blocksize | number of bytes following up to end of block | 4 bytes |
| Blockheader | consists of 6 items | 80 bytes |
| Transaction counter | positive integer VI = VarInt | 1 - 9 bytes |
| transactions | the (non empty) list of transactions | <transaction counter="">-many transactions</transaction> |

![`img.kongyixueyuan.com/09_blockchain.png`](img/676b6052f9f8170c4aba95a579c2993c.jpg)

```go
 区块链的结构
```

### 4.2 计算 Hash

在我们的简化版区块中，还有一个 `Hash` 字段，那么，要如何计算哈希呢？哈希计算，是区块链一个非常重要的部分。正是由于它，才保证了区块链的安全。计算一个哈希，是在计算上非常困难的一个操作。即使在高速电脑上，也要耗费很多时间 (这就是为什么人们会购买 GPU，FPGA，ASIC 来挖比特币) 。这是一个架构上有意为之的设计，它故意使得加入新的区块十分困难，继而保证区块一旦被加入以后，就很难再进行修改。在接下来的内容中，我们将会讨论和实现这个机制。

#### 4.2.1 什么是 Hash

在本节，我们会讨论哈希计算。如果你已经熟悉了这个概念，可以直接跳过。

获得指定数据的一个哈希值的过程，就叫做哈希计算。一个哈希，就是对所计算数据的一个唯一表示。对于一个哈希函数，输入任意大小的数据，它会输出一个固定大小的哈希值。下面是哈希的几个关键特性：

1.  无法从一个哈希值恢复原始数据。也就是说，哈希并不是加密。
2.  对于特定的数据，只能有一个哈希，并且这个哈希是唯一的。
3.  即使是仅仅改变输入数据中的一个字节，也会导致输出一个完全不同的哈希。

![`img.kongyixueyuan.com/15_hash2.png`](img/1daeb949433acddcc49d8bf4ebd66470.jpg)

所以说，`hash`是一种算法， 是将文本转换成具有相同长度的，不可逆的杂凑字符串散列(很多资料也叫指纹或叫消息摘要`Message Digest`)。本身词义"切碎，剁碎"。也可以说，`hash`就是找到一种数据内容和数据存放地址之间的映射关系。

哈希函数被广泛用于检测数据的一致性。软件提供者常常在除了提供软件包以外，还会发布校验和。当下载完一个文件以后，你可以用哈希函数对下载好的文件计算一个哈希，并与作者提供的哈希进行比较，以此来保证文件下载的完整性。

#### 4.2.2 Hash 算法

`Hash`算法有很多种，本文中只介绍了项目中使用的 SHA256S 算法。

**安全散列算法 SHA：**

安全散列算法 SHA（Secure Hash Algorithm）是美国国家安全局 （NSA） 设计，美国国家标准与技术研究院（NIST） 发布的一系列密码散列函数，包括 SHA-1、SHA-224、SHA-256、SHA-384 和 SHA-512 等变体。主要适用于数字签名标准（DigitalSignature Standard DSS）里面定义的数字签名算法（Digital Signature Algorithm DSA）。SHA-1 已经不是那么安全了，google 和微软都已经弃用这个加密算法。为此，我们使用热门的比特币使用过的算法 SHA-256 作为实例。其它 SHA 算法，也可以按照这种模式进行使用。

> 著名的 Hash 算法：
> 
> 1.  MD4(RFC 1320)是 MIT 的 Ronald L. Rivest 在 1990 年设计的，MD 是 Message Digest 的缩写。它适用在 32 位字长的处理器上用快速软件实现–它是基于 32 位操作数的位操作来实现的。
> 2.  MD5(RFC 1321)是 Rivest 于 1991 年对 MD4 的改进版本号。它对输入仍以 512 位分组，其输出是 4 个 32 位字的级联，与 MD4 同样。MD5 比 MD4 来得复杂，而且速度较之要慢一点，但更安全，在抗分析和抗差分方面表现更好

#### 4.2.3 代码实现

**拼接字段**

目前，我们仅取了 `Block` 结构的部分字段（`Height`,`Timestamp`, `Data` 和 `PrevBlockHash`），并将它们相互拼接起来，然后在拼接后的结果上计算一个 SHA-256，然后就得到了哈希.

因为`Height`是一个整型数据，我们需要将它转为`[]byte`

`utils.go`文件中：

```go
/*
将一个 int64 的整数：转为二进制后，每 8bit 一个 byte。转为[]byte
 */
func IntToHex(num int64) []byte {
    buff := new(bytes.Buffer)
    //将二进制数据写入 w
    //func Write(w io.Writer, order ByteOrder, data interface{}) error
    err := binary.Write(buff, binary.BigEndian, num)
    if err != nil {
        log.Panic(err)
    }
    //转为[]byte 并返回
    return buff.Bytes()
}
```

> 说明：在比特币种并没有拼接`Heigth`这个字段属性。但是这不重要，我们拼接字段只是为了生成`Hash`。

**设置 Hash**

在 `SetHash` 方法中完成这些操作：

`Block.go`文件中：

```go
// 设置区块的 hash
func (block *Block) SetHash(){
    //1.将高度转为字节数组
    heightBytes:= IntToHex(block.Height)
    //fmt.Println(heightBytes)
    //2.时间戳转为字节数组
    //timeBytes:=IntToHex(block.TimeStamp)
    //转为二进制的字符串
    //fmt.Println(block.TimeStamp)
    //fmt.Printf("%x,%b\n",block.TimeStamp,block.TimeStamp)
    timeString := strconv.FormatInt(block.TimeStamp,2)
    //fmt.Println("timeString:",timeString)
    timeBytes := [] byte(timeString)
    //fmt.Println("timeStamp:",timeBytes)
    //3.拼接所有的属性
    blockBytes:= bytes.Join([][]byte{
        heightBytes,
        block.PrevBlockHash,
        block.Data,
        timeBytes},[]byte{})
    //4.生成哈希值
    hash:=sha256.Sum256(blockBytes)//数组长度 32 位
    block.Hash = hash[:]
} 
```

### 4.3 创建区块

#### 4.3.1 创建一个新的区块

接下来，我们会实现一个用于简化创建区块的函数 `NewBlock()`：

`Block.go`文件中：

```go
//创建新的区块
func NewBlock(data string,provBlockHash []byte,height int64) *Block{
    //创建区块
    block:=&Block{height,provBlockHash,[]byte(data),time.Now().Unix(),nil}
    //设置哈希值
    block.SetHash()
    return block
}
```

在`main.go`文件中添加 main 函数，并进行测试：

```go
func main() {
    //1.测试 Block
    block:=BLC.NewBlock("I am a block",make([]byte,32,32),1)
    fmt.Printf("Heigth:%x\n",block.Height)
    fmt.Printf("Data:%s\n",block.Data)
}
```

进行测试：

```go
打开终端：点击 goland 底部的 Terminal：

输入命令：进入 day01_01_Base_Prototype 目录
    hanru:mypublicchain ruby$ cd day01_01_Base_Prototype/
    hanru:day01_01_Base_Prototype ruby$ 

查看该目录下的内容:
    hanru:day01_01_Base_Prototype ruby$ ls
    BLC     main.go

执行程序：
    hanru:day01_01_Base_Prototype ruby$ go run main.go
```

运行结果：

![`img.kongyixueyuan.com/10_%E6%89%A7%E8%A1%8C%E7%BB%93%E6%9E%9C.jpg`](img/74dff5b68707e2d0254b3a69db00dcc6.jpg)

#### 4.3.2 创建一个创世区块

创世区块因为是区块链中的第一个区块，还是有一些特殊的，比如`Heigth`固定为 0，`PrevBlockHash`固定为 0。

```go
//创建创世区块：
func CreateGenesisBlock(data string) *Block{
    return NewBlock(data,make([] byte,32,32),0)
}
```

在 main 函数中进行测试：

```go
 func main() {
    //1.测试 Block
    //block:=BLC.NewBlock("I am a block",make([]byte,32,32),1)
    //fmt.Printf("Heigth:%x\n",block.Height)
    //fmt.Printf("Data:%s\n",block.Data)
    //2.测试创世区块
    genesisBlock :=BLC.CreateGenesisBlock("Genesis Block..")
    fmt.Printf("Heigth:%x\n",genesisBlock.Height)
    fmt.Printf("PrevBlockHash:%x\n",genesisBlock.PrevBlockHash)
    fmt.Printf("Data:%s\n",genesisBlock.Data)
} 
```

运行结果：

![`img.kongyixueyuan.com/11_%E6%89%A7%E8%A1%8C%E7%BB%93%E6%9E%9C.png`](img/dbb4f1cde472d003aca68b775142d222.jpg)

### 4.4\. 区块链 BlockChain

有了区块，下面让我们来实现区块**链**。本质上，区块链就是一个有着特定结构的数据库，是一个有序，每一个块都连接到前一个块的链表。也就是说，区块按照插入的顺序进行存储，每个块都与前一个块相连。这样的结构，能够让我们快速地获取链上的最新块，并且高效地通过哈希来检索一个块。

![`img.kongyixueyuan.com/14_blockchain2.png`](img/26446113c3e416d8b10d2d5b3614c661.jpg)

#### 4.4.1 定义区块链

在 Golang 中，可以通过一个 `array` 和 `map` 来实现这个结构：`array` 存储有序的哈希（Golang 中 `array` 是有序的），`map` 存储 **hash -> block** 对(Golang 中, `map` 是无序的)。 但是在基本的原型阶段，我们只用到了 `array`，因为现在还不需要通过哈希来获取块。

`BlockChain.go`文件中：

```go
//定义区块链
type BlockChain struct {
    Blocks []*Block //存储有序的区块
}
```

#### 4.4.2 创建一个区块链

为了创建一个区块链，并且能够加入一个新的块，我们必须要有一个已有的块，因为新区块需要引用之前的区块`Hash`作为`PrevBlockHash`。但是，初始状态下，我们的链是空的，一个块都没有！所以，在任何一个区块链中，都必须至少有一个块。这个块，也就是链中的第一个块，通常叫做创世区块，简称叫创世块（**genesis block**），而创世区块的`PrevBlockHash`固定为 0。

接下来，我们提供一个方法，用于创建一个区块链，并且该区块链中包含了创世区块。

`BlockChain.go`文件中：

```go
//创建区块链，带有创世区块
func CreateBlockChainWithGenesisBlock(data string) *BlockChain{
    //创建创世区块
    genesisBlock := CreateGenesisBlock(data)
    //返回区块链对象
    return &BlockChain{[]*Block{genesisBlock}}
}
```

这就是我们的第一个区块链！是不是出乎意料地简单? 就是一个 `Block` 数组。

在 main 函数中进行测试：

```go
func main() {
    //1.测试 Block
    //block:=BLC.NewBlock("I am a block",make([]byte,32,32),1)
    //fmt.Printf("Heigth:%x\n",block.Height)
    //fmt.Printf("Data:%s\n",block.Data)
    //2.测试创世区块
    //genesisBlock :=BLC.CreateGenesisBlock("Genesis Block..")
    //fmt.Printf("Heigth:%x\n",genesisBlock.Height)
    //fmt.Printf("PrevBlockHash:%x\n",genesisBlock.PrevBlockHash)
    //fmt.Printf("Data:%s\n",genesisBlock.Data)

    //3.测试区块链
    genesisBlockChain := BLC.CreateBlockChainWithGenesisBlock("Genesis Block..")
    fmt.Println(genesisBlockChain)
    fmt.Println(genesisBlockChain.Blocks)
    fmt.Printf("Heigth:%x\n",genesisBlockChain.Blocks[0].Height)
    fmt.Printf("PrevBlockHash:%x\n",genesisBlockChain.Blocks[0].PrevBlockHash)
    fmt.Printf("Data:%s\n",genesisBlockChain.Blocks[0].Data)
}
```

运行结果：

![`img.kongyixueyuan.com/12_%E6%89%A7%E8%A1%8C%E7%BB%93%E6%9E%9C.png`](img/a91c23f5c5836efcaf697991a5548d8b.jpg)

#### 4.4.3 添加新区块

现在，让我们能够给它添加一个区块：

`BlockChain.go`文件中：

```go
//添加一个新的区块，到区块链中
func (bc *BlockChain) AddBlockToBlockChain(data string,height int64,prevHash [] byte){
    //创建新区块
    newBlock := NewBlock(data,prevHash,height)
    //添加到切片中
    bc.Blocks = append(bc.Blocks,newBlock)
}
```

在 main 函数中进行测试：

```go
func main() {
    //4.测试添加新区块
    blockChain:=BLC.CreateBlockChainWithGenesisBlock("Genesis Block..")
    blockChain.AddBlockToBlockChain("Send 1BTC To Wangergou",blockChain.Blocks[len(blockChain.Blocks)-1].Height+1,blockChain.Blocks[len(blockChain.Blocks)-1].Hash)
    blockChain.AddBlockToBlockChain("Send 3BTC To lixiaohua",blockChain.Blocks[len(blockChain.Blocks)-1].Height+1,blockChain.Blocks[len(blockChain.Blocks)-1].Hash)
    blockChain.AddBlockToBlockChain("Send 5BTC To rose",blockChain.Blocks[len(blockChain.Blocks)-1].Height+1,blockChain.Blocks[len(blockChain.Blocks)-1].Hash)

    for _, block := range blockChain.Blocks {
        fmt.Printf("Prev. hash: %x\n", block.PrevBlockHash)
        fmt.Printf("Data: %s\n", block.Data)
        fmt.Printf("Hash: %x\n", block.Hash)
        fmt.Println()
    }
}
```

运行结果：

![`img.kongyixueyuan.com/13_%E6%89%A7%E8%A1%8C%E7%BB%93%E6%9E%9C.png`](img/b4e1981961701258a66aa3f14628e473.jpg)

## 5\. 总结

我们创建了一个非常简单的区块链原型：它仅仅是一个数组构成的一系列区块，每个块都与前一个块相关联。真实的区块链要比这复杂得多。在我们的区块链中，加入新的块非常简单，也很快，但是在真实的区块链中，加入新的块需要很多工作：你必须要经过十分繁重的计算（这个机制叫做工作量证明），来获得添加一个新块的权力。并且，区块链是一个分布式数据库，并且没有单一决策者。因此，要加入一个新块，必须要被网络的其他参与者确认和同意（这个机制叫做共识（consensus））。

[项目源代码](https://github.com/rubyhan1314/PublicChain)