# 第四章 区块持久化存储

# 持久化(Persistence)

到目前为止，我们已经构建了一个有工作量证明机制的区块链。有了工作量证明，挖矿也就有了着落。虽然目前距离一个有着完整功能的区块链越来越近了，但是它仍然缺少了一些重要的特性。在今天的内容中，我们会将区块链持久化到一个数据库中，然后会提供一个简单的命令行接口，用来完成一些与区块链的交互操作。本质上，区块链是一个分布式数据库，不过，我们暂时先忽略 “分布式” 这个部分，仅专注于 “存储” 这一点。

## 1\. 课程目标

1.  了解什么是持久化
2.  学会在使用 boltDB 数据库
3.  学会 Block 区块对象的序列化和反序列化
4.  项目中的区块能够持久化存储
5.  学会使用迭代器遍历区块

## 2\. 项目代码及效果展示

### 2.1 项目代码结构

![`img.kongyixueyuan.com/0401_%E9%A1%B9%E7%9B%AE%E7%BB%93%E6%9E%84.png`](img/d87d2da2f36c143f7f250d9ea86b9ab7.jpg)

### 2.2 项目运行结果

![`img.kongyixueyuan.com/0402_%E8%BF%90%E8%A1%8C%E7%BB%93%E6%9E%9C.gif`](img/641782592241f8448fc8d3a67b6c216f.jpg)

## 3\. 创建项目

### 3.1 创建工程

首先打开 Goland 开发工具

打开工程：`mypublicchain`

创建项目：将上一次的项目代码，`day01_02_Proof_Of_Work`，复制为`day02_03_Persistence`

> 说明：我们每一章节的项目代码，都是在上一个章节上进行添加。所以拷贝上一次的项目代码，然后进行新内容的添加或修改。

### 3.2 代码实现

#### 3.2.1 创建 go 文件：`Constant.go`

在`Constant.go`文件中编写代码如下：

```go
package BLC

const DBNAME = "blockchain.db"  //数据库名
const BLOCKTABLENAME = "blocks" //表名 
```

#### 3.2.2 创建 go 文件：`BlockChainIterator.go`

在`BlockChainIterator.go`文件中编写代码如下：

```go
package BLC

import (
    "github.com/boltdb/bolt"
    "log"
)

//新增一个结构体
type BlockChainIterator struct {
    CurrentHash [] byte //当前区块的 hash
    DB          *bolt.DB //数据库
}

//获取区块
func (bcIterator *BlockChainIterator) Next() *Block {
    block:=new(Block)
    //1.打开数据库并读取
    err :=bcIterator.DB.View(func(tx *bolt.Tx) error {
        //2.打开数据表
        b:=tx.Bucket([]byte(BLOCKTABLENAME))
        if b != nil{
            //3.根据当前 hash 获取数据并反序列化
            blockBytes:=b.Get(bcIterator.CurrentHash)
            block = DeserializeBlock(blockBytes)
            //4.更新当前的 hash
            bcIterator.CurrentHash = block.PrevBlockHash
        }

        return nil
    })
    if err != nil{
        log.Panic(err)
    }
    return block
} 
```

#### 3.2.3 修改`Block.go`文件

打开`day02_03_Persistence`目录里的 BLC 包。修改`Block.go`文件。

```go
修改步骤：
step1：添加序列化方法 Serilalize()
step2：添加反序列化函数 DeserializeBlock() 
```

修改完后代码如下:

```go
package BLC

import (
    "time"
    "bytes"
    "encoding/gob"
    "log"
)

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

    Nonce int64
}

func NewBlock(data string,provBlockHash []byte,height int64) *Block{
    //创建区块
    block:=&Block{height,provBlockHash,[]byte(data),time.Now().Unix(),nil,0}
    //step5：设置 block 的 hash 和 nonce
    //设置哈希
    //block.SetHash()
    //调用工作量证明的方法，并且返回有效的 Hash 和 Nonce
    pow:=NewProofOfWork(block)
    hash,nonce:=pow.Run()
    block.Hash = hash
    block.Nonce = nonce

    return block
}

func CreateGenesisBlock(data string) *Block{
    return NewBlock(data,make([] byte,32,32),0)
}

//将区块序列化，得到一个字节数组---区块的行为，设计为方法
func (block *Block) Serilalize() []byte {
    //1.创建一个 buffer
    var result bytes.Buffer
    //2.创建一个编码器
    encoder := gob.NewEncoder(&result)
    //3.编码--->打包
    err := encoder.Encode(block)
    if err != nil {
        log.Panic(err)
    }
    return result.Bytes()
}

//反序列化，得到一个区块---设计为函数
func DeserializeBlock(blockBytes [] byte) *Block {
    var block Block
    var reader = bytes.NewReader(blockBytes)
    //1.创建一个解码器
    decoder := gob.NewDecoder(reader)
    //解包
    err := decoder.Decode(&block)
    if err != nil {
        log.Panic(err)
    }
    return &block
} 
```

#### 3.2.4 修改`BlockChain.go`文件

打开`day02_03_Persistence`目录里的 BLC 包。修改`BlockChain.go`文件。

```go
修改步骤：
step1：修改 BlockChain 的结构体
    设置 Tip 字段，DB 字段
step2：添加函数 dbExists()，判断数据库是否存在
step3：修改 CreateBlockChainWithGenesisBlock()函数
    测试数据库是否存在
    如果数据库存在，直接从数据库中读取
    如果数据库不存在，创建创世区块并存入到数据中。
step4：修改 AddBlockToBlockChain()方法
    将新创建的区块，存入到数据库中
step5：添加方法 PrintChains(),用于遍历数据库中的所有的区块信息 
```

修改完后代码如下：

