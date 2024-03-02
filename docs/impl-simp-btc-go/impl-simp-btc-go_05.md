# 第五章 CLI (Command Line Interface) 命令行界面

# CLI(Command Line Interface)

到目前为止，我们的实现还没有提供一个与程序交互的接口：目前只是在 `main` 函数中简单执行了 `CreateBlockChainWithGenesisBlock()` 和 `AddBlockToBlockChain()` 。是时候改变了！

## 1\. 课程目标

1.  了解什么是 CLI

2.  学会使用 flag 包的语法

3.  学会在项目中添加 cli 命令

## 2\. 项目代码及效果展示

### 2.1 项目代码结构

![`img.kongyixueyuan.com/0501_%E9%A1%B9%E7%9B%AE%E7%BB%93%E6%9E%84.png`](img/0dae932891ec692b9635eee1aee3819f.jpg)

### 2.2 项目运行结果

![`img.kongyixueyuan.com/0502_%E8%BF%90%E8%A1%8C%E7%BB%93%E6%9E%9C.gif`](img/34a2c03976679cfe3e8b0d8334640311.jpg)

## 3\. 创建项目

### 3.1 创建工程

首先打开 Goland 开发工具

打开工程：`mypublicchain`

创建项目：将上一次的项目代码，`day02_03_Persistence`，复制为`day02_04_cli`

> 说明：我们每一章节的项目代码，都是在上一个章节上进行添加。所以拷贝上一次的项目代码，然后进行新内容的添加或修改。

### 3.2 代码实现

#### 3.2.1 创建 go 文件：`CLI.go`

打开`day02_04_cli`目录里的 BLC 包,创建`CLI.go`文件。在`CLI.go`文件中编写代码如下：

```go
package BLC

import (
    "os"
    "fmt"
    "flag"
    "log"
)

//step1:
//CLI 结构体
type CLI struct {
    //Blockchain *BlockChain
}

//step2：添加 Run 方法
func (cli *CLI) Run(){
    //判断命令行参数的长度
    isValidArgs()

    //1.创建 flagset 标签对象
    addBlockCmd := flag.NewFlagSet("addblock",flag.ExitOnError)
    //fmt.Printf("%T\n",addBlockCmd) //*FlagSet
    printChainCmd:=flag.NewFlagSet("printchain",flag.ExitOnError)
    createBlockChainCmd:=flag.NewFlagSet("createblockchain",flag.ExitOnError)

    //2.设置标签后的参数
    flagAddBlockData:= addBlockCmd.String("data","helloworld..","交易数据")
    flagCreateBlockChainData := createBlockChainCmd.String("data","Genesis block data..","创世区块交易数据")

    //3.解析
    switch os.Args[1] {
    case "addblock":
        err:=addBlockCmd.Parse(os.Args[2:])
        if err != nil{
            log.Panic(err)
        }
        //fmt.Println("----",os.Args[2:])

    case "printchain":
        err :=printChainCmd.Parse(os.Args[2:])
        if err != nil{
            log.Panic(err)
        }
        //fmt.Println("====",os.Args[2:])

    case "createblockchain":
        err :=createBlockChainCmd.Parse(os.Args[2:])
        if err != nil{
            log.Panic(err)
        }

    default:
        printUsage()
        os.Exit(1)//退出
    }

    if addBlockCmd.Parsed(){
        if *flagAddBlockData == ""{
            printUsage()
            os.Exit(1)
        }
        cli.addBlock(*flagAddBlockData)
    }
    if printChainCmd.Parsed(){
        cli.printChains()
    }

    if createBlockChainCmd.Parsed(){
        if *flagCreateBlockChainData == ""{
            printUsage()
            os.Exit(1)
        }
        cli.createGenesisBlockchain(*flagCreateBlockChainData)
    }

}

func isValidArgs(){
    if len(os.Args) < 2{
        printUsage()
        os.Exit(1)
    }
}
func printUsage(){
    fmt.Println("Usage:")
    fmt.Println("\tcreateblockchain -data DATA -- 创建创世区块")
    fmt.Println("\taddblock -data Data -- 交易数据")
    fmt.Println("\tprintchain -- 输出信息")
}

func (cli *CLI)addBlock(data string){
    bc:=GetBlockchainObject()
    if bc == nil{
        fmt.Println("没有创世区块，无法添加。。")
        os.Exit(1)
    }
    defer bc.DB.Close()
    bc.AddBlockToBlockChain(data)
}

func (cli *CLI)printChains(){
    bc:=GetBlockchainObject()
    if bc == nil{
        fmt.Println("没有区块可以打印。。")
        os.Exit(1)
    }
    defer bc.DB.Close()
    bc.PrintChains()
}
func (cli *CLI) createGenesisBlockchain(data string){
    //fmt.Println(data)
    CreateBlockChainWithGenesisBlock(data)

} 
```

