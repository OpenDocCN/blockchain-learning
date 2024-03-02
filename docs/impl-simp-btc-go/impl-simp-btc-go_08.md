# 第八章 数字签名(digital signature)

# 数字签名(digital signature)

在数学和密码学中，有一个数字签名（digital signature）的概念，算法可以保证：

1.  当数据从发送方传送到接收方时，数据不会被修改；
2.  数据由某一确定的发送方创建；
3.  发送方无法否认发送过数据这一事实。

通过在数据上应用签名算法（也就是对数据进行签名），你就可以得到一个签名，这个签名晚些时候会被验证。生成数字签名需要一个私钥，而验证签名需要一个公钥。签名有点类似于印章，比方说我做了一幅画，完了用印章一盖，就说明了这幅画是我的作品。给数据生成签名，就是给数据盖了章。

## 1\. 课程目标

1.  知道什么是数字签名

2.  知道为什么要进行签名和验签

3.  学会如何进行签名

4.  学会在何处使用签名

## 2\. 项目代码及效果展示

### 2.1 项目代码结构

![`img.kongyixueyuan.com/0801_%E9%A1%B9%E7%9B%AE%E7%BB%93%E6%9E%84.png`](img/1d56d5bb426b734afc51edd6cbef9207.jpg)

### 2.2 项目运行结果

![`img.kongyixueyuan.com/0802_%E8%BF%90%E8%A1%8C%E6%95%88%E6%9E%9C.gif`](img/462dcb16782b0057f23dea127ef026eb.jpg)

其实运行效果和之前的没有区别，因为我们并没有增加新的功能，只是在创建交易的时候，添加了数字签名，在创建新区块的时候，进行签名验证。

## 3\. 创建项目

### 3.1 创建工程

首先打开 Goland 开发工具

打开工程：`mypublicchain`

创建项目：将上一次的项目代码，`day04_06_Address`，复制为`day05_07_signature`

> 说明：我们每一章节的项目代码，都是在上一个章节上进行添加。所以拷贝上一次的项目代码，然后进行新内容的添加或修改。

### 3.2 代码实现

#### 3.2.1 修改`BlockChain.go`文件

打开`day05_07_signature`目录里的 BLC 包，修改`BlockChain.go`文件。

修改步骤：

```go
修改步骤：
step1：添加 FindTransactionByTxID()方法，查找某个交易中 Input 所引用的 Output 所在的交易
step2：添加 SignTransaction()方法
step3：添加 VerifyTransaction()方法
step4：修改 MineNewBlock()方法 
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
    "crypto/ecdsa"
    "bytes"
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
                fmt.Printf("\t\t\tPublicKey:%v\n", in.PublicKey)
            }
            fmt.Println("\t\tVouts:")
            for _, out := range tx.Vouts {
                fmt.Printf("\t\t\tvalue:%d\n", out.Value)
                fmt.Printf("\t\t\tPubKeyHash:%v\n", out.PubKeyHash)
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

    //在建立新区块钱，对 txs 进行签名验证
    _txs :=[]*Transaction{}
    for _, tx := range txs {
        if bc.VerifyTransaction(tx,_txs) !=true{
            log.Panic("签名验证失败。。")
        }
        _txs = append(_txs,tx)
    }

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
    1.遍历数据库，获取每个块中的 Transaction
    2.判断是否被
     */
    var unUTXOs []*UTXO                      //未花费
    spentTxOutputs := make(map[string][]int) //存储已经花费

    //添加先从 txs 遍历，查找未花费
    //for i, tx := range txs {
    for i:=len(txs)-1;i>=0;i--{
        unUTXOs = caculate(txs[i], address, spentTxOutputs, unUTXOs)
    }

    bcIterator := bc.Iterator()
    for {
        block := bcIterator.Next()
        //统计未花费
        //1.获取 block 中的每个 Transaction
        //for _, tx := range block.Txs {
        //    unUTXOs = caculate(tx, address, spentTxOutputs, unUTXOs)
        //}
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
    fmt.Println(address, unUTXOs)
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
    fmt.Println(from,utxos)
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
            fullPayloadHash:=Base58Decode([]byte(address))
            pubKeyHash:=fullPayloadHash[1:len(fullPayloadHash)-addressChecksumLen]

            if in.UnLockWithAddress(pubKeyHash) {
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

//添加方法
func (bc *BlockChain) SignTransaction(tx *Transaction,privKey ecdsa.PrivateKey,txs []*Transaction){
    if tx.IsCoinbaseTransaction(){
        return
    }
    prevTxs:=make(map[string]*Transaction)
    for _,vin :=range tx.Vins{
        prevTx:=bc.FindTransactionByTxID(vin.TxID,txs)
        prevTxs[hex.EncodeToString(prevTx.TxID)] = prevTx
    }

    tx.Sign(privKey,prevTxs)
}

//根据交易 ID 查找对应的 Transaction
func (bc *BlockChain) FindTransactionByTxID(txID[]byte,txs []*Transaction)*Transaction{
    itertaor:=bc.Iterator()
    //先遍历 txs
    for _,tx:=range txs{
        if bytes.Compare(txID,tx.TxID) ==0{
            return tx
        }
    }

    for{
        block:=itertaor.Next()
        for _,tx:=range block.Txs{
            if bytes.Compare(txID,tx.TxID) == 0{
                return tx
            }
        }

        var hashInt big.Int
        hashInt.SetBytes(block.PrevBlockHash)
        if big.NewInt(0).Cmp(&hashInt) == 0{
            break
        }
    }
    return &Transaction{}
}

//验证数字签名：
func (bc *BlockChain) VerifyTransaction(tx *Transaction,txs []*Transaction)bool{
    prevTXs :=make(map[string] *Transaction)
    for _,vin:=range tx.Vins{
        prevTx := bc.FindTransactionByTxID(vin.TxID,txs)
        prevTXs[hex.EncodeToString(prevTx.TxID)] = prevTx
    }
    return tx.Verify(prevTXs)
} 
```