```go
package BLC

import (
    "github.com/boltdb/bolt"
    "os"
    "fmt"
    "log"
    "math/big"
    "time"
)

//step1：修改 BlockChain 的结构体
type BlockChain struct {
    //Blocks []*Block //存储有序的区块
    Tip [] byte  // 最后区块的 Hash 值
    DB  *bolt.DB //数据库对象
}

//step2：修改该方法
func CreateBlockChainWithGenesisBlock(data string) *BlockChain {
    //1.先判断数据库是否存在，如果有，从数据库读取
    if dbExists() {
        fmt.Println("数据库已经存在。。")
        //A：打开数据库
        db, err := bolt.Open(DBNAME, 0600, nil)
        if err != nil {
            log.Fatal(err)
        }
        //defer db.Close()
        var blockchain *BlockChain
        //B：读取数据库
        err = db.View(func(tx *bolt.Tx) error {
            //C：打开表
            b := tx.Bucket([]byte(BLOCKTABLENAME))
            if b != nil {
                //D：读取最后一个 hash
                hash := b.Get([]byte("l"))
                //E：创建 blockchain
                blockchain = &BlockChain{hash, db}
            }
            return nil
        })
        if err != nil {
            log.Fatal(err)
        }
        return blockchain
    }

    //2.数据库不存在，说明第一次创建，然后存入到数据库中
    fmt.Println("数据库不存在。。")
    //A：创建创世区块
    //创建创世区块
    genesisBlock := CreateGenesisBlock(data)
    //B：打开数据库
    db, err := bolt.Open(DBNAME, 0600, nil)
    if err != nil {
        log.Fatal(err)
    }
    //defer db.Close()
    //C：存入数据表
    err = db.Update(func(tx *bolt.Tx) error {
        b, err := tx.CreateBucket([]byte(BLOCKTABLENAME))
        if err != nil {
            log.Panic(err)
        }
        if b != nil {
            err = b.Put(genesisBlock.Hash, genesisBlock.Serilalize())
            if err != nil {
                log.Panic("创世区块存储有误。。。")
            }
            //存储最新区块的 hash
            b.Put([]byte("l"), genesisBlock.Hash)
        }
        return nil
    })
    if err != nil {
        log.Panic(err)
    }

    //返回区块链对象
    return &BlockChain{genesisBlock.Hash, db}
}

//step4：修改该方法
func (bc *BlockChain) AddBlockToBlockChain(data string) {
    //创建新区块
    //newBlock := NewBlock(data,prevHash,height)
    //添加到切片中
    //bc.Blocks = append(bc.Blocks,newBlock)
    //1.更新数据库
    err := bc.DB.Update(func(tx *bolt.Tx) error {
        //2.打开表
        b := tx.Bucket([]byte(BLOCKTABLENAME))
        if b != nil {
            //2.根据最新块的 hash 读取数据，并反序列化最后一个区块
            blockBytes := b.Get(bc.Tip)
            lastBlock := DeserializeBlock(blockBytes)
            //3.创建新的区块
            newBlock := NewBlock(data, lastBlock.Hash, lastBlock.Height+1)
            //4.将新的区块序列化并存储
            err := b.Put(newBlock.Hash, newBlock.Serilalize())
            if err != nil {
                log.Panic(err)
            }
            //5.更新最后一个哈希值，以及 blockchain 的 tip
            b.Put([]byte("l"), newBlock.Hash)
            bc.Tip = newBlock.Hash
        }

        return nil
    })
    if err != nil {
        log.Panic(err)
    }

}

//step3：
//判断数据库是否存在
func dbExists() bool {
    if _, err := os.Stat(DBNAME); os.IsNotExist(err) {
        return false
    }
    return true
}

//step5：新增方法，遍历数据库，打印输出所有的区块信息
/*
func (bc *BlockChain) PrintChains() {
    //1.根据 bc 的 tip，获取最新的 hash 值，表示当前的 hash
    var currentHash = bc.Tip
    //2.循环，根据当前 hash 读取数据，反序列化得到最后一个区块
    var count = 0
    block := new(Block) // var block *Block
    for {
        err := bc.DB.View(func(tx *bolt.Tx) error {
            b := tx.Bucket([]byte(BLOCKTABLENAME))

            if b != nil {
                count++
                fmt.Printf("第%d 个区块的信息：\n", count)
                //获取当前 hash 对应的数据，并进行反序列化
                blockBytes := b.Get(currentHash)
                block = DeserializeBlock(blockBytes)
                fmt.Printf("\t 高度：%d\n", block.Height)
                fmt.Printf("\t 上一个区块的 hash：%x\n", block.PrevBlockHash)
                fmt.Printf("\t 当前的 hash：%x\n", block.Hash)
                fmt.Printf("\t 数据：%s\n", block.Data)
                //fmt.Printf("\t 时间：%v\n", block.TimeStamp)
                fmt.Printf("\t 时间：%s\n",time.Unix(block.TimeStamp,0).Format("2006-01-02 15:04:05"))
                fmt.Printf("\t 次数：%d\n", block.Nonce)
            }

            return nil
        })
        if err != nil {
            log.Panic(err)
        }
        //3.直到父 hash 值为 0
        hashInt := new(big.Int)
        hashInt.SetBytes(block.PrevBlockHash)
        if big.NewInt(0).Cmp(hashInt) == 0 {
            break
        }
        //4.更新当前区块的 hash 值
        currentHash = block.PrevBlockHash
    }
}
*/

//2.获取一个迭代器的方法
func (bc *BlockChain) Iterator() *BlockChainIterator {
    return &BlockChainIterator{bc.Tip, bc.DB}
}

func (bc *BlockChain) PrintChains() {
    //1.获取迭代器对象
    bcIterator := bc.Iterator()

    var count = 0
    //2.循环迭代
    for {
        block := bcIterator.Next()
        count++
        fmt.Printf("第%d 个区块的信息：\n", count)
        //获取当前 hash 对应的数据，并进行反序列化
        fmt.Printf("\t 高度：%d\n", block.Height)
        fmt.Printf("\t 上一个区块的 hash：%x\n", block.PrevBlockHash)
        fmt.Printf("\t 当前的 hash：%x\n", block.Hash)
        fmt.Printf("\t 数据：%s\n", block.Data)
        //fmt.Printf("\t 时间：%v\n", block.TimeStamp)
        fmt.Printf("\t 时间：%s\n", time.Unix(block.TimeStamp, 0).Format("2006-01-02 15:04:05"))
        fmt.Printf("\t 次数：%d\n", block.Nonce)

        //3.直到父 hash 值为 0
        hashInt := new(big.Int)
        hashInt.SetBytes(block.PrevBlockHash)
        if big.NewInt(0).Cmp(hashInt) == 0 {
            break
        }
    }
} 
```