#### 3.2.2 修改`BlockChain.go`文件

打开`day02_04_cli`目录里的 BLC 包。修改`BlockChain.go`文件。

```go
修改步骤：
step1：修改 CreateBlockChainWithGenesisBlock()
step2：新增方法 GetBlockchainObject()，用于获取 BlockChain 实例对象 
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

type BlockChain struct {
    //Blocks []*Block //存储有序的区块
    Tip [] byte  // 最近的取快递 Hash 值
    DB  *bolt.DB //数据库对象
}

//修改该方法
/*
1.仅仅用来创建区块链
如果数据库存在，证明区块链存在，直接结束该方法
否则进行创建创世区块，并存入数据库中
 */
func CreateBlockChainWithGenesisBlock(data string) {
    if dbExists() {
        fmt.Println("数据库已经存在。。。")
        return
    }

    //
    fmt.Println("创建创世区块：", data)
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
    defer db.Close()
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
    //return &BlockChain{genesisBlock.Hash, db}
}

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

//判断数据库是否存在
func dbExists() bool {
    if _, err := os.Stat(DBNAME); os.IsNotExist(err) {
        return false
    }
    return true
}

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

    //2.循环迭代
    for {
        block := bcIterator.Next()
        fmt.Printf("第%d 个区块的信息：\n", block.Height+1)
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

//新增方法，用于获取区块链
func GetBlockchainObject() *BlockChain {
    /*
    1.如果数据库不存在，直接返回 nil
    2.读取数据库
     */
    if !dbExists() {
        fmt.Println("数据库不存在，无法获取区块链。。")
        return nil
    }

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
```

#### 3.2.3 修改`main.go`

在`main.go`中修改测试代码

```go
package main

import (
    "./BLC"

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
    //block:=BLC.NewBlock("helloworld",make([]byte,32,32),0)
    //db,err := bolt.Open("my.db",0600,nil)
    //if err != nil{
    //    log.Fatal(err)
    //}
    //
    //defer db.Close()
    //
    //err = db.Update(func(tx *bolt.Tx) error {
    //    //获取 bucket，没有就创建新表
    //    b := tx.Bucket([]byte("blocks"))
    //    if b == nil{
    //        b,err = tx.CreateBucket([] byte("blocks"))
    //        if err !=nil{
    //            log.Panic("创建表失败")
    //        }
    //    }
    //    //添加数据
    //    err  = b.Put([]byte("l"),block.Serilalize())
    //    if err !=nil{
    //        log.Panic(err)
    //    }
    //
    //    return nil
    //})
    //if err != nil{
    //    log.Panic(err)
    //}
    //err = db.View(func(tx *bolt.Tx) error {
    //    b := tx.Bucket([]byte("blocks"))
    //    if b !=nil{
    //        data := b.Get([]byte("l"))
    //        //fmt.Printf("%s\n",data)//直接打印会乱码
    //        //反序列化
    //        block2:=BLC.DeserializeBlock(data)
    //        //fmt.Println(block2)
    //        fmt.Printf("%v\n",block2)
    //
    //    }
    //    return nil
    //})

    //7.测试创世区块存入数据库
    //blockchain:=BLC.CreateBlockChainWithGenesisBlock("Genesis Block..")
    //fmt.Println(blockchain)
    //defer blockchain.DB.Close()

    //8.测试新添加的区块
    //blockchain.AddBlockToBlockChain("Send 100RMB to wangergou")
    //blockchain.AddBlockToBlockChain("Send 100RMB to lixiaohua")
    //blockchain.AddBlockToBlockChain("Send 100RMB to rose")
    //fmt.Println(blockchain)
    //blockchain.PrintChains()

    //9.CLI 操作
    cli:=BLC.CLI{}
    cli.Run()

} 
```