#### 3.2.2 修改`Transaction.go`文件

打开`day05_07_signature`目录里的 BLC 包，修改`Transaction.go`文件。

修改步骤：

```go
修改步骤：
step1：添加 Sign()方法，用于进行交易签名
step2：添加 TrimmedCopy()方法，用于备份签名和验证时交易的副本。
step3：添加 Serialize()方法，将交易进行序列化
step4：添加 NewTxID()方法，用于设置要签名的数据
step5：添加 Verify()方法，用于签名验证 
```

修改完后代码如下：

```go
package BLC

import (
    "bytes"
    "encoding/gob"
    "log"
    "crypto/sha256"
    "encoding/hex"
    "crypto/ecdsa"
    "crypto/rand"
    "crypto/elliptic"
    "math/big"
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
    txInput := &TXInput{[]byte{}, -1, nil, []byte{}}
    //txOutput := &TXOuput{10, address}
    txOutput := NewTXOuput(10, address)
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

func NewSimpleTransaction(from, to string, amount int64, bc *BlockChain, txs []*Transaction) *Transaction {
    var txInputs [] *TXInput
    var txOutputs [] *TXOuput

    balance, spendableUTXO := bc.FindSpendableUTXOs(from, amount, txs)

    //代表消费

    //txInput := &TXInput{bytes, 0, from}
    //txInputs = append(txInputs, txInput)

    //获取钱包
    wallets := NewWallets()
    wallet := wallets.WalletsMap[from]

    for txID, indexArray := range spendableUTXO {
        txIDBytes, _ := hex.DecodeString(txID)
        for _, index := range indexArray {
            txInput := &TXInput{txIDBytes, index, nil, wallet.PublicKey}
            txInputs = append(txInputs, txInput)
        }
    }

    //转账
    //txOutput1 := &TXOuput{amount, to}
    txOutput1 := NewTXOuput(amount, to)
    txOutputs = append(txOutputs, txOutput1)

    //找零
    //txOutput2 := &TXOuput{10 - amount, from}
    //txOutput2 := &TXOuput{4 - amount, from}
    //txOutput2 := &TXOuput{balance - amount, from}
    txOutput2 := NewTXOuput(balance-amount, from)

    txOutputs = append(txOutputs, txOutput2)

    tx := &Transaction{[]byte{}, txInputs, txOutputs}
    //设置 hash 值
    tx.SetTxID()

    //进行签名
    bc.SignTransaction(tx, wallet.PrivateKey,txs)

    return tx
}

//判断当前交易是否是 Coinbase 交易
func (tx *Transaction) IsCoinbaseTransaction() bool {
    return len(tx.Vins[0].TxID) == 0 && tx.Vins[0].Vout == -1
}

//签名
//正如上面提到的，为了对一笔交易进行签名，我们需要获取交易输入所引用的输出，因为我们需要存储这些输出的交易。
func (tx *Transaction) Sign(privKey ecdsa.PrivateKey, prevTXs map[string]*Transaction) {
    //1.如果时 coinbase 交易，无需签名
    if tx.IsCoinbaseTransaction() {
        return
    }

    //2.input 没有对应的 transaction,无法签名
    for _, vin := range tx.Vins {
        if prevTXs[hex.EncodeToString(vin.TxID)].TxID == nil {
            log.Panic("当前的 input 没有对应的 transaction")
        }
    }

    //3.获取 Transaction 的部分数据的副本
    txCopy:=tx.TrimmedCopy()

    //4.
    for index,input:=range txCopy.Vins{
        prevTx := prevTXs[hex.EncodeToString(input.TxID)]
        //为 txCopy 设置新的交易 ID：txID->[]byte{},Vout,sign-->nil, publlicKey-->对应输出的公钥哈希
        input.Signature = nil//双保险
        input.PublicKey = prevTx.Vouts[input.Vout].PubKeyHash//设置 input 的公钥为对应输出的公钥哈希
        data := txCopy.getData()//设置新的 txID

        input.PublicKey = nil//再将 publicKey 置为 nil

        //签名
        /*
        通过 privKey 对 txCopy.ID 进行签名。
        一个 ECDSA 签名就是一对数字，我们对这对数字连接起来，并存储在输入的 Signature 字段。
         */
        r,s,err := ecdsa.Sign(rand.Reader,&privKey,data)
        if err != nil{
            log.Panic(err)
        }
        signature:=append(r.Bytes(),s.Bytes()...)
        tx.Vins[index].Signature = signature

    }
}

//获取签名所需要的 Transaction 的副本
//创建 tx 的副本：需要剪裁数据
/*
TxID，
[]*TxInput,
    TxInput 中，去除 sign，publicKey
[]*TxOutput

这个副本包含了所有的输入和输出，但是 TXInput.Signature 和 TXIput.PubKey 被设置为 nil。
 */
func (tx *Transaction) TrimmedCopy() Transaction {
    var inputs [] *TXInput
    var outputs [] *TXOuput
    for _, input := range tx.Vins {
        inputs = append(inputs, &TXInput{input.TxID, input.Vout, nil, nil})
    }
    for _, output := range tx.Vouts {
        outputs = append(outputs, &TXOuput{output.Value, output.PubKeyHash})
    }
    txCopy := Transaction{tx.TxID, inputs, outputs}
    return txCopy

}

func (tx *Transaction) Serialize() []byte {
    jsonByte,err := json.Marshal(tx)
    if err != nil{
        //fmt.Println("序列化失败:",err)
        log.Panic(err)
    }
    return jsonByte
}

func (tx Transaction)getData()[]byte{
    txCopy :=tx
    txCopy.TxID=[]byte{}
    hash:=sha256.Sum256(txCopy.Serialize())
    return hash[:]
}

//验证数字签名
func (tx *Transaction) Verify(prevTXs map[string]*Transaction)bool{
    if tx.IsCoinbaseTransaction() {
        return true
    }

    //2.input 没有对应的 transaction,无法签名
    for _, vin := range tx.Vins {
        if prevTXs[hex.EncodeToString(vin.TxID)].TxID == nil {
            log.Panic("当前的 input 没有对应的 transaction,无法验证。。")
        }
    }
    txCopy:=tx.TrimmedCopy()

    curve:=elliptic.P256()
    for index,input:=range tx.Vins{
        prevTx:=prevTXs[hex.EncodeToString(input.TxID)]
        txCopy.Vins[index].Signature = nil
        txCopy.Vins[index].PublicKey = prevTx.Vouts[input.Vout].PubKeyHash
        data := txCopy.getData()
        txCopy.Vins[index].PublicKey = nil

        //签名中的 s 和 r
        r:=big.Int{}
        s:=big.Int{}
        sigLen:=len(input.Signature)
        r.SetBytes(input.Signature[:sigLen/2])
        s.SetBytes(input.Signature[sigLen/2:])

        //通过公钥，产生新的 s 和 r，与原来的进行对比
        x:=big.Int{}
        y:=big.Int{}
        keyLen:=len(input.PublicKey)
        x.SetBytes(input.PublicKey[:keyLen/2])
        y.SetBytes(input.PublicKey[keyLen/2:])

        //根据椭圆曲线，以及 x，y 获取公钥
        //我们使用从输入提取的公钥创建了一个 ecdsa.PublicKey
        rawPubKey:=ecdsa.PublicKey{curve,&x,&y}//
        //这里我们解包存储在 TXInput.Signature 和 TXInput.PubKey 中的值，
        // 因为一个签名就是一对数字，一个公钥就是一对坐标。
        // 我们之前为了存储将它们连接在一起，现在我们需要对它们进行解包在 crypto/ecdsa 函数中使用。

        //验证
        //在这里：我们使用从输入提取的公钥创建了一个 ecdsa.PublicKey，通过传入输入中提取的签名执行了 ecdsa.Verify。
        // 如果所有的输入都被验证，返回 true；如果有任何一个验证失败，返回 false.
        if ecdsa.Verify(&rawPubKey,data,&r,&s) ==false{
            //公钥，要验证的数据，签名的 r，s
            return false
        }
    }
    return true

} 
```