#### 3.2.5 修改`main.go`

在`main.go`中修改测试代码

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
    //blockChain:=BLC.CreateBlockChainWithGenesisBlock()
    //blockChain.AddBlockToBlockChain("Send 100RMB To Wangergou",blockChain.Blocks[len(blockChain.Blocks)-1].Height+1,blockChain.Blocks[len(blockChain.Blocks)-1].Hash)
    //blockChain.AddBlockToBlockChain("Send 300RMB To lixiaohua",blockChain.Blocks[len(blockChain.Blocks)-1].Height+1,blockChain.Blocks[len(blockChain.Blocks)-1].Hash)
    //blockChain.AddBlockToBlockChain("Send 500RMB To rose",blockChain.Blocks[len(blockChain.Blocks)-1].Height+1,blockChain.Blocks[len(blockChain.Blocks)-1].Hash)
    //
    //fmt.Println(blockChain)

    //5.测试序列化和反序列化
    //block:=BLC.NewBlock("helloworld",make([]byte,32,32),0)
    //data:=block.Serilalize()
    //fmt.Println(block)
    //fmt.Println(data)
    //block2:=BLC.DeserializeBlock(data)
    //fmt.Println(block2)

    //6.创建区块，存入数据库
    //打开数据库
    /*
    block:=BLC.NewBlock("helloworld",make([]byte,32,32),0)
    db,err := bolt.Open("my.db",0600,nil)
    if err != nil{
        log.Fatal(err)
    }

    defer db.Close()

    err = db.Update(func(tx *bolt.Tx) error {
        //获取 bucket，没有就创建新表
        b := tx.Bucket([]byte("blocks"))
        if b == nil{
            b,err = tx.CreateBucket([] byte("blocks"))
            if err !=nil{
                log.Panic("创建表失败")
            }
        }
        //添加数据
        err  = b.Put([]byte("l"),block.Serilalize())
        if err !=nil{
            log.Panic(err)
        }

        return nil
    })
    if err != nil{
        log.Panic(err)
    }
    err = db.View(func(tx *bolt.Tx) error {
        b := tx.Bucket([]byte("blocks"))
        if b !=nil{
            data := b.Get([]byte("l"))
            //fmt.Printf("%s\n",data)//直接打印会乱码
            //反序列化
            block2:=BLC.DeserializeBlock(data)
            //fmt.Println(block2)
            fmt.Printf("%v\n",block2)

        }
        return nil
    })
*/
    //7.测试创世区块存入数据库
    blockchain:=BLC.CreateBlockChainWithGenesisBlock("Genesis Block..")
    fmt.Println(blockchain)
    defer blockchain.DB.Close()
    //8.测试新添加的区块
    blockchain.AddBlockToBlockChain("Send 100RMB to wangergou")
    blockchain.AddBlockToBlockChain("Send 100RMB to lixiaohua")
    blockchain.AddBlockToBlockChain("Send 100RMB to rose")
    fmt.Println(blockchain)
    blockchain.PrintChains()

} 
```

## 4\. 持久化讲解

### 4.1 选择数据库

目前，我们的区块链实现里面并没有用到数据库，而是在每次运行程序时，简单地将区块链存储在内存中。那么一旦程序退出，所有的内容就都消失了。我们没有办法再次使用这条链，也没有办法与其他人共享，所以我们需要把它存储到磁盘上。

那么，我们要用哪个数据库呢？实际上，任何一个数据库都可以。在比特币原始论文 中，并没有提到要使用哪一个具体的数据库，它完全取决于开发者如何选择。 Bitcoin Core，最初由中本聪发布，现在是比特币的一个参考实现，它使用的是 LevelDB 而我们将要使用的是 BoltDB。

**BoltDB 的特点**

1.  非常简洁
2.  用 Go 实现
3.  不需要运行一个服务器
4.  能够允许我们构造想要的数据结构

BoltDB GitHub 上的 [README](https://github.com/boltdb/bolt) 是这么说的：

> Bolt 是一个纯键值存储的 Go 数据库，启发自 Howard Chu 的 LMDB. 它旨在为那些无须一个像 Postgres 和 MySQL 这样有着完整数据库服务器的项目，提供一个简单，快速和可靠的数据库。
> 
> 由于 Bolt 意在用于提供一些底层功能，简洁便成为其关键所在。它的 API 并不多，并且仅关注值的获取和设置。仅此而已。

听起来跟我们的需求完美契合！来快速过一下：

Bolt 使用键值存储，这意味着它没有像 SQL RDBMS （MySQL，PostgreSQL 等等）的表，没有行和列。相反，数据被存储为键值对（key-value pair，就像 Golang 的 map）。键值对被存储在 bucket 中，这是为了将相似的键值对进行分组（类似 RDBMS 中的表格）。因此，为了获取一个值，你需要知道一个 bucket 和一个键（key）。

需要注意的一个事情是，Bolt 数据库没有数据类型：键和值都是字节数组（byte array）。鉴于需要在里面存储 Go 的结构（准确来说，也就是存储**Block（块）**），我们需要对它们进行序列化，也就说，实现一个从 Go struct 转换到一个 byte array 的机制，同时还可以从一个 byte array 再转换回 Go struct。虽然我们将会使用 encoding/gob 来完成这一目标，但实际上也可以选择使用 **JSON**, **XML**, **Protocol Buffers** 等等。之所以选择使用 **encoding/gob**, 是因为它很简单，而且是 Go 标准库的一部分。

虽然 BoltDB 的作者出于个人原因已经不在对其维护, 不过关系不大，它已经足够稳定了，况且也有活跃的 fork:[coreos/bblot](https://github.com/coreos/bbolt)。

### 4.2 对比比特币

在开始实现持久化的逻辑之前，我们首先需要决定到底要如何在数据库中进行存储。为此，我们可以参考 Bitcoin Core 的做法：

简单来说，Bitcoin Core 使用两个 “bucket” 来存储数据：

1.  其中一个 bucket 是 **blocks**，它存储了描述一条链中所有块的元数据
2.  另一个 bucket 是 **chainstate**，存储了一条链的状态，也就是当前所有的未花费的交易输出，和一些元数据

此外，出于性能的考虑，Bitcoin Core 将每个区块（block）存储为磁盘上的不同文件。如此一来，就不需要仅仅为了读取一个单一的块而将所有（或者部分）的块都加载到内存中。但是，为了简单起见，我们并不会实现这一点。

在 **blocks** 中，**key -> value** 为：

| key | value |
| --- | --- |
| `b` + 32 字节的 block hash | block index record |
| `f` + 4 字节的 file number | file information record |
| `l` + 4 字节的 file number | the last block file number used |
| `R` + 1 字节的 boolean | 是否正在 reindex |
| `F` + 1 字节的 flag name length + flag name string | 1 byte boolean: various flags that can be on or off |
| `t` + 32 字节的 transaction hash | transaction index record |

在 **chainstate**，**key -> value** 为：

| key | value |
| --- | --- |
| `c` + 32 字节的 transaction hash | unspent transaction output record for that transaction |
| `B` | 32 字节的 block hash: the block hash up to which the database represents the unspent transaction outputs |

因为目前还没有交易，所以我们只需要 **blocks** bucket。另外，正如上面提到的，我们会将整个数据库存储为单个文件，而不是将区块存储在不同的文件中。所以，我们也不会需要文件编号（file number）相关的东西。最终，我们会用到的键值对有：

1.  32 字节的 block-hash -> block 结构
2.  `l` -> 链中最后一个块的 hash

这就是实现持久化机制所有需要了解的内容了。

### 4.3 BoltDB 的操作

#### 4.3.1 安装 BoltDB

打开终端输入命令：

```go
hanru:mypublicchain ruby$ go get "github.com/boltdb/bolt" 
```

如图：

![`img.kongyixueyuan.com/0403_%E5%AE%89%E8%A3%85%E6%95%B0%E6%8D%AE%E5%BA%93.png`](img/a180cc75ce5ef0397e88dcc82292ec24.jpg)

安装成功后，会在`gopath`下的 src 目录下，有`github.com`目录，里面会有`boltdb/bolt`目录，会有 boltdb 的 go 文件。(也可以直接进入 gopath 下进行查看)。

效果图：

![`img.kongyixueyuan.com/0404_boltdb%E5%AE%89%E8%A3%85%E6%95%88%E6%9E%9C.gif`](img/6a2a2fbc1700d038a3ed568f1c2f97c7.jpg)

#### 4.3.2 打开和关闭数据库

在`main.go`中，编写测试代码如下：

```go
package main

