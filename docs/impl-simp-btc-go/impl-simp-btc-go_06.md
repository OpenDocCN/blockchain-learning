# 第六章 交易 1-(UTXO 模型)

# 交易(Transactions)（1）

交易（transaction）是比特币的核心所在，而区块链唯一的目的，也正是为了能够安全可靠地存储交易。在区块链中，交易一旦被创建，就没有任何人能够再去修改或是删除它。今天，我们将会开始实现交易。不过，由于交易是很大的话题，我们把它分为两部分来讲：在今天这个部分，我们会实现交易的基本框架。在第二部分，我们会继续讨论它的一些细节。

## 1\. 课程目标

1.  了解什么是交易
2.  了解什么是输入和输出
3.  学会创建转账交易
4.  学会 UTXO 模型
5.  学会查询余额

## 2\. 项目代码及效果展示

### 2.1 项目代码结构

![`img.kongyixueyuan.com/0601_%E9%A1%B9%E7%9B%AE%E7%BB%93%E6%9E%84.png`](img/3b0fdf827fb2e9fe460182db6ea39787.jpg)

### 2.2 项目运行结果

![`img.kongyixueyuan.com/0602_%E8%BF%90%E8%A1%8C%E6%95%88%E6%9E%9C.gif`](img/66c62f926a72c2f2ee4e0b4063081a97.jpg)

## 3\. 创建项目

### 3.1 创建工程

首先打开 Goland 开发工具

打开工程：`mypublicchain`

创建项目：将上一次的项目代码，`day02_04_cli`，复制为`day03_05_Transaction`

> 说明：我们每一章节的项目代码，都是在上一个章节上进行添加。所以拷贝上一次的项目代码，然后进行新内容的添加或修改。

### 3.2 代码实现

#### 3.2.1 创建 go 文件：`Transaction.go`

打开`day03_05_Transaction`目录里的 BLC 包,创建`Transaction.go`文件。在`Transaction.go`文件中编写代码如下：

```go
package BLC

import (
    "bytes"
    "encoding/gob"
    "log"
    "crypto/sha256"
    "encoding/hex"
)

//step1：创建 Transaction 结构体
type Transaction struct {
    //1.交易 ID
    TxID []byte
    //2.输入
    Vins []*TXInput
    //3.输出
    Vouts [] *TXOuput
}

//step2:
/*
Transaction 创建分两种情况
1.创世区块创建时的 Transaction

2.转账时产生的 Transaction

 */
func NewCoinBaseTransaction(address string) *Transaction {
    txInput := &TXInput{[]byte{}, -1, "Genesis Data"}
    txOutput := &TXOuput{10, address}
    txCoinbase := &Transaction{[]byte{}, []*TXInput{txInput}, []*TXOuput{txOutput}}
    //设置 hash 值
    //txCoinbase.HashTransaction()
    txCoinbase.SetTxID()
    return txCoinbase
}

//设置交易 ID，其实就是 hash
func (tx *Transaction) SetTxID() {
    var buff bytes.Buffer
    encoder := gob.NewEncoder(&buff)
    err := encoder.Encode(tx)
    if err != nil {
        log.Panic(err)
    }

    buffBytes:=bytes.Join([][]byte{IntToHex(time.Now().Unix()),buff.Bytes()},[]byte{})

    hash := sha256.Sum256(buffBytes)
    tx.TxID = hash[:]
}

func NewSimpleTransaction(from,to string,amount int64,bc *BlockChain,txs []*Transaction)*Transaction{
    var txInputs [] *TXInput
    var txOutputs [] *TXOuput

    balance, spendableUTXO := bc.FindSpendableUTXOs(from,amount,txs)

    //代表消费
    for txID,indexArray:=range spendableUTXO{
        txIDBytes,_:=hex.DecodeString(txID)
        for _,index:=range indexArray{
            txInput := &TXInput{txIDBytes,index,from}
            txInputs = append(txInputs,txInput)
        }
    }

    //转账
    txOutput1 := &TXOuput{amount, to}
    txOutputs = append(txOutputs, txOutput1)

    //找零

    txOutput2 := &TXOuput{balance - amount, from}

    txOutputs = append(txOutputs, txOutput2)

    tx := &Transaction{[]byte{}, txInputs, txOutputs}
    //设置 hash 值
    tx.SetTxID()
    return tx
}

//判断当前交易是否是 Coinbase 交易
func (tx *Transaction) IsCoinbaseTransaction() bool {
    return len(tx.Vins[0].TxID) == 0 && tx.Vins[0].Vout == -1
} 
```

#### 3.2.2 创建`Transaction_TxInput.go`文件

打开`day03_05_Transaction`目录里的 BLC 包。创建`Transaction_TxInput.go`文件。

添加代码如下：

```go
package BLC

type TXInput struct {
    //1.交易的 ID
    TxID [] byte
    //2.存储 Txoutput 的 vout 里面的索引
    Vout int
    //3.用户名
    ScriptSiq string
}

//判断当前 txInput 消费，和指定的 address 是否一致
func (txInput *TXInput) UnLockWithAddress(address string) bool{
    return txInput.ScriptSiq == address
} 
```

#### 3.2.3 创建`Transaction_TxOutput.go`文件

打开`day03_05_Transaction`目录里的 BLC 包。创建`Transaction_TxInput.go`文件。

添加代码如下：

```go
package BLC
//交易的输出
type TXOuput struct {
    Value        int64
    //一个锁定脚本(ScriptPubKey)，要花这笔钱，必须要解锁该脚本。
    ScriptPubKey string //公钥：先理解为，用户名
}

//判断当前 txOutput 消费，和指定的 address 是否一致
func (txOutput *TXOuput) UnLockWithAddress(address string) bool{
    return txOutput.ScriptPubKey == address
} 
```

#### 3.2.4 新建`Transaction_UTXO.go`

打开`day03_05_Transaction`目录里的 BLC 包。新建`Transaction.go`文件。

并编写代码如下：

```go
package BLC

//step1：创建一个结构体 UTXO，用于表示所有未花费的
type UTXO struct {
    TxID   [] byte  //当前 Transaction 的交易 ID
    Index  int      //下标索引
    Output *TXOuput //
} 
```

#### 3.2.5 修改`utils.go`文件

打开`day03_05_Transaction`目录里的 BLC 包。修改`utils.go`文件。

修改步骤：

```go
修改步骤：
step1：添加 JSONToArray()方法，用于解析 JSON 
```

修改完后代码如下：

```go
package BLC

import (
    "bytes"
    "encoding/binary"
    "log"
    "encoding/json"
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

/*
Json 字符串转为[] string 数组
 */
func JSONToArray (jsonString string) [] string{
    var sArr [] string
    if err := json.Unmarshal([]byte(jsonString),&sArr);err != nil{
        log.Panic(err)
    }
    return sArr
} 
```

#### 3.2.6 修改`Block.go`文件

打开`day03_05_Transaction`目录里的 BLC 包。修改`Block.go`文件。

修改步骤：

```go
修改步骤：
step1：修改 Block 结构体
    修改 Data 字段
step2：修改 NewBlock()方法
    添加交易
step3：修改 CreateGenesisBlock()方法
    添加交易
step4：新增方法 HashTransactions()方法
    用于获取一个区块的交易 hash 
```

修改完后代码如下：

```go
package BLC

import (
    "time"
    "bytes"
    "encoding/gob"
    "log"
    "crypto/sha256"
)
//step2:修改 Block 的交易类型
type Block struct {
    //字段：
    //高度 Height：其实就是区块的编号，第一个区块叫创世区块，高度为 0
    Height int64
    //上一个区块的哈希值 ProvHash：
    PrevBlockHash []byte
    //交易数据 Data：目前先设计为[]byte,后期是 Transaction
    //Data [] byte
    Txs [] *Transaction
    //时间戳 TimeStamp：
    TimeStamp int64
    //哈希值 Hash：32 个的字节，64 个 16 进制数
    Hash []byte

    Nonce int64
}

func NewBlock(txs []*Transaction,provBlockHash []byte,height int64) *Block{
    //创建区块
    block:=&Block{height,provBlockHash,txs,time.Now().Unix(),nil,0}
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

func CreateGenesisBlock(txs []*Transaction) *Block{
    return NewBlock(txs,make([] byte,32,32),0)
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

//step4：新增方法
//将 Txs 转为[]byte
func (block *Block) HashTransactions()[]byte{
    var txHashes [][] byte
    var txHash [32]byte
    for _,tx :=range block.Txs{
        txHashes = append(txHashes,tx.TxID)
    }

    txHash = sha256.Sum256(bytes.Join(txHashes,[]byte{}))
    return txHash[:]
} 
```