#### 3.2.3 `main.go`无需修改

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

## 4\. 数字签名讲解

### 4.1 交易过程

一笔交易就是一个地址的比特币，转移到另一个地址。由于比特币的交易记录全部都是公开的，哪个地址拥有多少比特币，都是可以查到的。因此，支付方是否拥有足够的比特币，完成这笔交易，这是可以轻易验证的。

问题出在怎么防止其他人，冒用你的名义申报交易。举例来说，有人申报了一笔交易：地址 A 向地址 B 支付 10 个比特币。我们怎么知道这个申报是真的，数据是正确的？

比特币协议规定，申报交易的时候，除了交易金额，转出比特币的一方还必须提供以下数据。

*   上一笔交易的 Hash（你从哪里得到这些比特币）
*   本次交易双方的地址
*   支付方的公钥
*   支付方的私钥生成的数字签名

验证这笔交易是否属实，需要三步。

*   第一步，找到上一笔交易，确认支付方的比特币来源。

*   第二步，算出支付方公钥的指纹，确认与支付方的地址一致，从而保证公钥属实。

*   第三步，使用公钥去解开数字签名，保证签名属实、私钥属实。

*   经过上面三步，就可以认定这笔交易是真实的。

![`img.kongyixueyuan.com/0807_%E4%BA%A4%E6%98%93.jpg`](img/bd4fe5796213f683cc78a0a34b9a80ba.jpg)