## 4\. CLI 讲解

### 4.1 什么是 CLI

目前只是在 `main` 函数中简单执行了 `NewBlockchain` 和 `bc.AddBlock` 。是时候改变了！现在我们想要拥有这些命令,就需要通过 cli 实现。

```go
./bc createblockchain -data "genesisblock"
./bc addblock -data "send 1ETH to wangergou" 
```

**命令行界面**（英语：**command-line interface**，缩写：**CLI**）是在图形用户界面得到普及之前使用最为广泛的用户界面，它通常不支持鼠标，用户通过键盘输入指令，计算机接收到指令后，予以执行。

### 4.2 使用 CLI

所有命令行相关的操作都会通过 `CLI` 结构体进行处理：

```go
type CLI struct {
    //bc *Blockchain
} 
```

它的 “入口” 是 `Run` 函数：

```go
//step2：添加 Run 方法
func (cli *CLI) Run(){
    //判断命令行参数的长度
    isValidArgs()

    //1.创建 flagset 标签对象
    addBlockCmd := flag.NewFlagSet("addblock",flag.ExitOnError)
    //fmt.Printf("%T\n",addBlockCmd) //*FlagSet
    printChainCmd:=flag.NewFlagSet("printchain",flag.ExitOnError)
    createBlockChainCmd:=flag.NewFlagSet("createblockchain",flag.ExitOnError)

    //2.设置标签后的参数
    flagAddBlockData:= addBlockCmd.String("data","helloworld..","交易数据")
    flagCreateBlockChainData := createBlockChainCmd.String("data","Genesis block data..","创世区块交易数据")

    //3.解析
    switch os.Args[1] {
    case "addblock":
        err:=addBlockCmd.Parse(os.Args[2:])
        if err != nil{
            log.Panic(err)
        }
        //fmt.Println("----",os.Args[2:])

    case "printchain":
        err :=printChainCmd.Parse(os.Args[2:])
        if err != nil{
            log.Panic(err)
        }
        //fmt.Println("====",os.Args[2:])

    case "createblockchain":
        err :=createBlockChainCmd.Parse(os.Args[2:])
        if err != nil{
            log.Panic(err)
        }

    default:
        printUsage()
        os.Exit(1)//退出
    }

    if addBlockCmd.Parsed(){
        if *flagAddBlockData == ""{
            printUsage()
            os.Exit(1)
        }
        cli.addBlock(*flagAddBlockData)
    }
    if printChainCmd.Parsed(){
        cli.printChains()
    }

    if createBlockChainCmd.Parsed(){
        if *flagCreateBlockChainData == ""{
            printUsage()
            os.Exit(1)
        }
        cli.createGenesisBlockchain(*flagCreateBlockChainData)
    }

} 
```