#### 3.2.7 修改`ProofOfWork.go`文件

打开`day03_05_Transaction`目录里的 BLC 包。修改`ProofOfWork.go`文件。

修改步骤：

```go
修改步骤：
step1：修改 prepareData()方法
    添加交易 hash 
```

修改完后代码如下：

```go
package BLC

import (
    "math/big"
    "bytes"
    "crypto/sha256"
    "fmt"
)

// 0000 0000 0000 0000 1001 0001 0000  .... 0001
//256 位 Hash 里面前面至少有 16 个零
const TargetBit = 16 // 20

type ProofOfWork struct {
    //要验证的区块
    Block *Block

    //大整数存储,目标哈希
    Target *big.Int
}

func NewProofOfWork(block *Block) *ProofOfWork {
    //1.创建一个 big 对象 0000000.....00001
    /*
    0000 0001
    0010 0000
     */
    target := big.NewInt(1)

    //2.左移 256-bits 位
    target = target.Lsh(target, 256-TargetBit)

    return &ProofOfWork{block, target}
}

func (pow *ProofOfWork) Run() ([] byte, int64) {
    //1.将 Block 的属性拼接成字节数组
    //2.生成 Hash
    //3.循环判断 Hash 的有效性，满足条件，跳出循环结束验证
    nonce := 0
    //var hashInt big.Int //用于存储新生成的 hash
    hashInt := new(big.Int)
    var hash [32]byte
    for{
        //获取字节数组
        dataBytes := pow.prepareData(nonce)
        //生成 hash
        hash = sha256.Sum256(dataBytes)
        //fmt.Printf("%d: %x\n",nonce,hash)
        fmt.Printf("\r%d: %x",nonce,hash)
        //将 hash 存储到 hashInt
        hashInt.SetBytes(hash[:])
        //判断 hashInt 是否小于 Block 里的 target
        /*
        Com compares x and y and returns:
        -1 if x < y
        0 if x == y
        1 if x > y
         */
         if pow.Target.Cmp(hashInt) == 1{
             break
         }
         nonce++
    }
    fmt.Println()
    return hash[:], int64(nonce)
}

func (pow *ProofOfWork) prepareData(nonce int)[]byte{
    data := bytes.Join(
        [][] byte{
            pow.Block.PrevBlockHash,
            pow.Block.HashTransactions(),
            IntToHex(pow.Block.TimeStamp),
            IntToHex(int64(TargetBit)),
            IntToHex(int64(nonce)),
            IntToHex(int64(pow.Block.Height)),
        },
        [] byte{},
    )
    return data
} 
```

#### 3.2.8 修改`BlockChain.go`文件

打开`day03_05_Transaction`目录里的 BLC 包。修改`BlockChain.go`文件。

修改步骤：

```go
修改步骤：
step1：添加 MineNewBlock()方法
    用于挖掘新的区块
step2：添加 UnUTXOs()方法
    找到所有未花费的交易输出
step3：添加 GetBalance()方法
    查询余额
step4：添加 FindSpendableUTXOs()方法和 caculate()方法
    转账时查获在可用的 UTXO 
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
    "strconv"
    "encoding/hex"
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
func CreateBlockChainWithGenesisBlock(address string) {
    if dbExists() {
        fmt.Println("数据库已经存在。。。")
        return
    }

    //
    fmt.Println("创建创世区块：")
    //2.数据库不存在，说明第一次创建，然后存入到数据库中
    fmt.Println("数据库不存在。。")
    //A：创建创世区块
    //创建创世区块
    //先创建 coinbase 交易
    txCoinBase := NewCoinBaseTransaction(address)
    genesisBlock := CreateGenesisBlock([]*Transaction{txCoinBase})
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
    //return &BlockChain{genesisBlock.Hash, db}
}

func (bc *BlockChain) AddBlockToBlockChain(txs [] *Transaction) {
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
            newBlock := NewBlock(txs, lastBlock.Hash, lastBlock.Height+1)
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
        //fmt.Printf("\t 数据：%v\n", block.Txs)
        fmt.Println("\t 交易：")
        for _, tx := range block.Txs {
            fmt.Printf("\t\t 交易 ID：%x\n", tx.TxID)
            fmt.Println("\t\tVins:")
            for _, in := range tx.Vins {
                fmt.Printf("\t\t\tTxID:%x\n", in.TxID)
                fmt.Printf("\t\t\tVout:%d\n", in.Vout)
                fmt.Printf("\t\t\tScriptSiq:%s\n", in.ScriptSiq)
            }
            fmt.Println("\t\tVouts:")
            for _, out := range tx.Vouts {
                fmt.Printf("\t\t\tvalue:%d\n", out.Value)
                fmt.Printf("\t\t\tScriptPubKey:%s\n", out.ScriptPubKey)
            }
        }

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

//挖掘新的区块
func (bc *BlockChain) MineNewBlock(from, to, amount []string) {
    /*
    ./bc send -from '["wangergou"]' -to '["lixiaohua"]' -amount '["4"]'
["wangergou"]
["lixiaohua"]
["4"]

     */
    //fmt.Println(from)
    //fmt.Println(to)
    //fmt.Println(amount)
    //1.新建交易
    //2.新建区块
    //3.将区块存入到数据库
    var txs []*Transaction
    for i := 0; i < len(from); i++ {

        amountInt, _ := strconv.ParseInt(amount[i], 10, 64)
        tx := NewSimpleTransaction(from[i], to[i], amountInt, bc, txs)

        txs = append(txs, tx)
    }

    //amountInt, _ := strconv.ParseInt(amount[0], 10, 64)
    //
    //tx := NewSimpleTransaction(from[0], to[0], amountInt, bc)
    //
    //txs = append(txs, tx)

    var block *Block    //数据库中的最后一个 block
    var newBlock *Block //要创建的新的 block
    bc.DB.View(func(tx *bolt.Tx) error {
        b := tx.Bucket([]byte(BLOCKTABLENAME))
        if b != nil {
            hash := b.Get([] byte("l"))
            blockBytes := b.Get(hash)
            block = DeserializeBlock(blockBytes) //数据库中的最后一个 block
        }
        return nil
    })

    newBlock = NewBlock(txs, block.Hash, block.Height+1)

    bc.DB.Update(func(tx *bolt.Tx) error {
        b := tx.Bucket([]byte(BLOCKTABLENAME))
        if b != nil {
            b.Put(newBlock.Hash, newBlock.Serilalize())
            b.Put([]byte("l"), newBlock.Hash)
            bc.Tip = newBlock.Hash
        }
        return nil
    })

}

//找到所有未花费的交易输出
func (bc *BlockChain) UnUTXOs(address string, txs []*Transaction) []*UTXO {
    /*
    1.先遍历未打包的交易(参数 txs)，找出未花费的 Output。
    2.遍历数据库，获取每个块中的 Transaction，找出未花费的 Output。
     */
    var unUTXOs []*UTXO                      //未花费
    spentTxOutputs := make(map[string][]int) //存储已经花费

    //1.添加先从 txs 遍历，查找未花费
    //for i, tx := range txs {
    for i:=len(txs)-1;i>=0;i--{
        unUTXOs = caculate(txs[i], address, spentTxOutputs, unUTXOs)
    }

    bcIterator := bc.Iterator()
    for {
        block := bcIterator.Next()
        //统计未花费
        //2.获取 block 中的每个 Transaction
        for i := len(block.Txs) - 1; i >= 0; i-- {
            unUTXOs = caculate(block.Txs[i], address, spentTxOutputs, unUTXOs)
        }

        //结束迭代
        hashInt := new(big.Int)
        hashInt.SetBytes(block.PrevBlockHash)
        if big.NewInt(0).Cmp(hashInt) == 0 {
            break
        }
    }
    return unUTXOs
}

func (bc *BlockChain) GetBalance(address string, txs []*Transaction) int64 {
    //txOutputs:=bc.UnUTXOs(address)
    unUTXOs := bc.UnUTXOs(address, txs)
    //fmt.Println(address, unUTXOs)
    var amount int64
    for _, utxo := range unUTXOs {
        amount = amount + utxo.Output.Value
    }
    return amount

}

//转账时查获在可用的 UTXO
func (bc *BlockChain) FindSpendableUTXOs(from string, amount int64, txs []*Transaction) (int64, map[string][]int) {
    /*
    1.获取所有的 UTXO
    2.遍历 UTXO

    返回值：map[hash]{index}
     */

    var balance int64
    utxos := bc.UnUTXOs(from, txs)
    //fmt.Println(from,utxos)
    spendableUTXO := make(map[string][]int)
    for _, utxo := range utxos {
        balance += utxo.Output.Value
        hash := hex.EncodeToString(utxo.TxID)
        spendableUTXO[hash] = append(spendableUTXO[hash], utxo.Index)
        if balance >= amount {
            break
        }
    }
    if balance < amount {
        fmt.Printf("%s 余额不足。。总额：%d，需要：%d\n", from,balance,amount)
        os.Exit(1)
    }
    return balance, spendableUTXO

}

func caculate(tx *Transaction, address string, spentTxOutputs map[string][]int, unUTXOs []*UTXO) []*UTXO {
    //2.先遍历 TxInputs，表示花费
    if !tx.IsCoinbaseTransaction() {
        for _, in := range tx.Vins {
            //如果解锁
            if in.UnLockWithAddress(address) {
                key := hex.EncodeToString(in.TxID)
                spentTxOutputs[key] = append(spentTxOutputs[key], in.Vout)
            }
        }
    }

    //fmt.Println("===>", spentTxOutputs)
    //3.遍历 TxOutputs
outputs:
    for index, out := range tx.Vouts {
        if out.UnLockWithAddress(address) {
            //fmt.Println("height,", block.Height, ",index---", index, out, "map-->", spentTxOutputs, len(spentTxOutputs))
            //如果对应的花费容器中长度不为 0,
            if len(spentTxOutputs) != 0 {
                var isSpentUTXO bool

                for txID, indexArray := range spentTxOutputs {
                    for _, i := range indexArray {
                        if i == index && txID == hex.EncodeToString(tx.TxID) {
                            isSpentUTXO = true
                            continue outputs
                        }
                    }
                }
                if !isSpentUTXO {
                    utxo := &UTXO{tx.TxID, index, out}
                    unUTXOs = append(unUTXOs, utxo)
                    //unSpentTxOutputs = append(unSpentTxOutputs, out)
                }

            } else {
                utxo := &UTXO{tx.TxID, index, out}
                unUTXOs = append(unUTXOs, utxo)
                //unSpentTxOutputs = append(unSpentTxOutputs, out)
            }
            //fmt.Println(block.Height, "   ", index, "----....", unUTXOs)
        }
    }
    return unUTXOs
} 
```