import (
    "github.com/boltdb/bolt"
    "log"
)
func main()  {
    /*
    1.安装数据库
        打开终端：go get "github.com/boltdb/bolt"
        此处需要稍微等待一下

    2.导入数据库的包
     */

    // Open the my.db data file in your current directory.
    // It will be created if it doesn't exist.
    db, err := bolt.Open("my.db", 0600, nil)
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()
} 
```

接下来我们仔细的来一段一段地看下代码：

```go
db, err := bolt.Open("my.db", 0600, nil) 
```

这是打开一个 BoltDB 文件的标准做法。注意，即使不存在这样的文件，它也不会返回错误。

以上代码用于打开一个数据库`my.db`，如果数据库存在就直接打开获得 db 对象，如果不存在，那么会创建数据库，文件权限是 0600。

执行程序:

```go
打开终端进入该项目目录：
//输入以下命令：将程序进行编译，编译成功后会产生可执行文件命名为 bc
hanru:day02_03_Persistence ruby$ go build -o bc main.go
//继续输入命令：执行程序，因为数据库文件不存在，则会创建 my.db 文件。
hanru:day02_03_Persistence ruby$ ./bc 
```

执行结果：

![`img.kongyixueyuan.com/0405_%E6%89%93%E5%BC%80%E6%95%B0%E6%8D%AE%E5%BA%93.gif`](img/e62cf6523b6fef44f09692768efe562f.jpg)

> 说明一下：
> 
> 上面示例代码，创建的数据库是相对路径。
> 
> A：此处我们先将`main.go`进行编译成一个可执行文件 bc。然后执行 bc 文件，那么创建出来的`my.db`数据库文件和 bc 在同一个目录下，`mypublicchain/day02_03_Persistence/my.db`。
> 
> B：如果直接通过 go run main.go 进行程序执行，那么数据库的路径名是相对于当前工程，那么`my.db`的目录是`mypublicchain/my.db`。

#### 4.3.2 数据库的操作

在 BoltDB 中，数据库操作通过一个事务（transaction）进行操作。有两种类型的事务：只读（read-only）和读写（read-write）。

有两个常用的事务函数：读写事务`Update()`和只读事务`View()`。

`Update()`操作，用于对数据库执行增加，删除，修改数据。所以该操作具有读写权限。

```go
//Read-write transactions
//To start a read-write transaction, you can use the DB.Update() function:

err := db.Update(func(tx *bolt.Tx) error {
    ...
    return nil
}) 
```

`View()`，用于从数据库中读取数据。所以该操作只有读取的权限。

```go
//Read-only transactions
//To start a read-only transaction, you can use the DB.View() function:

err := db.View(func(tx *bolt.Tx) error {
    ...
    return nil
}) 
```

**存储数据**

向数据库中存储数据，在`main.go`中添加代码如下：

```go
package main

import (
    "github.com/boltdb/bolt"
    "log"
    "fmt"
)
func main()  {
    /*
    1.安装数据库
        打开终端：go get "github.com/boltdb/bolt"
        此处需要稍微等待一下

    2.导入数据库的包
     */

    // Open the my.db data file in your current directory.
    // It will be created if it doesn't exist.
    db, err := bolt.Open("my.db", 0600, nil)
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()

    /*
    Update(),读写
    View(),只读
     */
    //1.创建表
    err = db.Update(func(tx *bolt.Tx) error{
        //1.创建 MyBucket
        b,err := tx.CreateBucket([]byte("MyBycket"))
        if err != nil{
            return fmt.Errorf("create bucket:%s",err)
        }
        //2.向表中存储数据
        if b != nil{
            err := b.Put([] byte("l"),[] byte("send 100 BTC to 王二狗"))
            if err != nil{
                log.Panic("数据存储失败。。")
            }
        }
        return nil
    })

    if err != nil{
        log.Panic(err)
    }
} 
```

这部分比较直观，我们需要调用`Update()`才能进行数据存储。首先，创建`bucket`，然后调用`Put()`方法存储键值对。(类似于`map`操作。)

执行程序，结果如下：

![`img.kongyixueyuan.com/0406_%E5%AD%98%E5%82%A8%E6%95%B0%E6%8D%AE.gif`](img/ecfc1b4db018f760a6f0e697c585ce30.jpg)

**读取数据**

从数据库中存储数据，在`main.go`中修改代码如下：

```go
package main

import (
    "github.com/boltdb/bolt"
    "log"
    "fmt"
)

func main() {
    /*
    1.安装数据库
        打开终端：go get "github.com/boltdb/bolt"
        此处需要稍微等待一下

    2.导入数据库的包
     */

    // Open the my.db data file in your current directory.
    // It will be created if it doesn't exist.
    db, err := bolt.Open("my.db", 0600, nil)
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()

    /*
    Update(),读写
    View(),只读
     */
    //读取数据
    err = db.View(func(tx *bolt.Tx) error {
        //获取 bucket 对象
        b := tx.Bucket([]byte("MyBycket"))
        if b != nil {
            //根据 key 查看数据
            data := b.Get([] byte("l"))//根据 key 获取对应的 value 值
            fmt.Println(data)
            fmt.Printf("%s\n", data)
            data2 := b.Get([] byte("ll"))//key 不存在
            fmt.Println(data2)
            fmt.Printf("%s\n", data2) //[]，如果对应的 key 不存在，那么取出的是空。

        }

        return nil
    })

    if err != nil {
        log.Panic(err)
    }
} 
```

如果数据库已经存在，并且存储了数据，我们先获取到`bucket`对象，然后通过`Get()`方法，根据`key`来读取数据。如果`key`不存在，那么会读到空。

执行程序，并打印结果如下:

![`img.kongyixueyuan.com/0407_%E8%AF%BB%E5%8F%96%E6%95%B0%E6%8D%AE.gif`](img/fab3fb3fb46d5e8d87692fe99cdc11eb.jpg)

**操作 Cursor**

使用`Cursor`查询数据库中的所有数据。上述操作是通过`key`获取对应的`value`，我们还可以通过`Cursor`获取`bucket`中的所有的`key-value`。

修改`main.go`中的代码如下：

```go
package main

import (
    "github.com/boltdb/bolt"
    "log"
    "fmt"
)

func main() {
    db, err := bolt.Open("my.db", 0600, nil)
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()

    db.View(func(tx *bolt.Tx) error {
        b :=tx.Bucket([]byte("MyBycket"))
        c :=b.Cursor()
        for k,v:=c.First();k!=nil;k,v=c.Next(){
            fmt.Printf("key=%s,value=%s\n",k,v)
        }
        return nil
    })
} 
```

通过`Cursor`对象，可以获取`bucket`中的所有数据，直到获取完毕。

运行结果如下：

![`img.kongyixueyuan.com/0408_cursor.gif`](img/ff0154337b5b131ae787ed9b9e7b2525.jpg)

### 4.4 序列化

#### 4.4.1 序列化和反序列化

所谓**序列化**就是将对象状态转换为可保持或传输的格式(比如[]byte，或者二进制数据等)的过程。与*序列化*相对的是**反序列化**,再将这些数据转换为对象。这两个过程结合起来,可以轻松地存储和传输数据。

我们若要将区块链持久化存储到数据库中，其实就是将每个区块对象存入到数据库中。

上面提到，在 BoltDB 中，值只能是 `[]byte` 类型，但是我们想要存储 `Block` 结构。所以，我们需要使用 [encoding/gob](https://golang.org/pkg/encoding/gob/) 来对这些结构进行序列化。

让我们来实现 `Block.go`文件中添加 的 `Serialize` 方法）：

```go
//将区块序列化，得到一个字节数组---区块的行为，设计为方法
func (block *Block) Serilalize() []byte {
    //1.创建一个 buffer
    var result bytes.Buffer
    //2.创建一个编码器
    encoder := gob.NewEncoder(&result)
    //3.编码--->打包
    err := encoder.Encode(block)
    if err != nil {
        log.Panic(err)
    }
    return result.Bytes()
} 
```

这个部分比较直观：首先，我们定义一个 buffer 存储序列化之后的数据。然后，我们初始化一个 gob `encoder` 并对 block 进行编码，结果作为一个字节数组返回。

接下来，我们需要一个解序列化的函数，它会接受一个字节数组作为输入，并返回一个 `Block`. 它不是一个方法（method），而是一个单独的函数（function）：

```go
 //反序列化，得到一个区块---设计为函数