我们会使用标准库里面的 [flag](https://golang.org/pkg/flag/) 包来解析命令行参数：

```go
createBlockChainCmd:=flag.NewFlagSet("createblockchain",flag.ExitOnError)
addBlockCmd := flag.NewFlagSet("addblock",flag.ExitOnError)
printChainCmd:=flag.NewFlagSet("printchain",flag.ExitOnError) 
```

首先，我们创建三个子命令:`createblockchain`， `addblock` 和 `printchain`, 然后给`createblockchain` ，`addblock` 添加 `-data` 参数。`printchain` 没有任何标志。

```go
//2.设置标签后的参数
flagCreateBlockChainData := createBlockChainCmd.String("data","Genesis block data..","创世区块交易数据")
flagAddBlockData:= addBlockCmd.String("data","helloworld..","交易数据") 
```

我们希望当程序运行时，命令行提示信息如下:

![`img.kongyixueyuan.com/0503_flag.png`](img/f59f004fb2096ed1e020f2fab59bc686.jpg)

然后，我们通过一个分支语句，检查用户提供的命令，解析相关的 `flag` 子命令：

```go
//3.解析
    switch os.Args[1] {
    case "addblock":
        err:=addBlockCmd.Parse(os.Args[2:])
        if err != nil{
            log.Panic(err)
        }
        //fmt.Println("----",os.Args[2:])

    case "printchain":
        err :=printChainCmd.Parse(os.Args[2:])
        if err != nil{
            log.Panic(err)
        }
        //fmt.Println("====",os.Args[2:])

    case "createblockchain":
        err :=createBlockChainCmd.Parse(os.Args[2:])
        if err != nil{
            log.Panic(err)
        }

    default:
        printUsage()
        os.Exit(1)//退出
    } 
```

接下来，根据被解析的命令，执行相应的操作。

```go
if addBlockCmd.Parsed(){
        if *flagAddBlockData == ""{
            printUsage()
            os.Exit(1)
        }
        cli.addBlock(*flagAddBlockData)
    }
if printChainCmd.Parsed(){
        cli.printChains()
}

if createBlockChainCmd.Parsed(){
        if *flagCreateBlockChainData == ""{
            printUsage()
            os.Exit(1)
        }
        cli.createGenesisBlockchain(*flagCreateBlockChainData)
} 
```

接着检查解析是哪一个子命令，并调用相关函数：

```go
 func (cli *CLI)addBlock(data string){
    bc:=GetBlockchainObject()
    if bc == nil{
        fmt.Println("没有创世区块，无法添加。。")
        os.Exit(1)
    }
    defer bc.DB.Close()
    bc.AddBlockToBlockChain(data)
}

func (cli *CLI)printChains(){
    bc:=GetBlockchainObject()
    if bc == nil{
        fmt.Println("没有区块可以打印。。")
        os.Exit(1)
    }
    defer bc.DB.Close()
    bc.PrintChains()
}
func (cli *CLI) createGenesisBlockchain(data string){
    //fmt.Println(data)
    CreateBlockChainWithGenesisBlock(data)

} 
```

修改`BlockChain.go`中的`CreateBlockChainWithGenesisBlock()`方法，修改前该方法用来创建创世区块并返回`BlockChain`实例对象。现在我们只希望这个方法用于创建创世区块，并不需要返回`BlockChain`实例对象。稍后我们专门提供一个方法返回`BlockChain`实例对象。

修改后的代码如下：

```go
//仅仅用来创建区块链
//如果数据库存在，证明区块链存在，直接结束该方法
//否则进行创建创世区块，并存入数据库中
func CreateBlockChainWithGenesisBlock(data string) {
    if dbExists() {
        fmt.Println("数据库已经存在。。。")
        return
    }

    //
    fmt.Println("创建创世区块：", data)
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
    defer db.Close()
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
    //return &BlockChain{genesisBlock.Hash, db}
} 
```

然后我们在`BlockChain.go`中添加一个方法`GetBlockchainObject()`，专门用于获取`BlockChain`实例对象。代码如下：

```go
//新增方法，用于获取区块链
func GetBlockchainObject() *BlockChain {
    /*
    1.如果数据库不存在，直接返回 nil
    2.读取数据库
     */
    if !dbExists() {
        fmt.Println("数据库不存在，无法获取区块链。。")
        return nil
    }

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
```

最后，在`main.go`中修改代码：

```go
package main

import (
    "./BLC"

)

func main() {
    //9.CLI 操作
    cli:=BLC.CLI{}
    cli.Run()
} 
```

运行结果：创建创世区块

![`img.kongyixueyuan.com/0504_%E5%88%9B%E5%BB%BA%E5%88%9B%E4%B8%96%E5%9D%97.png`](img/15d01283ccde48e6b3acdd9cd933f3e4.jpg)

添加新的区块：

![`img.kongyixueyuan.com/0505_%E6%B7%BB%E5%8A%A0%E6%96%B0%E5%8C%BA%E5%9D%97.png`](img/be5b373ce38535c8aaca86ce717426dd.jpg)

遍历打印区块：

![`img.kongyixueyuan.com/0506_%E9%81%8D%E5%8E%86%E5%8C%BA%E5%9D%97.png`](img/162c72aa0199f7a62561a2193caa79b2.jpg)

## 5\. 总结

通过本章节的学习，我们知道了什么是 CLI，并通过 CLI 命令执行程序。通过`flag`包设置终端命令，通过命令配合命令参数执行对应的功能。本章节中我们并没有新增功能，项目功能目前为止还是 3 个，创建创世区块：`createblockchain`，添加新区块：`addblock`，以及打印区块：`printchain`。

[项目源代码](https://github.com/rubyhan1314/PublicChain)