#### 3.2.9 修改修改`CLI.go`文件，每个功能新建`CLI_xxx.go`文件

打开`day03_05_Transaction`目录里的 BLC 包。修改`CLI.go`文件。

修改步骤：

```go
修改步骤：
step1：修改 printUsage()方法
    添加转账命令提示信息
step2：修改 Run()方法
    添加 send 转账标签
step3：将功能拆解
    新建 CLI_createBlockChain.go
    新建 CLI_send.go
    新建 CLI_printChains.go
    新建 CLI_getBalance.go 
```

修改完后 CLI.go 代码如下：

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
    sendBlockCmd := flag.NewFlagSet("send",flag.ExitOnError)
    //fmt.Printf("%T\n",addBlockCmd) //*FlagSet
    printChainCmd:=flag.NewFlagSet("printchain",flag.ExitOnError)
    createBlockChainCmd:=flag.NewFlagSet("createblockchain",flag.ExitOnError)
    getBalanceCmd:=flag.NewFlagSet("getbalance",flag.ExitOnError)

    //2.设置标签后的参数
    //flagAddBlockData:= addBlockCmd.String("data","helloworld..","交易数据")
    flagFromData:=sendBlockCmd.String("from","","转帐源地址")
    flagToData:=sendBlockCmd.String("to","","转帐目标地址")
    flagAmountData:=sendBlockCmd.String("amount","","转帐金额")
    flagCreateBlockChainData := createBlockChainCmd.String("address","","创世区块交易地址")
    flagGetBalanceData := getBalanceCmd.String("address","","要查询的某个账户的余额")

    //3.解析
    switch os.Args[1] {
    case "send":
        err:=sendBlockCmd.Parse(os.Args[2:])
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
    case "getbalance":
        err :=getBalanceCmd.Parse(os.Args[2:])
        if err != nil{
            log.Panic(err)
        }

    default:
        printUsage()
        os.Exit(1)//退出
    }

    if sendBlockCmd.Parsed(){
        if *flagFromData == "" || *flagToData =="" ||*flagAmountData == "" {
            printUsage()
            os.Exit(1)
        }
        //cli.addBlock([]*Transaction{})
        fmt.Println(*flagFromData)
        fmt.Println(*flagToData)
        fmt.Println(*flagAmountData)
        //fmt.Println(JSONToArray(*flagFrom))
        //fmt.Println(JSONToArray(*flagTo))
        //fmt.Println(JSONToArray(*flagAmount))
        from:=JSONToArray(*flagFromData)
        to:=JSONToArray(*flagToData)
        amount:=JSONToArray(*flagAmountData)

        cli.send(from,to,amount)
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

    if getBalanceCmd.Parsed(){
        if *flagGetBalanceData == ""{
            fmt.Println("查询地址不能为空")
            printUsage()
            os.Exit(1)
        }
        cli.getBalance(*flagGetBalanceData)

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
    fmt.Println("\tcreateblockchain -address DATA -- 创建创世区块")
    fmt.Println("\tsend -from From -to To -amount Amount - 交易数据")
    fmt.Println("\tprintchain - 输出信息")
    fmt.Println("\tgetbalance -address DATA -- 查询账户余额")
} 
```

新建`CLI_createBlockChain.go`，代码如下:

```go
package BLC

func (cli *CLI) createGenesisBlockchain(address string){
    //fmt.Println(data)
    CreateBlockChainWithGenesisBlock(address)

} 
```

新建`CLI_send.go`，代码如下:

```go
package BLC

import (
    "fmt"
    "os"
)

//转账
func (cli *CLI) send(from, to, amount [] string) {
    if !dbExists() {
        fmt.Println("数据库不存在。。。")
        os.Exit(1)
    }
    blockchain := GetBlockchainObject()

    blockchain.MineNewBlock(from, to, amount)
    defer blockchain.DB.Close()
} 
```

新建`CLI_printChains.go`，代码如下:

```go
package BLC

import (
    "fmt"
    "os"
)

func (cli *CLI)printChains(){
    bc:=GetBlockchainObject()
    if bc == nil{
        fmt.Println("没有区块可以打印。。")
        os.Exit(1)
    }
    defer bc.DB.Close()
    bc.PrintChains()
} 
```

新建`CLI_getBalance.go`，代码如下：

```go
package BLC

import (
    "fmt"
    "os"
)

//查询余额
func (cli *CLI)getBalance(address string){
    fmt.Println("查询余额：",address)
    bc := GetBlockchainObject()

    if bc == nil{
        fmt.Println("数据库不存在，无法查询。。")
        os.Exit(1)
    }
    defer bc.DB.Close()
    //txOutputs:= bc.UnUTXOs(address)
    //for i,out:=range txOutputs{
    //    fmt.Println(i,"---->",out)
    //}
    balance:=bc.GetBalance(address,[]*Transaction{})
    fmt.Printf("%s,一共有%d 个 Token\n",address,balance)
} 
```

#### 3.2.10 `main.go`无需修改

在`main.go`中代码不变，依然如下:

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

## 4\. Transaction 讲解

### 4.1 交易 Transaction

1、交易，简单地说就是把比特币从一个地址转到另一个地址，准确地说，一笔交易指一个经过签名运算的，表达价值转移的数据结构；

2、交易实质上是包含了一组输入列表和输出列表的数据结构，也就是转账记录。

*   比特币交易都是由 inputs（输入）、outputs（输出）组成。可以简单理解为发币地址是输入、收币地址是输出。
*   inputs 用来追溯上一笔交易，以便明确转出者是否有权动用这笔钱，outputs 用来进行一次新的加密，加密后只有收款者才能解密并动用这笔钱。
*   输入中包含解锁脚本（unlocking script），输出中包含锁定脚本（locking script）。
*   锁定脚本往往含有一个公钥或比特币地址，所以曾经被称为公钥脚本，在代码中常用 scriptPubKey 表示。
*   解锁脚本往往是支付方用自己的私钥所做的签名，曾被称为签名脚本，代码中用 scriptSig 表示。
*   在一个交易中，锁定脚本相当于是加密难题，解锁脚本是解开锁定脚本的题解。但解锁脚本的题解是针对上一笔交易输出中产生的加密难题的题解，而并非当前交易。

![`img.kongyixueyuan.com/0615_%E4%BA%A4%E6%98%93.png`](img/951eb80e1d750db927ead59a4940976a.jpg)

> *   交易 a 中，A 转账给 B
> *   交易 b 中，B 转账给 C
> *   交易 c 中，C 转账给 D
> *   在交易 a 中，当 A 给 B 转账时，A 给 B 出了一道加密难题，以脚本的形式附加在了转账金额末尾，该脚本锁定了其中的资产。
> *   在交易 b 中，B 要转账给 C，需要花费当初 A 转给他的资产。作为条件，B 必须解开 A 给他出的加密难题才能花费其中被锁定的资产。所以 b 交易中的输入中必须有解锁脚本。该解锁脚本用来解交易 a 中的锁定脚本。
> *   在交易 b 中，当 B 给 C 转账时，B 给 C 同样出了一道加密难题，将其中的资产进行了锁定。
> *   在交易 d 中，C 要转账给 D，就需要在输入中有解锁脚本。该解锁脚本是用来解交易 b 中的锁定脚本。

由于比特币采用的是 UTXO 模型，并非账户模型，并不直接存在“余额”这个概念，余额需要通过遍历整个交易历史得来。

**比特币交易**：(点击 [这里](https://blockchain.info/zh-cn/tx/b6f6b339b546a13822192b06ccbdd817afea5311845f769702ae2912f7d94ab5) 在 blockchain.info 查看下图中的交易信息。)

![`img.kongyixueyuan.com/0603_%E6%AF%94%E7%89%B9%E5%B8%81%E4%BA%A4%E6%98%93.png`](img/bab08a6bd0587f86d452342a66503122.jpg)

从上图可以看出，比特币中的交易，都是由一些输入（input）和输出（output）组合而来。

在`Transaction.go`文件中，添加 Transaction 结构体。

```go
type Transaction struct {
    //1.交易 ID
    TxID []byte
    //2.输入
    Vins []*TXInput
    //3.输出
    Vouts [] *TXOuput
} 
```

对于每一笔新的交易，它的输入会引用（reference）之前一笔交易的输出（这里有个例外，coinbase 交易），引用就是花费的意思。所谓引用之前的一个输出，也就是将之前的一个输出包含在另一笔交易的输入当中，就是花费之前的交易输出。交易的输出，就是币实际存储的地方。下面的图示阐释了交易之间的互相关联：

![`img.kongyixueyuan.com/0604_inputoutput.png`](img/f3eb4ca1b796376c50d139e3f52affef.jpg)

注意：

1.  有一些输出并没有被关联到某个输入上
2.  一笔交易的输入可以引用之前多笔交易的输出
3.  一个输入必须引用一个输出

贯穿本文，我们将会使用像“钱（money）”，“币（coin）”，“花费（spend）”，“发送（send）”，“账户（account）” 等等这样的词。但是在比特币中，其实并不存在这样的概念。交易仅仅是通过一个脚本（script）来锁定（lock）一些值（value），而这些值只可以被锁定它们的人解锁（unlock）。

每一笔比特币交易都会创造输出，输出都会被区块链记录下来。给某个人发送比特币，实际上意味着创造新的 UTXO 并注册到那个人的地址，可以为他所用。

### 4.2 交易输出

在`Transaction_TxOutput.go`中，添加 TXOutput 结构体。

```go
type TXOuput struct {
    Value        int64
    //一个锁定脚本(ScriptPubKey)，要花这笔钱，必须要解锁该脚本。
    ScriptPubKey string //公钥：先理解为，用户名
} 
```

输出主要包含两部分：

1.  一定量的比特币(`Value`)
2.  一个锁定脚本(`ScriptPubKey`)，要花这笔钱，必须要解锁该脚本。

实际上，正是输出里面存储了“币”（注意，也就是上面的 `Value` 字段）。而这里的存储，指的是用一个数学难题对输出进行锁定，这个难题被存储在 `ScriptPubKey` 里面。在内部，比特币使用了一个叫做 *Script* 的脚本语言，用它来定义锁定和解锁输出的逻辑。虽然这个语言相当的原始（这是为了避免潜在的黑客攻击和滥用而有意为之），并不复杂，但是我们也并不会在这里讨论它的细节。

> 在比特币中，`value` 字段存储的是 *satoshi* 的数量，而不是 BTC 的数量。一个 *satoshi* 等于一亿分之一的 BTC(0.00000001 BTC)，这也是比特币里面最小的货币单位（就像是 1 分的硬币）。

由于还没有实现地址（address），所以目前我们会避免涉及逻辑相关的完整脚本。`ScriptPubKey` 将会存储一个任意的字符串（用户定义的钱包地址）。

> 顺便说一下，有了一个这样的脚本语言，也意味着比特币其实也可以作为一个智能合约平台。

关于输出，非常重要的一点是：它们是**不可再分的（indivisible）**。也就是说，你无法仅引用它的其中某一部分。要么不用，如果要用，必须一次性用完。当一个新的交易中引用了某个输出，那么这个输出必须被全部花费。如果它的值比需要的值大，那么就会产生一个找零，找零会返还给发送方。这跟现实世界的场景十分类似，当你想要支付的时候，如果一个东西值 1 美元，而你给了一个 5 美元的纸币，那么你会得到一个 4 美元的找零。

### 4.3 交易输入

在`Transaction_TxInput.go`中，添加 TXInput 结构体。

```go
type TXInput struct {
    //1.交易的 ID
    TxID [] byte
    //2.存储 Txoutput 的 vout 里面的索引
    Vout int
    //3.用户名
    ScriptSiq string
} 
```

正如之前所提到的，一个输入引用了之前交易的一个输出：`TxiD` 存储的是之前交易的 ID，`Vout` 存储的是该输出在那笔交易中所有输出的索引（因为一笔交易可能有多个输出，需要有信息指明是具体的哪一个）。`ScriptSig` 是一个脚本，提供了可解锁输出结构里面 `ScriptPubKey` 字段的数据。如果 `ScriptSig` 提供的数据是正确的，那么输出就会被解锁，然后被解锁的值就可以被用于产生新的输出；如果数据不正确，输出就无法被引用在输入中，或者说，无法使用这个输出。这种机制，保证了用户无法花费属于其他人的币。

再次强调，由于我们还没有实现地址，所以目前 `ScriptSig` 将仅仅存储一个用户自定义的任意钱包地址。我们会在下一篇文章中实现公钥（public key）和签名（signature）。

来简要总结一下。输出，就是 “币” 存储的地方。每个输出都会带有一个解锁脚本，这个脚本定义了解锁该输出的逻辑。每笔新的交易，必须至少有一个输入和输出。一个输入引用了之前一笔交易的输出，并提供了解锁数据（也就是 `ScriptSig` 字段），该数据会被用在输出的解锁脚本中解锁输出，解锁完成后即可使用它的值去产生新的输出。

每一笔输入都是之前一笔交易的输出，那么假设从某一笔交易开始不断往前追溯，它所涉及的输入和输出到底是谁先存在呢？换个说法，这是个鸡和蛋谁先谁后的问题，是先有蛋还是先有鸡呢？

### 4.4 CoinBase 交易

在比特币中，是先有蛋，然后才有鸡。输入引用输出的逻辑，是经典的“蛋还是鸡”问题：输入先产生输出，然后输出使得输入成为可能。在比特币中，最先有输出，然后才有输入。换而言之，第一笔交易只有输出，没有输入。

当矿工挖出一个新的块时，它会向新的块中添加一个 **coinbase** 交易。coinbase 交易是一种特殊的交易，它不需要引用之前一笔交易的输出。它“凭空”产生了币（也就是产生了新币），这是矿工获得挖出新块的奖励，也可以理解为“发行新币”。

在区块链的最初，也就是第一个块，叫做创世块。正是这个创世块，产生了区块链最开始的输出。对于创世块，不需要引用之前的交易输出。因为在创世块之前根本不存在交易，也就没有不存在交易输出。

在`Transaction.go`文件 中，添加一个方法，用于创建一个 coinbase 交易，代码如下：

```go
/*
Transaction 创建分两种情况
1.创世区块创建时的 Transaction
2.转账时产生的 Transaction
*/
func NewCoinBaseTransaction(address string) *Transaction {
    txInput := &TXInput{[]byte{}, -1, "Genesis Data"}
    txOutput := &TXOuput{10, address}
    txCoinbase := &Transaction{[]byte{}, []*TXInput{txInput}, []*TXOuput{txOutput}}
    //设置 hash 值
    //txCoinbase.HashTransaction()
    txCoinbase.SetTxID()
    return txCoinbase
}

//设置交易 ID，其实就是 hash
func (tx *Transaction) SetTxID() {
    var buff bytes.Buffer
    encoder := gob.NewEncoder(&buff)
    err := encoder.Encode(tx)
    if err != nil {
        log.Panic(err)
    }

    buffBytes:=bytes.Join([][]byte{IntToHex(time.Now().Unix()),buff.Bytes()},[]byte{})

    hash := sha256.Sum256(buffBytes)
    tx.TxID = hash[:]
} 
```

coinbase 交易只有一个输出，没有输入。在我们的实现中，它表现为 `TxiD` 为空，`Vout` 等于 -1。并且，在当前实现中，coinbase 交易也没有在 `ScriptSig` 中存储脚本，而只是存储了一个任意的字符串 `data`。

> 在比特币中，第一笔 coinbase 交易包含了如下信息：“The Times 03/Jan/2009 Chancellor on brink of second bailout for banks”。[可点击这里查看](https://blockchain.info/tx/4a5e1e4baab89f3a32518a88c31bc87f618f76673e2cc77ab2127b7afdeda33b?show_adv=true).

`subsidy` 是挖出新块的奖励金。在比特币中，实际并没有存储这个数字，而是基于区块总数进行计算而得：区块总数除以 210000 就是 `subsidy`。挖出创世块的奖励是 50 BTC，每挖出 `210000` 个块后，奖励减半。在我们的实现中，这个奖励值将会是一个常量，只是目前我们代码中还没有加入挖矿奖励金。

### 4.5 将交易保存到区块链

从现在开始，每个块必须存储至少一笔交易。如果没有交易，也就不可能出新的块。这意味着我们应该移除 `Block` 的 `Data`字段，取而代之的是存储交易。

在`Block.go`文件中，修改 Block 结构体，代码如下:

```go
type Block struct {
    //字段：
    //高度 Height：其实就是区块的编号，第一个区块叫创世区块，高度为 0
    Height int64
    //上一个区块的哈希值 ProvHash：
    PrevBlockHash []byte
    //交易数据 Data：目前先设计为[]byte,后期是 Transaction
    //Data [] byte
    Txs [] *Transaction
    //时间戳 TimeStamp：
    TimeStamp int64
    //哈希值 Hash：32 个的字节，64 个 16 进制数
    Hash []byte

    Nonce int64
} 
```

接下来，`NewBlock()` 和 `CreateGenesisBlock()` 也必须做出相应改变，修改后代码如下：

```go
func NewBlock(txs []*Transaction,provBlockHash []byte,height int64) *Block{
    //创建区块
    block:=&Block{height,provBlockHash,txs,time.Now().Unix(),nil,0}
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
```

以及如下:

```go
func CreateGenesisBlock(txs []*Transaction) *Block{
    return NewBlock(txs,make([] byte,32,32),0)
} 
```

接下来修改`BlockChain.go`文件中，创建区块链的函数：

```go
func CreateBlockChainWithGenesisBlock(address string) {
    if dbExists() {
        fmt.Println("数据库已经存在。。。")
        return
    }

    //
    fmt.Println("创建创世区块：")
    //2.数据库不存在，说明第一次创建，然后存入到数据库中
    fmt.Println("数据库不存在。。")
    //A：创建创世区块
    //创建创世区块
    //先创建 coinbase 交易
    txCoinBase := NewCoinBaseTransaction(address)
    genesisBlock := CreateGenesisBlock([]*Transaction{txCoinBase})
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
    //return &BlockChain{genesisBlock.Hash, db}
} 
```

现在，这个函数会接受一个地址作为参数，这个地址将会被用来接收挖出创世块的奖励。

### 4.6 工作量证明

工作量证明算法必须要将存储在区块里面的交易考虑进去，从而保证区块链交易存储的一致性和可靠性。所以，我们必须修改`ProofOfWork.go`文件中的 `prepareData()`方法，修改后代码如下：

```go
func (pow *ProofOfWork) prepareData(nonce int)[]byte{
    data := bytes.Join(
        [][] byte{
            pow.Block.PrevBlockHash,
            pow.Block.HashTransactions(),//此行代码改变
            IntToHex(pow.Block.TimeStamp),
            IntToHex(int64(TargetBit)),
            IntToHex(int64(nonce)),
            IntToHex(int64(pow.Block.Height)),
        },
        [] byte{},
    )
    return data
} 
```

接下来我们需要在 Block.go 文件中，添加一个 Block 对象的方法，HashTransactions()，代码如下：

```go
func (block *Block) HashTransactions()[]byte{
    var txHashes [][] byte
    var txHash [32]byte
    for _,tx :=range block.Txs{
        txHashes = append(txHashes,tx.TxID)
    }

    txHash = sha256.Sum256(bytes.Join(txHashes,[]byte{}))
    return txHash[:]
} 
```

通过哈希提供数据的唯一表示，这种做法我们已经不是第一次遇到了。我们想要通过仅仅一个哈希，就可以识别一个块里面的所有交易。为此，先获得每笔交易的哈希，然后将它们关联起来，最后获得一个连接后的组合哈希。

> 比特币使用了一个更加复杂的技术：它将一个块里面包含的所有交易表示为一个 [Merkle tree](https://en.wikipedia.org/wiki/Merkle_tree) ，然后在工作量证明系统中使用树的根哈希（root hash）。这个方法能够让我们快速检索一个块里面是否包含了某笔交易，即只需 root hash 而无需下载所有交易即可完成判断。

来检查一下到目前为止是否正确：

首先先创建一个创世区块，运行效果如下：

![`img.kongyixueyuan.com/0605_%E5%88%9B%E5%BB%BA%E5%88%9B%E4%B8%96%E5%9D%97.png`](img/8681a59f42c3752313d17df6f21a64a6.jpg)

打印区块信息，效果如下:

![`img.kongyixueyuan.com/0606_%E6%89%93%E5%8D%B0%E5%88%9B%E4%B8%96%E5%8C%BA%E5%9D%97.png`](img/aa7c55c4c0eb0fa7c27d0320791e0783.jpg)

很好！我们已经获得了第一笔挖矿奖励。

### 4.7 发送币

现在，我们想要给其他人发送一些币。为此，我们需要创建一笔新的交易，将它放到一个块里，然后挖出这个块。之前我们只实现了 coinbase 交易（这是一种特殊的交易），现在我们需要一种通用的普通交易。这部分相当的复杂，接下来我们一步一步来实现。

之前已经提到过，每笔交易，都包含输入和输出。输入要引用之前的未花费的输出。

我们模拟这样一个场景：我们用 hanru 的地址创建一个创世区块，创建 CoinBase 交易后，hanru 有 10 个 Token。然后我们进行转账，hanru 转给 wangergou4 个 Token。

![`img.kongyixueyuan.com/0607_%E8%BD%AC%E8%B4%A6.png`](img/16d54b51f501c38173c05a2f14f68dfb.jpg)

> 说明：
> 
> 1.因为创建了创世区块，所以产生了 CoinBase 交易，它的 Input 是空的，Output 表示未花费的输出。理解为 hanru 有 10 个 Token(Unspent)。
> 
> 2.hanru 给 wangergou 转账，那么就表示 hanru 要花费掉自己的未花费的 output，创建 input，产生新的 output，创建普通交易。
> 
> 3.Input 中的 TxID 字段，表示引用的未花费的 output 所在的交易 ID，Vout 表示引用的未花费的 output 在所在交易的下标(我们 Transaction 中的 output 采用数组存储)。ScriptSiq 目前仅仅当做账户名(实际上并没有账户名这个东西，因为我们没有学习地址，暂且这样理解)。

综上，要想实现真正的转账，就要找到该账户下的未花费的 output，但是因为 output 中仅仅设置了 Value 和 ScriptPubKey 两个字段。所以我们在转账的时候要知道该 output 所在的交易 ID 和下标，所以我们可以根据 output 创建 UTXO。

所以此处我们创建一个新的 go 文件，命名为`Transaction_UTXO.go`。并在其中添加一个 UTXO 的结构体。代码如下:

```go
package BLC

//step1：创建一个结构体 UTXO，用于表示所有未花费的
type UTXO struct {
    TxID   [] byte  //当前 Transaction 的交易 ID
    Index  int      //下标索引
    Output *TXOuput //要使用的未花费的 Output
} 
```

### 4.8 查找未花费的 UTXO

不管要实现转账还是查询余额，我们需要找出某个账户下所有的未花费交易输出（unspent transactions outputs, UTXO）。**未花费（unspent）** 指的是这个输出还没有被包含在任何交易的输入中，或者说没有被任何输入引用。

接下来我们来实现算法。

在`BlockChain.go`文件中，添加 UnUTXOs()方法，用于查找所有未花费。代码如下:

```go
 //找到所有未花费的交易输出

//找到所有未花费的交易输出
func (bc *BlockChain) UnUTXOs(address string, txs []*Transaction) []*UTXO {
    /*
    1.先遍历未打包的交易(参数 txs)，找出未花费的 Output。
    2.遍历数据库，获取每个块中的 Transaction，找出未花费的 Output。
     */
    var unUTXOs []*UTXO                      //未花费
    spentTxOutputs := make(map[string][]int) //存储已经花费

    //1.添加先从 txs 遍历，查找未花费
    //for i, tx := range txs {
    for i:=len(txs)-1;i>=0;i--{
        unUTXOs = caculate(txs[i], address, spentTxOutputs, unUTXOs)
    }

    bcIterator := bc.Iterator()
    for {
        block := bcIterator.Next()
        //统计未花费
        //2.获取 block 中的每个 Transaction
        for i := len(block.Txs) - 1; i >= 0; i-- {
            unUTXOs = caculate(block.Txs[i], address, spentTxOutputs, unUTXOs)
        }

        //结束迭代
        hashInt := new(big.Int)
        hashInt.SetBytes(block.PrevBlockHash)
        if big.NewInt(0).Cmp(hashInt) == 0 {
            break
        }
    }
    return unUTXOs
} 
```

以及`caculate()`方法，代码如下:

```go
 func caculate(tx *Transaction, address string, spentTxOutputs map[string][]int, unUTXOs []*UTXO) []*UTXO {
    //2.先遍历 TxInputs，表示花费
    if !tx.IsCoinbaseTransaction() {
        for _, in := range tx.Vins {
            //如果解锁
            if in.UnLockWithAddress(address) {
                key := hex.EncodeToString(in.TxID)
                spentTxOutputs[key] = append(spentTxOutputs[key], in.Vout)
            }
        }
    }

    //fmt.Println("===>", spentTxOutputs)
    //3.遍历 TxOutputs
outputs:
    for index, out := range tx.Vouts {
        if out.UnLockWithAddress(address) {
            //fmt.Println("height,", block.Height, ",index---", index, out, "map-->", spentTxOutputs, len(spentTxOutputs))
            //如果对应的花费容器中长度不为 0,
            if len(spentTxOutputs) != 0 {
                var isSpentUTXO bool

                for txID, indexArray := range spentTxOutputs {
                    for _, i := range indexArray {
                        if i == index && txID == hex.EncodeToString(tx.TxID) {
                            isSpentUTXO = true
                            continue outputs
                        }
                    }
                }
                if !isSpentUTXO {
                    utxo := &UTXO{tx.TxID, index, out}
                    unUTXOs = append(unUTXOs, utxo)
                    //unSpentTxOutputs = append(unSpentTxOutputs, out)
                }

            } else {
                utxo := &UTXO{tx.TxID, index, out}
                unUTXOs = append(unUTXOs, utxo)
                //unSpentTxOutputs = append(unSpentTxOutputs, out)
            }
            //fmt.Println(block.Height, "   ", index, "----....", unUTXOs)
        }
    }
    return unUTXOs
} 
```

在本项目代码中，我们的实现分为两种情况，一种是转账时产生一笔交易，一种是转账时产生多笔交易。对应的命令分别如下：

```go
// 一笔交易：韩茹转账给王二狗 3 个 Token
send -from '["hanru"]' -to '["wangergou"]' -amount '["3"]'

//多笔交易：表示韩茹转账给 ruby，4 个 Token，王二狗转账给李小花 2 个 Token。
send -from '["hanru","wangergou"]' -to '["ruby","lixiaohua"]' -amount '["4","2"]' 
```

所以此处我们拼接了一个 json 字符串。那么就需要在`utils.go`文件中，添加一个解析 json 的方法，代码如下：

```go
/*
Json 字符串转为[] string 数组
 */
func JSONToArray (jsonString string) [] string{
    var sArr [] string
    if err := json.Unmarshal([]byte(jsonString),&sArr);err != nil{
        log.Panic(err)
    }
    return sArr
} 
```

当然了，无论是转账还是查询余额时，我们并不需要知道整个区块链上所有的 UTXO，只需要关注那些我们能够解锁的那些 UTXO（目前我们还没有实现密钥，所以我们将会使用用户定义的地址来代替）。首先，让我们定义在输入和输出上的锁定和解锁方法：

```go
//判断当前 txInput 消费，和指定的 address 是否一致
func (txInput *TXInput) UnLockWithAddress(address string) bool{
    return txInput.ScriptSiq == address
} 
```

```go
//判断当前 txOutput 消费，和指定的 address 是否一致
func (txOutput *TXOuput) UnLockWithAddress(address string) bool{
    return txOutput.ScriptPubKey == address
} 
```

在这里，我们只是将 script 字段与 address 进行了比较。在后续文章我们基于私钥实现了地址以后，会对这部分进行改进。

好了，现在我们针对于以上的查找未花费的 UTXO 算法，进行说明，其实这一步相当困难：

如果存在多笔交易，比如之前的韩茹和王二狗同时进行转账，韩茹转账产生的交易，因为还没有添加到区块中，所以可以叫做未打包的交易。那么王二狗在转账时，可以使用这个未打包的交易中所产生的未花费 UTXO。

那么我们在统计未花费的 UTXO 时，就需要先统计未打包的交易中的 UTXO，以及之前所有的区块的交易列表中的 UTXO。因为我们都是使用数组来存储交易，后面的交易创建时，需要使用之前的交易中的未花费 Output，所以应该从最后的交易，向前遍历每个交易。也就是倒叙遍历每个交易。

在`UnUTXOs()`方法中，第一部分是查找未打包的交易中的未花费的 Output，创建出对应的 UTXO：

```go
//添加先从 txs 遍历，查找未花费
//for i, tx := range txs {
for i:=len(txs)-1;i>=0;i--{
    unUTXOs = caculate(txs[i], address, spentTxOutputs, unUTXOs)
} 
```

第二部分就是遍历数据库，获取每个 Block 区块

由于交易被存储在区块里，所以我们不得不检查区块链里的每个 Block 中的每一笔交易。对于该笔交易，依次遍历找出里面的未花费的 Output，创建出对应的 UTXO：

```go
bcIterator := bc.Iterator()
    for {
        block := bcIterator.Next()
        //统计未花费
        //2.获取 block 中的每个 Transaction
        for i := len(block.Txs) - 1; i >= 0; i-- {
            unUTXOs = caculate(block.Txs[i], address, spentTxOutputs, unUTXOs)
        }

        //结束迭代
        ...
    } 
```

因为不论是获取未打包的交易中的未花费的 Output，还是获取数据库中区块里交易中的未花费的 Output，算法都是一致的。所以我们可以设计一个方法`caculate()`，用于找出某个交易中的所有的未花费的 Output，然后创建对应的 UTXO。

首先创建一个 map(`spentTxOutputs := make(map[string][]int) //存储已经花费`)，用于存储该交易中的所有的 Input 信息，表示已经花费。

```go
if !tx.IsCoinbaseTransaction() {
        for _, in := range tx.Vins {
            //如果解锁
            if in.UnLockWithAddress(address) {
                key := hex.EncodeToString(in.TxID)
                spentTxOutputs[key] = append(spentTxOutputs[key], in.Vout)
            }
        }
    } 
```

因为 CoinBase 交易中没有 Input，所以我们需要判断是否是 CoinBase 交易，如果不是那么要存储该交易中的 Input 信息。

然后判断交易中的每个 Output，如果该 Output 被一个地址锁定，并且这个地址恰好是我们要找的地址，那么这个输出就是我们想要的。不过在获取它之前，我们需要对比存储了 Input 信息的 map，检查该输出是否已经被包含在一个交易的输入中，也就是检查它是否已经被花费了：

如果 output 的下标以及当前交易的 TxID，和 map 中存储的信息对应，那么表示该 Output 已经被花费掉了，使用`isSpentUTXO`变量进行标记一下。

```go
var isSpentUTXO bool
for txID, indexArray := range spentTxOutputs {
    for _, i := range indexArray {
        if i == index && txID == hex.EncodeToString(tx.TxID) {
            isSpentUTXO = true
            continue outputs
        }
    }
} 
```

我们跳过那些已经被包含在其他输入中的输出（这说明这个输出已经被花费，无法再用了）。检查完输出以后，我们将给定地址所有能够解锁的 Output（这并不适用于 coinbase 交易，因为它们不解锁输出），创建对应的 UTXO，并存储到 unUTXOs 的数组中。

```go
if !isSpentUTXO {
    utxo := &UTXO{tx.TxID, index, out}
    unUTXOs = append(unUTXOs, utxo)
    //unSpentTxOutputs = append(unSpentTxOutputs, out)
} 
```

### 4.9 创建转账交易

根据终端命令获取到转账信息，比如`send -from '["hanru"]' -to '["wangergou"]' -amount '["4"]'`，表示韩茹要转账给王二狗 4 个 Token。那么我们可以根据上一步的方法`UnUTXOs()`找出韩茹名下所有的未花费的 UTXO。如果要实现转账，我们就需要在这些 UTXO 中，本次转账要使用的 UTXO。

```go
 //转账时查获在可用的 UTXO
func (bc *BlockChain) FindSpendableUTXOs(from string, amount int64, txs []*Transaction) (int64, map[string][]int) {
    /*
    1.获取所有的 UTXO
    2.遍历 UTXO

    返回值：map[hash]{index}
     */

    var balance int64
    utxos := bc.UnUTXOs(from, txs)
    //fmt.Println(from,utxos)
    spendableUTXO := make(map[string][]int)
    for _, utxo := range utxos {
        balance += utxo.Output.Value
        hash := hex.EncodeToString(utxo.TxID)
        spendableUTXO[hash] = append(spendableUTXO[hash], utxo.Index)
        if balance >= amount {
            break
        }
    }
    if balance < amount {
        fmt.Printf("%s 余额不足。。总额：%d，需要：%d\n", from,balance,amount)
        os.Exit(1)
    }
    return balance, spendableUTXO

} 
```

我们需要遍历这些 UTXO，并统计总金额，如果大于本次要转账的金额，那么使用总金额减掉要转账的金额，就是本次交易的找零。所以我们需要将总金额也一并返回。如果遍历了所有的 UTXO，总金额小于要转账的金额，那么表示余额不足，无法实现本次转账。需要结束程序。

然后我们去创建交易，在 Transaction.go 文件中，添加`NewSimpleTransaction()`方法，用于创建转账交易，代码如下：

```go
 func NewSimpleTransaction(from,to string,amount int64,bc *BlockChain,txs []*Transaction)*Transaction{
    var txInputs [] *TXInput
    var txOutputs [] *TXOuput

    balance, spendableUTXO := bc.FindSpendableUTXOs(from,amount,txs)

    //代表消费

    //txInput := &TXInput{bytes, 0, from}
    //txInputs = append(txInputs, txInput)

    for txID,indexArray:=range spendableUTXO{
        txIDBytes,_:=hex.DecodeString(txID)
        for _,index:=range indexArray{
            txInput := &TXInput{txIDBytes,index,from}
            txInputs = append(txInputs,txInput)
        }
    }

    //转账
    txOutput1 := &TXOuput{amount, to}
    txOutputs = append(txOutputs, txOutput1)

    //找零
    //txOutput2 := &TXOuput{10 - amount, from}
    //txOutput2 := &TXOuput{4 - amount, from}
    txOutput2 := &TXOuput{balance - amount, from}

    txOutputs = append(txOutputs, txOutput2)

    tx := &Transaction{[]byte{}, txInputs, txOutputs}
    //设置 hash 值
    tx.SetTxID()
    return tx
} 
```

首先要调用`FindSpendableUTXOs()`方法，获取本次转账要使用的 UTXO 以及总额。然后根据这些 UTXO，创建 txInput，并添加到 txInputs 中。

然后创建 txOutput，一个是转账金额的去向，一个转账产生的找零，并添加到 txOutputs 中。

根据 txInputs，txOutputs 创建交易，并设置 TxID。

### 4.10 根据转账交易创建区块

在此处，我们直接根据转账信息，创建交易，根据交易进行挖矿产生新的区块，并将新区块添加到数据库中，表示上链。

在 BlockChain.go 文件中，添加`MineNewBlock()`方法，用于挖掘新的区块，代码如下 :

```go
//挖掘新的区块
func (bc *BlockChain) MineNewBlock(from, to, amount []string) {
    /*
    ./bc send -from '["wangergou"]' -to '["lixiaohua"]' -amount '["4"]'
["wangergou"]
["lixiaohua"]
["4"]

     */
    //fmt.Println(from)
    //fmt.Println(to)
    //fmt.Println(amount)
    //1.新建交易
    //2.新建区块
    //3.将区块存入到数据库
    var txs []*Transaction
    for i := 0; i < len(from); i++ {

        amountInt, _ := strconv.ParseInt(amount[i], 10, 64)
        tx := NewSimpleTransaction(from[i], to[i], amountInt, bc, txs)

        txs = append(txs, tx)
    }

    //amountInt, _ := strconv.ParseInt(amount[0], 10, 64)
    //
    //tx := NewSimpleTransaction(from[0], to[0], amountInt, bc)
    //
    //txs = append(txs, tx)

    var block *Block    //数据库中的最后一个 block
    var newBlock *Block //要创建的新的 block
    bc.DB.View(func(tx *bolt.Tx) error {
        b := tx.Bucket([]byte(BLOCKTABLENAME))
        if b != nil {
            hash := b.Get([] byte("l"))
            blockBytes := b.Get(hash)
            block = DeserializeBlock(blockBytes) //数据库中的最后一个 block
        }
        return nil
    })

    newBlock = NewBlock(txs, block.Hash, block.Height+1)

    bc.DB.Update(func(tx *bolt.Tx) error {
        b := tx.Bucket([]byte(BLOCKTABLENAME))
        if b != nil {
            b.Put(newBlock.Hash, newBlock.Serilalize())
            b.Put([]byte("l"), newBlock.Hash)
            bc.Tip = newBlock.Hash
        }
        return nil
    })

} 
```

首先根据转账信息创建交易，查询数据库获取最后一个区块的信息，创建新的区块，并存储到数据库中。

现在我们可以测试一下了，打开终端执行以下命令：

```go
hanru:day03_05_Transaction ruby$ ./bc send -from '["hanru"]' -to '["wangergou"]' -amount '["4"]' 
```

运行效果如下：

![`img.kongyixueyuan.com/0609_%E8%BD%AC%E8%B4%A6.png`](img/81e00e5023c258b1455958a232af15fd.jpg)

查看新的区块，在终端输入以下命令：

```go
hanru:day03_05_Transaction ruby$ ./bc printchain 
```

运行效果如下：

![`img.kongyixueyuan.com/0610.png`](img/a211425f534539fabbdc3b48134959e7.jpg)

### 4.11 查询余额

根据终端输入命令，查询指定账户的余额。

```go
./bc getbalance -address 'hanru' 
```

那么就需要获取韩茹账户下的所有的未花费的 UTXO，然后累加所有的 Value，得到的总和就是余额。

在`BlockChain.go`中添加方法`GetBalance()`，用于查询指定账户的余额，代码如下：

```go
func (bc *BlockChain) GetBalance(address string, txs []*Transaction) int64 {
    //txOutputs:=bc.UnUTXOs(address)
    unUTXOs := bc.UnUTXOs(address, txs)
    //fmt.Println(address, unUTXOs)
    var amount int64
    for _, utxo := range unUTXOs {
        amount = amount + utxo.Output.Value
    }
    return amount

} 
```

现在可以进行代码测试，在终端输入以下命令：

```go
hanru:day03_05_Transaction ruby$ ./bc getbalance -address 'hanru'

hanru:day03_05_Transaction ruby$ ./bc getbalance -address 'wangergou' 
```

运行效果如下：

![`img.kongyixueyuan.com/0611_%E4%BD%99%E9%A2%9D.png`](img/33d46381bd1681cce0b22d1549be8dcd.jpg)

接下来我们进行多笔交易测试，在终端输入以下命令：

```go
hanru:day03_05_Transaction ruby$ ./bc send -from '["hanru","wangergou"]' -to '["ruby","lixiaohua"]' -amount '["3","3"]' 
```

运行效果如下：

![`img.kongyixueyuan.com/0612_%E5%A4%9A%E7%AC%94.png`](img/3e142937fc302e7da4c995b010875c2a.jpg)

查看区块信息，在终端输入以下命令：

```go
hanru:day03_05_Transaction ruby$ ./bc printchain 
```

![`img.kongyixueyuan.com/0613.png`](img/baa169c4b8d831303c9ff555c12c1123.jpg)

接下来查询余额，在终端输入以下命令：

```go
hanru:day03_05_Transaction ruby$ ./bc getbalance -address 'hanru'
hanru:day03_05_Transaction ruby$ ./bc getbalance -address 'wangergou'
hanru:day03_05_Transaction ruby$ ./bc getbalance -address 'ruby'
hanru:day03_05_Transaction ruby$ ./bc getbalance -address 'lixiaohua' 
```

运行结果如下：

![`img.kongyixueyuan.com/0614.png`](img/2fa2aee28d670098c389775ce0e8f36c.jpg)

### 4.12 优化分离

现在我们将所有的功能都写在了 CLI.go 文件中，为了优化程序我们可以将功能拆解出来放在单独的 go 文件中。

首先在 BLC 包下，创建`CLI_createBlockChain.go`文件，并将`CLI.go`中的`createGenesisBlockchain()`方法，剪切过来，代码效果如下:

```go
package BLC

func (cli *CLI) createGenesisBlockchain(address string){
    //fmt.Println(data)
    CreateBlockChainWithGenesisBlock(address)

} 
```

接下来在 BLC 包下，继续创建 go 文件。创建`CLI_send.go`文件，并将`CLI.go`中的`send()`方法，剪切过来，代码效果如下:

```go
package BLC

import (
    "fmt"
    "os"
)

//转账
func (cli *CLI) send(from, to, amount [] string) {
    if !dbExists() {
        fmt.Println("数据库不存在。。。")
        os.Exit(1)
    }
    blockchain := GetBlockchainObject()

    blockchain.MineNewBlock(from, to, amount)
    defer blockchain.DB.Close()
} 
```

继续在 BLC 包下创建 go 文件，创建`CLI_getBalance.go`文件，并将`CLI.go`中的`getBalance()`方法，剪切过来，代码效果如下:

```go
package BLC

import (
    "fmt"
    "os"
)

//查询余额
func (cli *CLI)getBalance(address string){
    fmt.Println("查询余额：",address)
    bc := GetBlockchainObject()

    if bc == nil{
        fmt.Println("数据库不存在，无法查询。。")
        os.Exit(1)
    }
    defer bc.DB.Close()
    //txOutputs:= bc.UnUTXOs(address)
    //for i,out:=range txOutputs{
    //    fmt.Println(i,"---->",out)
    //}
    balance:=bc.GetBalance(address,[]*Transaction{})
    fmt.Printf("%s,一共有%d 个 Token\n",address,balance)
} 
```

在 BLC 包下，继续创建 go 文件。创建`CLI_printChains.go`文件，并将`CLI.go`中的`printChains()`方法，剪切过来，代码效果如下:

```go
package BLC

import (
    "fmt"
    "os"
)

func (cli *CLI)printChains(){
    bc:=GetBlockchainObject()
    if bc == nil{
        fmt.Println("没有区块可以打印。。")
        os.Exit(1)
    }
    defer bc.DB.Close()
    bc.PrintChains()
} 
```

## 5\. 总结

通过本章节的学习，我们知道了什么是转账交易的输入，输出，以及转账的原理。为了能够实现转账交易和查询余额，我们需要学习 UTXO 模型，以及统计未花费的 Output 的算法。

虽然不容易，但是现在终于实现交易了！不过，我们依然缺少了一些像比特币那样的一些关键特性：

1.  地址（address）。我们还没有基于私钥（private key）的真实地址。
2.  奖励（reward）。现在挖矿是肯定无法盈利的！
3.  UTXO 集。获取余额需要扫描整个区块链，而当区块非常多的时候，这么做就会花费很长时间。并且，如果我们想要验证后续交易，也需要花费很长时间。而 UTXO 集就是为了解决这些问题，加快交易相关的操作。
4.  内存池（mempool）。在交易被打包到块之前，这些交易被存储在内存池里面。在我们目前的实现中，一个块仅仅包含一笔交易，这是相当低效的。

[项目源代码](https://github.com/rubyhan1314/PublicChain)