### 4.1 数字签名

好了，现在我们已经知道了在比特币中证明用户身份的是私钥。而为了保证交易有效性，需要使用数字签名。接下来我们详细谈一下数字签名。

#### 4.1.1 数字签名的概念

所谓数字签名(Digital Signature)（又称公开密钥数字签名、电子签章）。是一种类似写在纸上的普通的物理签名，但是使用了公钥加密领域的技术实现，用于鉴别数字信息的方法。一套数字签名通常定义两种互补的运算，一个用于签名，另一个用于验证。

#### 4.1.2 数字签名如何工作

数字签名由两部分组成：第一部分是使用私钥（签名密钥）从消息（交易）创建签名的算法； 第二部分是允许任何人验证签名的算法。

![`img.kongyixueyuan.com/0804_%E7%AD%BE%E5%90%8D%E6%B5%81%E7%A8%8B.jpg`](img/6e258cf5c0c847131800b48d8c5c5dea.jpg)

#### 4.1.3 数字签名的特性

1、签名不可伪造性；

2、签名不可抵赖的（简直通俗易懂~）；

3、签名可信性，签名的识别和应用相对容易，任何人都可以验证签名的有效性；

4、签名是不可复制的，签名与原文是不可分割的整体；

5、签名消息不可篡改，因为任意比特数据被篡改，其签名便被随之改变，那么任何人可以验证而拒绝接受此签名。

#### 4.1.4 设计数字签名

为了对数据进行签名，我们需要下面两样东西：

1.  要签名的数据
2.  私钥

应用签名算法可以生成一个签名，并且这个签名会被存储在交易输入中。为了对一个签名进行验证，我们需要以下三样东西：

1.  被签名的数据
2.  签名
3.  公钥

简单来说，验证过程可以被描述为：检查签名是由被签名数据加上私钥得来，并且公钥恰好是由该私钥生成。

> 数据签名并不是加密，你无法从一个签名重新构造出数据。这有点像哈希：你在数据上运行一个哈希算法，然后得到一个该数据的唯一表示。签名与哈希的区别在于密钥对：有了密钥对，才有签名验证。但是密钥对也可以被用于加密数据：私钥用于加密，公钥用于解密数据。不过比特币并不使用加密算法。

在比特币中，每一笔交易输入都会由创建交易的人签名。在被放入到一个块之前，必须要对每一笔交易进行验证。除了一些其他步骤，验证意味着：

1.  检查交易输入有权使用来自之前交易的输出
2.  检查交易签名是正确的

如图，对数据进行签名和对签名进行验证的过程大致如下：

![`img.kongyixueyuan.com/0803_%E6%95%B0%E5%AD%97%E7%AD%BE%E5%90%8D.png`](img/f8072a79930d99ecb659f6f0939e6ba7.jpg)

现在来回顾一个交易完整的生命周期：

1.  起初，创世块里面包含了一个 coinbase 交易。在 coinbase 交易中，没有输入，所以也就不需要签名。coinbase 交易的输出包含了一个哈希过的公钥（使用的是 **RIPEMD16(SHA256(PubKey))** 算法）
2.  当一个人发送币时，就会创建一笔交易。这笔交易的输入会引用之前交易的输出。每个输入会存储一个公钥（没有被哈希）和整个交易的一个签名。
3.  比特币网络中接收到交易的其他节点会对该交易进行验证。除了一些其他事情，他们还会检查：在一个输入中，公钥哈希与所引用的输出哈希相匹配（这保证了发送方只能花费属于自己的币）；签名是正确的（这保证了交易是由币的实际拥有者所创建）。
4.  当一个矿工准备挖一个新块时，他会将交易放到块中，然后开始挖矿。
5.  当新块被挖出来以后，网络中的所有其他节点会接收到一条消息，告诉其他人这个块已经被挖出并被加入到区块链。
6.  当一个块被加入到区块链以后，交易就算完成，它的输出就可以在新的交易中被引用。

### 4.2 实现签名

交易必须被签名，因为这是比特币里面保证发送方不会花费属于其他人的币的唯一方式。如果一个签名是无效的，那么这笔交易就会被认为是无效的，因此，这笔交易也就无法被加到区块链中。

我们现在离实现交易签名还差一件事情：用于签名的数据。一笔交易的哪些部分需要签名？又或者说，要对完整的交易进行签名？选择签名的数据相当重要。因为用于签名的这个数据，必须要包含能够唯一识别数据的信息。比如，如果仅仅对输出值进行签名并没有什么意义，因为签名不会考虑发送方和接收方。