func DeserializeBlock(blockBytes [] byte) *Block {
    var block Block
    var reader = bytes.NewReader(blockBytes)
    //1.创建一个解码器
    decoder := gob.NewDecoder(reader)
    //解包
    err := decoder.Decode(&block)
    if err != nil {
        log.Panic(err)
    }
    return &block
} 
```

这就是序列化部分的内容了。

#### 4.4.2 代码测试

接下来，我们可以在`main.go`中进行测试，我们创建一个区块，并且进行序列化后存入到数据库中，然后再读取出来。

修改`main.go`代码如下：

```go
package main

import (
    "./BLC"
    "github.com/boltdb/bolt"
    "log"
    "fmt"
)

func main() {
    //6.创建区块，存入数据库
    //打开数据库
    block:=BLC.NewBlock("helloworld",make([]byte,32,32),0)
    db,err := bolt.Open("my.db",0600,nil)
    if err != nil{
        log.Fatal(err)
    }

    defer db.Close()

    //存储一个 block 区块
    err = db.Update(func(tx *bolt.Tx) error {
        //获取 bucket，没有就创建新表
        b := tx.Bucket([]byte("blocks"))
        if b == nil{
            b,err = tx.CreateBucket([] byte("blocks"))
            if err !=nil{
                log.Panic("创建表失败")
            }
        }
        //添加数据
        err  = b.Put([]byte("l"),block.Serilalize())
        if err !=nil{
            log.Panic(err)
        }

        return nil
    })
    if err != nil{
        log.Panic(err)
    }
    //从数据库中读取该区块数据
    err = db.View(func(tx *bolt.Tx) error {
        b := tx.Bucket([]byte("blocks"))
        if b !=nil{
            data := b.Get([]byte("l"))
            //fmt.Printf("%s\n",data)//直接打印会乱码
            //反序列化
            block2:=BLC.DeserializeBlock(data)
            //fmt.Println(block2)
            fmt.Printf("%v\n",block2)

        }
        return nil
    })
} 
```

以上内容就是创建一个`block`，序列化后进行持久化存储。然后再从数据库中读取出并打印输出。

程序执行结果：

![`img.kongyixueyuan.com/0409_%E5%BA%8F%E5%88%97%E5%8C%96.gif`](img/339435bd7577c21ca887f02fe756b16c.jpg)

### 4.5 持久化

经过上面的讲解，我们已经会使用 boltDB 数据库进行存储和读取。也学会了将 Block 对象进行序列化和反序列化。接下来，让我们在项目中进行代码修改，实现挖矿后将区块进行持久化存储。

**对于数据库中，我们设计的结构是，每个 block 的 hash 值作为 key，而 block 序列化后的数据作为 value。还需要单独存储一个"l"作为 key(也可以选择其他的字符串作为 key，比如："lasthash"等)，用于存储最后一个区块的 hash 值。这样，我们就可以根据 l 获取最后一个区块 hash，根据该 hash 可以获取到最后一个区块。然后再获取前一个区块，以此类推，直到创世区块。我们就可以获取所有的区块数据了。**

#### 4.5.1 更改 BlockChain 结构体

程序中的`BlockChain`，是用于存储`Block`的，修改前我们使用了`[]*Block`数组来存储 Block 区块。如果要实现持久化操作，就不能使用数组，所以修改 BlockChain 的字段为：`*bolt.DB`用于操作数据库，以及一个`Tip []byte`,用于存储数据库中最后一个区块的`hash`值。（tip 有尾部，尖端的意思，在这里 tip 存储的是最后一个块的哈希）

在`BlockChain.go`中修改`BlockChain`结构体如下：

```go
//step1：修改 BlockChain 的结构体
type BlockChain struct {
    //Blocks []*Block //存储有序的区块
    Tip [] byte  // 最近的取快递 Hash 值
    DB  *bolt.DB //数据库对象
} 
```

> 这次，我们不在里面存储所有的区块了，而是仅存储区块链的 `tip`。另外，我们存储了一个数据库链接对象。因为我们想要一旦打开它的话，就让它一直运行，直到程序运行结束。

#### 4.5.2 添加`Constant.go`

添加`Constant.go`，用于存储一些常量数值，比如数据库的名字，bucket 的名字等。

在`Constant.go`中添加两个常量：

```go
package BLC

const DBNAME = "blockchain.db"  //数据库名
const BLOCKTABLENAME = "blocks" //表名 
```

#### 4.5.3 添加`dbExists()`函数

`dbExists()`函数，用于判断给定的数据库是否存在，代码如下:

```go
//判断数据库是否存在
func dbExists() bool {
    if _, err := os.Stat(DBNAME); os.IsNotExist(err) {
        return false
    }
    return true
} 
```

#### 4.5.4 更改 CreateBlockChainWithGenesisBlock()函数

`CreateBlockChainWithGenesisBlock()`函数，用于获得一个`BlockChain`对象，修改前，会创建一个新的 `Blockchain` 实例，并向其中加入创世块。。而现在，我们希望它做的事情有:

1.  判断数据库是否存在。如果数据库存在：
    1.  创建 BlockChain 实例。
    2.  读取数据库中最后一个区块的 hash，并设置给 BlockChain 实例的 Tip 字段。
2.  如果数据库不存在：
    1.  首先我们需要先创建一个创世区块
    2.  打开数据库，并且创建 bucket
    3.  将创世区块序列化后存入到数据库中
    4.  将创世区块的 hash 保存为最后一个块的 hash
    5.  创建 BlockChain 实例，设置 Tip 为创世区块的 hash，并返回该 blockchain 实例。

在`BlockChain.go`中，修改`CreateBlockChainWithGenesisBlock()`函数代码如下:

```go
 //step2：修改该方法