考虑到交易解锁的是之前的输出，然后重新分配里面的价值，并锁定新的输出，那么必须要签名以下数据：

1.  存储在已解锁输出的公钥哈希。它识别了一笔交易的“发送方”。
2.  存储在新的锁定输出里面的公钥哈希。它识别了一笔交易的“接收方”。
3.  新的输出值。

> 在比特币中，锁定/解锁逻辑被存储在脚本中，它们被分别存储在输入和输出的 `ScriptSig` 和 `ScriptPubKey` 字段。由于比特币允许这样不同类型的脚本，它对 `ScriptPubKey` 的整个内容进行了签名。

可以看到，我们不需要对存储在输入里面的公钥签名。因此，在比特币里， 所签名的并不是一个交易，而是一个去除部分内容的输入副本，输入里面存储了被引用输出的 `ScriptPubKey` 。

看着有点复杂，来开始写代码吧。先从修改`TxInput`和`TxOutput`的结构开始：

在 BLC 包下，修改`Transaction_TxInput.go`文件，修改后代码如下：

```go
type TXInput struct {
    //1.交易的 ID
    TxID [] byte
    //2.存储 Txoutput 的 vout 里面的索引
    Vout int
    //3.用户名
    //ScriptSiq string
    Signature [] byte //数字签名
    PublicKey [] byte //公钥，钱包里面
}

//判断当前 txInput 消费，和指定的 address 是否一致
func (txInput *TXInput) UnLockWithAddress(pubKeyHash []byte) bool{
    //return txInput.ScriptSiq == address
    publicKey:=PubKeyHash(txInput.PublicKey)
    return bytes.Compare(pubKeyHash,publicKey) == 0
} 
```

在这里，我们需要重新设置两个字段，`Signature`表示数字签名，`PublicKey`表示原始公钥(就是钱包里面的公钥)。

接下来，我们修改`Transaction_TxOutput.go`文件，修改后代码如下：

```go
//交易的输出，就是币实际存储的地方
type TXOuput struct {
    Value int64
    //一个锁定脚本(ScriptPubKey)，要花这笔钱，必须要解锁该脚本。
    //ScriptPubKey string //公钥：先理解为，用户名
    PubKeyHash [] byte // 公钥
}

//判断当前 txOutput 消费，和指定的 address 是否一致
func (txOutput *TXOuput) UnLockWithAddress(address string) bool {
    //return txOutput.ScriptPubKey == address
    fullPayloadHash := Base58Decode([]byte(address))
    pubKeyHash := fullPayloadHash[1:len(fullPayloadHash)-4]
    return bytes.Compare(txOutput.PubKeyHash, pubKeyHash) == 0
} 
```

因为在给`TxInput`设置签名需要用到该`TxInput`对应的`TxOutput`的数据，所以要找到这个`TxOutput`所在的`Transaction`。现在我们修改`BlockChain.go`文件，添加一个方法`FindTransactionByTxID()`：

```go
 //根据交易 ID 查找对应的 Transaction
func (bc *BlockChain) FindTransactionByTxID(txID[]byte,txs []*Transaction)*Transaction{
    itertaor:=bc.Iterator()
    //先遍历 txs
    for _,tx:=range txs{
        if bytes.Compare(txID,tx.TxID) ==0{
            return tx
        }
    }

    for{
        block:=itertaor.Next()
        for _,tx:=range block.Txs{
            if bytes.Compare(txID,tx.TxID) == 0{
                return tx
            }
        }

        var hashInt big.Int
        hashInt.SetBytes(block.PrevBlockHash)
        if big.NewInt(0).Cmp(&hashInt) == 0{
            break
        }
    }
    return &Transaction{}
} 
```

在查找的时候，因为可能存在一次转账多笔交易的情况，所以先找未打包的`Transaction`，然后再查找每个区块中的`Transaction`。

接下来在`BlockChain.go`文件中，继续添加方法，表示签名一笔交易：

```go
//添加方法
func (bc *BlockChain) SignTransaction(tx *Transaction,privKey ecdsa.PrivateKey,txs []*Transaction){
    if tx.IsCoinbaseTransaction(){
        return
    }
    prevTxs:=make(map[string]*Transaction)
    for _,vin :=range tx.Vins{
        prevTx:=bc.FindTransactionByTxID(vin.TxID,txs)
        prevTxs[hex.EncodeToString(prevTx.TxID)] = prevTx
    }

    tx.Sign(privKey,prevTxs)
} 
```

其实签名交易，就是给交易中的每个`TxInput`，设置`Signature`字段。所以接下来在`Transaction.go`文件中，添加签名方法，代码如下：

```go
 //签名
//正如上面提到的，为了对一笔交易进行签名，我们需要获取交易输入所引用的输出，因为我们需要存储这些输出的交易。
func (tx *Transaction) Sign(privKey ecdsa.PrivateKey, prevTXs map[string]*Transaction) {
    //1.如果时 coinbase 交易，无需签名
    if tx.IsCoinbaseTransaction() {
        return
    }

    //2.input 没有对应的 transaction,无法签名
    for _, vin := range tx.Vins {
        if prevTXs[hex.EncodeToString(vin.TxID)].TxID == nil {
            log.Panic("当前的 input 没有对应的 transaction")
        }
    }

    //3.获取 Transaction 的部分数据的副本
    txCopy:=tx.TrimmedCopy()

    //4.
    for index,input:=range txCopy.Vins{
        prevTx := prevTXs[hex.EncodeToString(input.TxID)]
        //为 txCopy 设置新的交易 ID：txID->[]byte{},Vout,sign-->nil, publlicKey-->对应输出的公钥哈希
        input.Signature = nil//双保险
        input.PublicKey = prevTx.Vouts[input.Vout].PubKeyHash//设置 input 的公钥为对应输出的公钥哈希
        data := txCopy.getData()//设置新的 txID

        input.PublicKey = nil//再将 publicKey 置为 nil

        //txCopy：txID：新 id，Vout，sign-->nil,publicKey-->nil

        //签名
        /*
        通过 privKey 对 txCopy.ID 进行签名。
        一个 ECDSA 签名就是一对数字，我们对这对数字连接起来，并存储在输入的 Signature 字段。
         */
        r,s,err := ecdsa.Sign(rand.Reader,&privKey,data)
        if err != nil{
            log.Panic(err)
        }
        signature:=append(r.Bytes(),s.Bytes()...)
        tx.Vins[index].Signature = signature

    }
} 
```

这个方法接受一个私钥和一个之前交易的 `map`。正如上面提到的，为了对一笔交易进行签名，我们需要获取交易输入所引用的输出，因为我们需要存储这些输出的交易。

这个方法，是签名的核心方法，我们来一步一步地分析该方法：

```go
if tx.IsCoinbaseTransaction() {
        return
    } 
```

`coinbase` 交易因为没有实际输入，所以没有被签名。

```go
//3.获取 Transaction 的部分数据的副本
txCopy:=tx.TrimmedCopy() 
```

将会被签署的是修剪后的交易副本，而不是一个完整交易，接下来添加一个方法，用于拷贝一个交易，代码如下：

```go
//获取签名所需要的 Transaction 的副本
//创建 tx 的副本：需要剪裁数据
/*
TxID，
[]*TxInput,
    TxInput 中，去除 sign，publicKey
[]*TxOutput

这个副本包含了所有的输入和输出，但是 TXInput.Signature 和 TXIput.PubKey 被设置为 nil。
 */
func (tx *Transaction) TrimmedCopy() Transaction {
    var inputs [] *TXInput
    var outputs [] *TXOuput
    for _, input := range tx.Vins {
        inputs = append(inputs, &TXInput{input.TxID, input.Vout, nil, nil})
    }
    for _, output := range tx.Vouts {
        outputs = append(outputs, &TXOuput{output.Value, output.PubKeyHash})
    }
    txCopy := Transaction{tx.TxID, inputs, outputs}
    return txCopy

} 
```

这个副本包含了所有的输入和输出，但是 `TXInput.Signature` 和 `TXIput.PubKey` 被设置为 `nil`。

接下来，我们会迭代副本中每一个输入：

```go
for index,input:=range txCopy.Vins{
        prevTx := prevTXs[hex.EncodeToString(input.TxID)]
        //为 txCopy 设置新的交易 ID：txID->[]byte{},Vout,sign-->nil, publlicKey-->对应输出的公钥哈希
        input.Signature = nil//双保险
        input.PublicKey = prevTx.Vouts[input.Vout].PubKeyHash//设置 input 的公钥为对应输出的公钥哈希
        data := txCopy.getData()//设置新的 txID

        input.PublicKey = nil//再将 publicKey 置为 nil

        ...

    } 
```

在每个输入中，`Signature` 被设置为 `nil` (仅仅是一个双重检验)，`PublicKey` 被设置为所引用输出的 `PubKeyHash`。现在，除了当前交易，其他所有交易都是“空的”，也就是说他们的 `Signature` 和 `PublicKey` 字段被设置为 `nil`。因此，**输入是被分开签名的**，尽管这对于我们的应用并不十分紧要，但是比特币允许交易包含引用了不同地址的输入。

```go
data := txCopy.getData()//设置新的 txID
input.PublicKey = nil//再将 publicKey 置为 nil 
```

`getData()` 方法对交易进行序列化，并使用 SHA-256 算法进行哈希。哈希后的结果就是我们要签名的数据。在获取完哈希，我们应该重置 `PublicKey` 字段，以便于它不会影响后面的迭代。

现在，关键点：

```go
//签名
/*
通过 privKey 对 txCopy.ID 进行签名。
一个 ECDSA 签名就是一对数字，我们对这对数字连接起来，并存储在输入的 Signature 字段。
*/
r,s,err := ecdsa.Sign(rand.Reader,&privKey,data)
if err != nil{
    log.Panic(err)
}
signature:=append(r.Bytes(),s.Bytes()...)
tx.Vins[index].Signature = signature 
```