func CreateBlockChainWithGenesisBlock(data string) *BlockChain {
    //1.先判断数据库是否存在，如果有，从数据库读取
    if dbExists() {
        fmt.Println("数据库已经存在。。")
        //A：打开数据库
        db, err := bolt.Open(DBNAME, 0600, nil)
        if err != nil {
            log.Fatal(err)
        }
        //defer db.Close()
        var blockchain *BlockChain
        //B：读取数据库
        err = db.View(func(tx *bolt.Tx) error {
            //C：打开表
            b := tx.Bucket([]byte(BLOCKTABLENAME))
            if b != nil {
                //D：读取最后一个 hash
                hash := b.Get([]byte("l"))
                //E：创建 blockchain
                blockchain = &BlockChain{hash, db}
            }
            return nil
        })
        if err != nil {
            log.Fatal(err)
        }
        return blockchain
    }

    //2.数据库不存在，说明第一次创建，然后存入到数据库中
    fmt.Println("数据库不存在。。")
    //A：创建创世区块
    //创建创世区块
    genesisBlock := CreateGenesisBlock(data)
    //B：打开数据库
    db, err := bolt.Open(DBNAME, 0600, nil)
    if err != nil {
        log.Fatal(err)
    }
    //defer db.Close()
    //C：存入数据表
    err = db.Update(func(tx *bolt.Tx) error {
        b, err := tx.CreateBucket([]byte(BLOCKTABLENAME))
        if err != nil {
            log.Panic(err)
        }
        if b != nil {
            err = b.Put(genesisBlock.Hash, genesisBlock.Serilalize())
            if err != nil {
                log.Panic("创世区块存储有误。。。")
            }
            //存储最新区块的 hash
            b.Put([]byte("l"), genesisBlock.Hash)
        }
        return nil
    })
    if err != nil {
        log.Panic(err)
    }

    //D:返回区块链对象
    return &BlockChain{genesisBlock.Hash, db}
} 
```

接下来我们仔细的来一段一段地看下代码：

如果数据库存在，我们打开的是只读事务(db.View(...))，在这里，我们先获取了存储区块的 bucket，然后从中读取 `l` 键；另外，注意创建 `Blockchain` 一个新的方式：

```go
//D：读取最后一个 hash
hash := b.Get([]byte("l"))
//E：创建 blockchain
blockchain = &BlockChain{hash, db} 
```

如果数据库不存在，就生成创世块，创建 bucket，并将区块保存到里面，然后更新 `l` 键以存储链中最后一个块的哈希。这里，打开的是一个读写事务（`db.Update(...)`），因为我们会向数据库中添加创世块。

首先创建创世区块：

```go
//创建创世区块
    genesisBlock := CreateGenesisBlock(data) 
```

将创世区块序列化后存入到数据库中：

```go
if b != nil {
    err = b.Put(genesisBlock.Hash, genesisBlock.Serilalize())
    if err != nil {
        log.Panic("创世区块存储有误。。。")
    }
    //存储最新区块的 hash
    b.Put([]byte("l"), genesisBlock.Hash)
} 
```

最后返回 BlockChain 实例对象：

```go
//D:返回区块链对象
return &BlockChain{genesisBlock.Hash, db} 
```

#### 4.5.5 更改`AddBlockToBlockChain()`

接下来我们想要更新的是 `AddBlockToBlockChain()` 方法：现在向链中加入区块，就不是像之前向一个数组中加入一个元素那么简单了。从现在开始，我们会将区块存储在数据库里面：

```go
 //step4：修改该方法
func (bc *BlockChain) AddBlockToBlockChain(data string) {
    //创建新区块
    //newBlock := NewBlock(data,prevHash,height)
    //添加到切片中
    //bc.Blocks = append(bc.Blocks,newBlock)
    //1.更新数据库
    err := bc.DB.Update(func(tx *bolt.Tx) error {
        //2.打开表
        b := tx.Bucket([]byte(BLOCKTABLENAME))
        if b != nil {
            //2.根据最新块的 hash 读取数据，并反序列化最后一个区块
            blockBytes := b.Get(bc.Tip)
            lastBlock := DeserializeBlock(blockBytes)
            //3.创建新的区块
            newBlock := NewBlock(data, lastBlock.Hash, lastBlock.Height+1)
            //4.将新的区块序列化并存储
            err := b.Put(newBlock.Hash, newBlock.Serilalize())
            if err != nil {
                log.Panic(err)
            }
            //5.更新最后一个哈希值，以及 blockchain 的 tip
            b.Put([]byte("l"), newBlock.Hash)
            bc.Tip = newBlock.Hash
        }

        return nil
    })
    if err != nil {
        log.Panic(err)
    }

} 
```

继续来一段一段分解开来：

```go
//2.根据最新块的 hash 读取数据，并反序列化最后一个区块
blockBytes := b.Get(bc.Tip)
lastBlock := DeserializeBlock(blockBytes) 
```

在这里，我们会从数据库中获取最后一个块的哈希，然后用它来挖出一个新的块的哈希，并序列化后存入数据库：

```go
//3.创建新的区块
newBlock := NewBlock(data, lastBlock.Hash, lastBlock.Height+1)
//4.将新的区块序列化并存储
err := b.Put(newBlock.Hash, newBlock.Serilalize()) 
```

最后，更新数据库中 l 键的 value 值为新区块的 hash，以及 BlockChain 的 Tip。

```go
//5.更新最后一个哈希值，以及 blockchain 的 tip
b.Put([]byte("l"), newBlock.Hash)
bc.Tip = newBlock.Hash 
```

### 4.6 检查遍历区块链

现在，产生的所有块都会被保存到一个数据库里面，所以我们可以重新打开一个链，然后向里面加入新块。但是在实现这一点后，我们失去了之前一个非常好的特性：再也无法打印区块链的区块了，因为现在不是将区块存储在一个数组，而是放到了数据库里面。让我们来解决这个问题！

#### 4.6.1 定义一个`BlockChainIterator`结构体

`BoltDB` 允许对一个 `bucket` 里面的所有 key 进行迭代，但是所有的 `key` 都以字节序进行存储，而且我们想要以区块能够进入区块链中的顺序进行打印。此外，因为我们不想将所有的块都加载到内存中（因为我们的区块链数据库可能很大！或者现在可以假装它可能很大），我们将会一个一个地读取它们。故而，我们需要一个区块链迭代器（`BlockchainIterator`）：

创建`BlockChainIterator.go`文件，并在里面定义结构体：

```go
//1.新增一个结构体
type BlockChainIterator struct {
    CurrentHash [] byte //当前区块的 hash
    DB          *bolt.DB //数据库
} 
```

#### 4.6.2 获取`Iterator`实例

每当要对链中的块进行迭代时，我们就会创建一个迭代器，里面存储了当前迭代的块哈希（`currentHash`）和数据库的连接（`db`）。通过 `db`，迭代器逻辑上被附属到一个区块链上（这里的区块链指的是存储了一个数据库连接的 `Blockchain` 实例），并且通过 `Blockchain` 方法进行创建：

在`BlockChain.go`文件中，添加一个方法用于获取`BlockChainIterator`实例，代码如下:

```go
//2.获取一个迭代器的方法
func (bc *BlockChain) Iterator() *BlockChainIterator {
    return &BlockChainIterator{bc.Tip, bc.DB}
} 
```

注意，迭代器的初始状态为链中的 tip，因此区块将从尾到头（创世块为头），也就是从最新的到最旧的进行获取。实际上，**选择一个 tip 就是意味着给一条链“投票”**。一条链可能有多个分支，最长的那条链会被认为是主分支。在获得一个 tip （可以是链中的任意一个块）之后，我们就可以重新构造整条链，找到它的长度和需要构建它的工作。这同样也意味着，一个 tip 也就是区块链的一种标识符。

#### 4.6.3 添加`Next()`方法

`BlockchainIterator` 只会做一件事情：返回链中的下一个块。我们根据迭代器的原理，添加一个`Next()`方法，用于获取一个区块。

在`BlockChainIterator.go`文件中，添加`Next()`方法，代码如下:

```go
 //3.获取区块