我们通过 `privKey` 对 data 进行签名。一个 ECDSA 签名就是一对数字，我们对这对数字连接起来，并存储在输入的 `Signature` 字段。

### 4.3 签名验证

现在，验证函数：

在`Transaction.go`文件中，添加验证签名方法，代码如下:

```go
 //验证数字签名
func (tx *Transaction) Verify(prevTXs map[string]*Transaction)bool{
    if tx.IsCoinbaseTransaction() {
        return true
    }

    //2.input 没有对应的 transaction,无法签名
    for _, vin := range tx.Vins {
        if prevTXs[hex.EncodeToString(vin.TxID)].TxID == nil {
            log.Panic("当前的 input 没有对应的 transaction,无法验证。。")
        }
    }
    txCopy:=tx.TrimmedCopy()

    curve:=elliptic.P256()
    for index,input:=range tx.Vins{
        prevTx:=prevTXs[hex.EncodeToString(input.TxID)]
        txCopy.Vins[index].Signature = nil
        txCopy.Vins[index].PublicKey = prevTx.Vouts[input.Vout].PubKeyHash
        data := txCopy.getData()
        txCopy.Vins[index].PublicKey = nil

        //签名中的 s 和 r
        r:=big.Int{}
        s:=big.Int{}
        sigLen:=len(input.Signature)
        r.SetBytes(input.Signature[:sigLen/2])
        s.SetBytes(input.Signature[sigLen/2:])

        //通过公钥，产生新的 s 和 r，与原来的进行对比
        x:=big.Int{}
        y:=big.Int{}
        keyLen:=len(input.PublicKey)
        x.SetBytes(input.PublicKey[:keyLen/2])
        y.SetBytes(input.PublicKey[keyLen/2:])

        //根据椭圆曲线，以及 x，y 获取公钥
        //我们使用从输入提取的公钥创建了一个 ecdsa.PublicKey
        rawPubKey:=ecdsa.PublicKey{curve,&x,&y}//
        //这里我们解包存储在 TXInput.Signature 和 TXInput.PubKey 中的值，
        // 因为一个签名就是一对数字，一个公钥就是一对坐标。
        // 我们之前为了存储将它们连接在一起，现在我们需要对它们进行解包在 crypto/ecdsa 函数中使用。

        //验证
        //在这里：我们使用从输入提取的公钥创建了一个 ecdsa.PublicKey，通过传入输入中提取的签名执行了 ecdsa.Verify。
        // 如果所有的输入都被验证，返回 true；如果有任何一个验证失败，返回 false.
        if ecdsa.Verify(&rawPubKey,data,&r,&s) ==false{
            //公钥，要验证的数据，签名的 r，s
            return false
        }
    }
    return true

} 
```

这个方法十分直观。首先，我们需要同一笔交易的副本：

```go
txCopy:=tx.TrimmedCopy() 
```

然后，我们需要相同的区块用于生成密钥对：

```go
curve:=elliptic.P256() 
```

接下来，我们检查每个输入中的签名：

```go
for index,input:=range tx.Vins{
        prevTx:=prevTXs[hex.EncodeToString(input.TxID)]
        txCopy.Vins[index].Signature = nil
        txCopy.Vins[index].PublicKey = prevTx.Vouts[input.Vout].PubKeyHash
        data := txCopy.getData()
        txCopy.Vins[index].PublicKey = nil 
```

这个部分跟 `Sign` 方法一模一样，因为在验证阶段，我们需要的是与签名相同的数据。

```go
//签名中的 s 和 r
r:=big.Int{}
s:=big.Int{}
sigLen:=len(input.Signature)
r.SetBytes(input.Signature[:sigLen/2])
s.SetBytes(input.Signature[sigLen/2:])

//通过公钥，产生新的 s 和 r，与原来的进行对比
x:=big.Int{}
y:=big.Int{}
keyLen:=len(input.PublicKey)
x.SetBytes(input.PublicKey[:keyLen/2])
y.SetBytes(input.PublicKey[keyLen/2:]) 
```

这里我们解包存储在 `TXInput.Signature` 和 `TXInput.PublicKey` 中的值，因为一个签名就是一对数字，一个公钥就是一对坐标。我们之前为了存储将它们连接在一起，现在我们需要对它们进行解包在 `crypto/ecdsa` 函数中使用。

```go
//根据椭圆曲线，以及 x，y 获取公钥
//我们使用从输入提取的公钥创建了一个 ecdsa.PublicKey
rawPubKey:=ecdsa.PublicKey{curve,&x,&y}//
        //这里我们解包存储在 TXInput.Signature 和 TXInput.PubKey 中的值，
        // 因为一个签名就是一对数字，一个公钥就是一对坐标。
        // 我们之前为了存储将它们连接在一起，现在我们需要对它们进行解包在 crypto/ecdsa 函数中使用。

        //验证
        //在这里：我们使用从输入提取的公钥创建了一个 ecdsa.PublicKey，通过传入输入中提取的签名执行了 ecdsa.Verify。
        // 如果所有的输入都被验证，返回 true；如果有任何一个验证失败，返回 false.
        if ecdsa.Verify(&rawPubKey,data,&r,&s) ==false{
            //公钥，要验证的数据，签名的 r，s
            return false
        }
    }
    return true

} 
```