func (bcIterator *BlockChainIterator) Next() *Block {
    block:=new(Block)
    //1.打开数据库并读取
    err :=bcIterator.DB.View(func(tx *bolt.Tx) error {
        //2.打开数据表
        b:=tx.Bucket([]byte(BLOCKTABLENAME))
        if b != nil{
            //3.根据当前 hash 获取数据并反序列化
            blockBytes:=b.Get(bcIterator.CurrentHash)
            block = DeserializeBlock(blockBytes)
            //4.更新当前的 hash
            bcIterator.CurrentHash = block.PrevBlockHash
        }

        return nil
    })
    if err != nil{
        log.Panic(err)
    }
    return block
} 
```

#### 4.6.4 遍历数据库中的 Block

接下来我们就可以在`BlockChain`中添加打印数据库中区块的方法了。我们需要先获取迭代器实例，然后调用`Next()`方法获取区块。并判断该区块是否是创世块，如果是创世块，表示已经获取完了所有的区块，停止迭代。

在`BlockChain.go`文件中，添加`PrintChains()`，代码如下：

```go
 func (bc *BlockChain) PrintChains() {
    //1.获取迭代器对象
    bcIterator := bc.Iterator()

    var count = 0
    //2.循环迭代
    for {
        block := bcIterator.Next()
        count++
        fmt.Printf("第%d 个区块的信息：\n", count)
        //获取当前 hash 对应的数据，并进行反序列化
        fmt.Printf("\t 高度：%d\n", block.Height)
        fmt.Printf("\t 上一个区块的 hash：%x\n", block.PrevBlockHash)
        fmt.Printf("\t 当前的 hash：%x\n", block.Hash)
        fmt.Printf("\t 数据：%s\n", block.Data)
        //fmt.Printf("\t 时间：%v\n", block.TimeStamp)
        fmt.Printf("\t 时间：%s\n", time.Unix(block.TimeStamp, 0).Format("2006-01-02 15:04:05"))
        fmt.Printf("\t 次数：%d\n", block.Nonce)

        //3.直到父 hash 值为 0
        hashInt := new(big.Int)
        hashInt.SetBytes(block.PrevBlockHash)
        if big.NewInt(0).Cmp(hashInt) == 0 {
            break
        }
    }
} 
```

#### 4.6.5 `main.go`中测试

接下来我们在`main`中进行测试，首先创建`BlockChain`实例，并添加几个`Block`，然后遍历打印。

修改`main.go`代码的内容如下:

```go
package main

import (
    "./BLC"
    "fmt"
)

func main() {

    //7.测试创世区块存入数据库
    blockchain:=BLC.CreateBlockChainWithGenesisBlock("Genesis Block..")
    fmt.Println(blockchain)
    defer blockchain.DB.Close()
    //8.测试新添加的区块
    blockchain.AddBlockToBlockChain("Send 100RMB to wangergou")
    blockchain.AddBlockToBlockChain("Send 100RMB to lixiaohua")
    blockchain.AddBlockToBlockChain("Send 100RMB to rose")
    fmt.Println(blockchain)
    blockchain.PrintChains()

} 
```

运行结果:

![`img.kongyixueyuan.com/0410_%E8%BF%90%E8%A1%8C%E7%BB%93%E6%9E%9C.png`](img/04f4143ffa52ae5c1245235a49fc295b.jpg)

这就是数据库部分的内容了！

## 5\. 总结

通过本章节的学习，我们了解持久化的原理，并采用`BoltDB`进行区块的持久化存储。通过 BoltDB 的两种事务方法`db.Update(...)`和`db.View(...)`可以进行存储和读取数据。获取`BlockChain`实例也是通过数据库操作，首先判断数据库是否存在，如果存在直接从数据库中读取最后一个区块的`hash`值创建`BlockChain`实例。否则创建创世区块并序列化后进行存储，根据创世区块的`hash`值创建`BlockChain`实例。添加新区块也改为了将新的区块序列化后存入到数据库中。此外，我们还提供了迭代器获取每个区块，并进行区块的打印。

[项目源代码](https://github.com/rubyhan1314/PublicChain)