在这里：我们使用从输入提取的公钥创建了一个 `ecdsa.PublicKey`，通过传入输入中提取的签名执行了 `ecdsa.Verify`。如果所有的输入都被验证，返回 `true`；如果有任何一个验证失败，返回 `false`.

在一笔交易被放入一个块之前进行验证，所以接下来我们需要修改`BlockChain.go`文件中的`MineNewBlock()`方法，

```go
 //挖掘新的区块
func (bc *BlockChain) MineNewBlock(from, to, amount []string) {

    var txs []*Transaction
    for i := 0; i < len(from); i++ {

        amountInt, _ := strconv.ParseInt(amount[i], 10, 64)
        tx := NewSimpleTransaction(from[i], to[i], amountInt, bc, txs)

        txs = append(txs, tx)
    }

    ...

    //在建立新区块钱，对 txs 进行签名验证
    _txs :=[]*Transaction{}
    for _, tx := range txs {
        if bc.VerifyTransaction(tx,_txs) !=true{
            log.Panic("签名验证失败。。")
        }
        _txs = append(_txs,tx)
    }

    ...

} 
```

至此，我们已经在项目中，添加了交易的签名和签名验证。

`main.go`文件无需修改，接下来我们测试一下代码：还是进行创建钱包地址，创建创世区块，然后转账交易等。

在终端中输入以下命令:

```go
hanru:day05_07_signature ruby$ go build -o bc main.go
hanru:day05_07_signature ruby$ ./bc
hanru:day05_07_signature ruby$ ./bc createwallet
hanru:day05_07_signature ruby$ ./bc createwallet
hanru:day05_07_signature ruby$ ./bc createwallet
hanru:day05_07_signature ruby$ ./bc createblockchain -address '12zRREDSkRR1Kx86hGqhWnv7GjSnrjHdgH'
hanru:day05_07_signature ruby$ ./bc send -from '["12zRREDSkRR1Kx86hGqhWnv7GjSnrjHdgH"]' -to '["1JLAFY1uD88Ljtd6C4nA9LXxqi9Gtg63bE"]' -amount '["4"]' 
```

运行结果如下：

![`img.kongyixueyuan.com/0805_%E8%BF%90%E8%A1%8C%E7%BB%93%E6%9E%9C1.png`](img/40a582867938a585a22252c56a0eab60.jpg)

继续输入以下命令：

```go
hanru:day05_07_signature ruby$ ./bc getbalance -address '12zRREDSkRR1Kx86hGqhWnv7GjSnrjHdgH'
hanru:day05_07_signature ruby$ ./bc getbalance -address '1JLAFY1uD88Ljtd6C4nA9LXxqi9Gtg63bE'
hanru:day05_07_signature ruby$ ./bc send -from '["12zRREDSkRR1Kx86hGqhWnv7GjSnrjHdgH","1JLAFY1uD88Ljtd6C4nA9LXxqi9Gtg63bE"]' -to '["1Q7cBoaB44E5D8421et9pXYbbVLEMQpaGq","1Q7cBoaB44E5D8421et9pXYbbVLEMQpaGq"]' -amount '["3","2"]'
hanru:day05_07_signature ruby$ ./bc getbalance -address '12zRREDSkRR1Kx86hGqhWnv7GjSnrjHdgH'
hanru:day05_07_signature ruby$ ./bc getbalance -address '1JLAFY1uD88Ljtd6C4nA9LXxqi9Gtg63bE'
hanru:day05_07_signature ruby$ ./bc getbalance -address '1Q7cBoaB44E5D8421et9pXYbbVLEMQpaGq' 
```

运行结果如下:

![`img.kongyixueyuan.com/0806_%E8%BF%90%E8%A1%8C%E7%BB%93%E6%9E%9C2.png`](img/e3461297053f20fe7d770d7febf60d16.jpg)

## 5\. 总结

通过本章节的学习，我们知道了什么是签名，为何签名，以及如何签名。只有转账人才能生成的一段防伪造的字符串。通过验证该字符串，一方面证明该交易是转出方本人发起的，另一方面证明交易信息在传输过程中没有被更改。数字签名由：数字摘要和非对称加密技术组成。数字摘要把交易信息 hash 成固定长度的字符串，再用私钥对 hash 后的交易信息进行加密形成数字签名。交易中，需要将完整的交易信息和数字签名一起广播给矿工。矿工节点用转账人公钥对签名验证，验证成功说明该交易确实是转账人发起的；矿工节点将交易信息进行 hash 后与签名中的交易信息摘要进行比对，如果一致，则说明交易信息在传输过程中没有被篡改。

[项目源代码](https://github.com/rubyhan1314/PublicChain)