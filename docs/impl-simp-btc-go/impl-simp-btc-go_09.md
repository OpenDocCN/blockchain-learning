# 第九章 UTXO 集优化

# 交易(Transactions)（2）

在这个系列文章的一开始，我们就提到了，区块链是一个分布式数据库。不过在之前的文章中，我们选择性地跳过了“分布式”这个部分，而是将注意力都放到了“数据库”部分。到目前为止，我们几乎已经实现了一个区块链数据库的所有元素(基本原型，工作量证明，持久化，命令行接口，交易，地址，数字签名等)。今天，我们将会分析之前跳过的一些机制。而在下一篇文章中，我们将会开始讨论区块链的分布式特性。

> 本文的代码实现变化很大。

## 1\. 课程目标

1.  知道什么是奖励
2.  知道什么是 UTXO 集
3.  学会 UTXO 集优化的原理
4.  项目中修改代码实现 UTXO 集查询余额
5.  项目中修改代码实现 UTXO 集进行转账交易

## 2\. 项目代码及效果展示

### 2.1 项目代码结构

![`img.kongyixueyuan.com/0901_%E9%A1%B9%E7%9B%AE%E7%BB%93%E6%9E%84.png`](img/ded62200155000c9c81d48d1533ed9aa.jpg)

### 2.2 项目运行结果

![`img.kongyixueyuan.com/0902_%E8%BF%90%E8%A1%8C%E6%95%88%E6%9E%9C.gif`](img/ebd8ab6684db172df76f338e579ccc27.jpg)

## 3\. 创建项目

### 3.1 创建工程

首先打开 Goland 开发工具

打开工程：`mypublicchain`

创建项目：将上一次的项目代码，`day05_07_signature`，复制为`day06_08_Update`

> 说明：我们每一章节的项目代码，都是在上一个章节上进行添加。所以拷贝上一次的项目代码，然后进行新内容的添加或修改。

### 3.2 代码实现

#### 3.2.1 创建 go 文件：`TxOutputs.go`

打开`day06_08_Update`目录里的 BLC 包，创建`TxOutputs.go`文件。在`TxOutputs.go`文件中编写代码如下：

```go
package BLC

import (
    "bytes"
    "encoding/gob"
    "log"
)

type TxOutputs struct {
    UTXOS []*UTXO
}

//序列化
func (outs *TxOutputs) Serilalize()[]byte{
    //1.创建一个 buffer
    var result bytes.Buffer
    //2.创建一个编码器
    encoder := gob.NewEncoder(&result)
    //3.编码--->打包
    err := encoder.Encode(outs)
    if err != nil {
        log.Panic(err)
    }
    return result.Bytes()
}

//反序列化
func DeserializeTXOutputs(txOutputsBytes [] byte) *TxOutputs {
    var txOutputs TxOutputs
    var reader = bytes.NewReader(txOutputsBytes)
    //1.创建一个解码器
    decoder := gob.NewDecoder(reader)
    //解包
    err := decoder.Decode(&txOutputs)
    if err != nil {
        log.Panic(err)
    }
    return &txOutputs
} 
```

#### 3.2.2 创建`UTXO_Set.go`文件

打开`day06_08_Update`目录里的 BLC 包，创建`UTXO_Set.go`文件。

添加代码如下：

```go
package BLC

import (
    "github.com/boltdb/bolt"
    "log"
    "encoding/hex"
    "fmt"
    "os"
    "bytes"
)

type UTXOSet struct {
    BlockChain *BlockChain
}

const utxoTableName = "utxoTable"

//重置数据库表
func (utxoSet *UTXOSet) ResetUTXOSet() {
    err := utxoSet.BlockChain.DB.Update(func(tx *bolt.Tx) error {
        b := tx.Bucket([]byte(utxoTableName))
        if b != nil {
            err := tx.DeleteBucket([]byte(utxoTableName))
            if err != nil {
                log.Panic("重置中，删除表失败。。")
            }

        }
        b, err := tx.CreateBucket([]byte(utxoTableName))
        if err != nil {
            log.Panic("重置中，创建新表失败。。")
        }
        if b != nil {
            txOutputMap := utxoSet.BlockChain.FindUnSpentOutputMap()
            //fmt.Println("未花费 outputmap：",txOutputMap)
            for txIDStr, outputs := range txOutputMap {
                txID, _ := hex.DecodeString(txIDStr)
                b.Put(txID, outputs.Serilalize())
            }
        }

        return nil
    })
    if err != nil {
        log.Panic(err)
    }
}

func (utxoSet *UTXOSet) GetBalance(address string) int64 {
    utxos := utxoSet.FindUnspentOutputsForAddress(address)
    var amount int64
    for _, utxo := range utxos {
        amount += utxo.Output.Value
        fmt.Println(address,"余额：",utxo.Output.Value)
    }
    //fmt.Printf("%s 账户，有%d 个 Token\n",address,amount)
    return amount

}

func (utxoSet *UTXOSet) FindUnspentOutputsForAddress(address string) []*UTXO {
    var utxos [] *UTXO
    //查询数据，遍历所有的未花费
    err := utxoSet.BlockChain.DB.View(func(tx *bolt.Tx) error {
        b := tx.Bucket([]byte(utxoTableName))
        if b != nil {
            c := b.Cursor()
            for k, v := c.First(); k != nil; k, v = c.Next() {
                //fmt.Printf("key=%s,value=%v\n", k, v)
                txOutputs := DeserializeTXOutputs(v)
                for _, utxo := range txOutputs.UTXOS {
                    if utxo.Output.UnLockWithAddress(address) {
                        utxos = append(utxos, utxo)
                    }
                }
            }
        }

        return nil
    })
    if err != nil {
        log.Panic(err)
    }
    return utxos
}

//添加一个方法：用于查询给定地址下的，要转账使用的可以使用的 utxo
func (utxoSet *UTXOSet) FindSpendableUTXOs(from string, amount int64, txs [] *Transaction) (int64, map[string][]int) {
    spentableUTXO := make(map[string][]int)
    var total int64 = 0
    //1.找出未打包的 Transaction 中未花费的
    unPackageUTXOs := utxoSet.FindUnPackageSpentableUTXOs(from, txs)
    for _, utxo := range unPackageUTXOs {
        total += utxo.Output.Value
        txIDStr := hex.EncodeToString(utxo.TxID)
        spentableUTXO[txIDStr] = append(spentableUTXO[txIDStr], utxo.Index)
        fmt.Println(amount,",未打包，转账花费：",utxo.Output.Value)
        if total >= amount {
            return total, spentableUTXO
        }
    }

    //钱不够
    //2.找出已经存在数据库中的未花费的
    err := utxoSet.BlockChain.DB.View(func(tx *bolt.Tx) error {
        b := tx.Bucket([]byte(utxoTableName))
        if b != nil {
            c := b.Cursor()
        dbLoop:
            for k, v := c.First(); k != nil; k, v = c.Next() {
                txOutpus := DeserializeTXOutputs(v)
                for _, utxo := range txOutpus.UTXOS {
                    if utxo.Output.UnLockWithAddress(from) {
                        total += utxo.Output.Value
                        txIDStr := hex.EncodeToString(utxo.TxID)
                        spentableUTXO[txIDStr] = append(spentableUTXO[txIDStr], utxo.Index)
                        fmt.Println(amount,",数据库，转账花费：",utxo.Output.Value)
                        if total >= amount {
                            break dbLoop
                        }
                    }

                }
            }
        }
        return nil
    })
    if err != nil {
        log.Panic(err)
    }

    if total < amount {
        fmt.Printf("%s,账户余额不足，不能转账。。", from)
        os.Exit(1)
    }
    return total, spentableUTXO

}

func (utxoSet *UTXOSet) FindUnPackageSpentableUTXOs(from string, txs []*Transaction) []*UTXO {
    var unUTXOs []*UTXO
    //存储已经花费
    spentTxOutput := make(map[string][]int)

    for i := len(txs) - 1; i >= 0; i-- {
        unUTXOs = caculate(txs[i], from, spentTxOutput, unUTXOs)
    }
    return unUTXOs
}

//每次创建区块后，更新未花费的表
func (utxoSet *UTXOSet) Update() {
    /*
    每当创建新区块后，都会花掉一些原来的 utxo，产生新的 utxo。
    删除已经花费的，增加新产生的未花费
    表中存储的数据结构：
    key：交易 ID
    value：TxInputs
        TxInputs 里是 UTXO 数组

     */

    //1.获取最新的区块，由于该 block 的产生，
    newBlock := utxoSet.BlockChain.Iterator().Next()

    //2.遍历该区块的交易
    inputs := []*TXInput{}
    //spentUTXOMap := make(map[string]*TxOutputs)
    //未花费
    outsMap := make(map[string]*TxOutputs)

    //获取已经花费的
    for _, tx := range newBlock.Txs {
        if tx.IsCoinbaseTransaction(){
            continue
        }
        for _, in := range tx.Vins {
            inputs = append(inputs, in)
        }
    }
    fmt.Println("inputs 的长度：", len(inputs), inputs)
    //以上是找出新添加的区块中的所有的 Input

    //以下是找到新添加的区块中的未花费了的 Output
    for _, tx := range newBlock.Txs {

        utoxs := []*UTXO{}

    outLoop:
        for index, out := range tx.Vouts {
            isSpent := false
            for _, in := range inputs {
                if bytes.Compare(in.TxID, tx.TxID) == 0 && in.Vout == index && bytes.Compare(out.PubKeyHash, PubKeyHash(in.PublicKey)) == 0 {
                    isSpent = true
                    continue outLoop
                }
            }
            if isSpent == false {
                utxo := &UTXO{tx.TxID, index, out}
                utoxs = append(utoxs, utxo)
                fmt.Println("outsMaps,",out.Value)
            }
        }
        if len(utoxs) > 0 {
            txIDStr := hex.EncodeToString(tx.TxID)
            outsMap[txIDStr] = &TxOutputs{utoxs}
        }

    }
    fmt.Println("outsMap 的长度：", len(outsMap), outsMap)
    /*
    outsMap[7788] = utxo
    outsMap[5555] = utxo,utxo
    outsMap[4444] = utxo

     */

    //删除已经花费了的
    err := utxoSet.BlockChain.DB.Update(func(tx *bolt.Tx) error {
        b := tx.Bucket([]byte(utxoTableName))
        if b != nil {
            //删除 ins 中
            for i:=0;i<len(inputs);i++{

                in:=inputs[i]
                fmt.Println(i,"=========================")

                //txIDStr:=hex.EncodeToString(in.TxID)
                //outputs:=outsMap[txIDStr]
                //if len(outputs.UTXOS) > 0{
                //    utxos:=[]*UTXO{}
                //    for _,utxo:=range outputs.UTXOS{
                //        if bytes.Compare(utxo.Output.PubKeyHash, PubKeyHash(in.PublicKey)) == 0 && in.Vout ==utxo.Index {
                //
                //        }else{
                //            utxos = append(utxos,utxo)
                //        }
                //    }
                //
                //    outsMap[txIDStr] = &TxOutputs{utxos}
                //
                //
                //}else{
                txOutputsBytes := b.Get(in.TxID)
                if len(txOutputsBytes) == 0 {
                    //fmt.Println("break",i)
                    continue
                }

                txOutputs := DeserializeTXOutputs(txOutputsBytes)

                //根据 IxID，如果该 txOutputs 中已经有 output 被新区块花掉了，那么将未花掉的添加到 utxos 里，并标记该 txouputs 要删除
                // 判断是否需要
                isNeedDelete := false
                utxos := []*UTXO{} //存储未花费
                for _, utxo := range txOutputs.UTXOS {
                    if bytes.Compare(utxo.Output.PubKeyHash, PubKeyHash(in.PublicKey)) == 0 && in.Vout == utxo.Index {
                        isNeedDelete = true
                    } else {
                        utxos = append(utxos, utxo)
                    }
                }
                fmt.Println(len(utxos))
                if isNeedDelete == true {
                    b.Delete(in.TxID)
                    if len(utxos) > 0 {
                        //该删除的 txoutputs 中还有其他未花费的
                        //preTXOutputs := outsMap[hex.EncodeToString(in.TxID)]
                        //preTXOutputs.UTXOS = append(preTXOutputs.UTXOS,utxos...)
                        //outsMap[hex.EncodeToString(in.TxID)] = preTXOutputs

                        txOutputs := &TxOutputs{utxos}
                        b.Put(in.TxID, txOutputs.Serilalize())
                        fmt.Println("删除时：map：", len(outsMap), outsMap)
                    }

                }

                //}

            }

            //增加
            for keyID, outPuts := range outsMap {
                keyHashBytes, _ := hex.DecodeString(keyID)
                b.Put(keyHashBytes, outPuts.Serilalize())
            }

        }

        return nil
    })

    if err != nil {
        log.Panic(err)
    }
} 
```

#### 3.2.3 修改`CLI_createBlockChain.go`

打开`day06_08_Update`目录里的 BLC 包。修改`CLI_createBlockChain.go`文件。

修改步骤：

```go
修改步骤：
step1：添加 createGenesisBlockchain()方法，创建完创世区块后，重置 utxoSet。 
```

修改完后代码如下：

```go
package BLC

func (cli *CLI) createGenesisBlockchain(address string){
    //fmt.Println(data)
    CreateBlockChainWithGenesisBlock(address)

    bc:=GetBlockchainObject()
    defer bc.DB.Close()
    if bc != nil{
        utxoSet:=&UTXOSet{bc}
        utxoSet.ResetUTXOSet()
    }

} 
```

#### 3.2.4 修改`CLI_getBalance.go`文件

打开`day06_08_Update`目录里的 BLC 包。修改`CLI_getBalance.go`文件。

修改步骤：

```go
修改步骤：
step1：修改 getBalance()方法，查询余额从 utxoSet 中查询。 
```

修改完后代码如下：

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
    //balance:=bc.GetBalance(address,[]*Transaction{})
    utxoSet:=&UTXOSet{bc}
    balance:= utxoSet.GetBalance(address)
    fmt.Printf("%s,一共有%d 个 Token\n",address,balance)
} 
```

#### 3.2.5 修改`Transaction.go`文件

打开`day06_08_Update`目录里的 BLC 包。修改`Transaction.go`文件。

修改步骤：

```go
修改步骤：
step1：修改 NewSimpleTransaction()方法，从 utxoSet 中查找转账要使用的 utxo。 
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
    "time"
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

func NewSimpleTransaction(from, to string, amount int64, utxoSet *UTXOSet, txs []*Transaction) *Transaction {
    var txInputs [] *TXInput
    var txOutputs [] *TXOuput

    //balance, spendableUTXO := bc.FindSpendableUTXOs(from, amount, txs)
    balance, spendableUTXO := utxoSet.FindSpendableUTXOs(from, amount, txs)

    //代表消费

    //bytes,_:=hex.DecodeString("558e697b0f0f7adfcb6b8f8b542fb5980cd78a58a103a7a9a19e274aabefed5e")
    //bytes,_:=hex.DecodeString("0704aabb159bd1b276dc819b85e49581d80eb53aba69a48c84dc12442b799d8e")

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
    utxoSet.BlockChain.SignTransaction(tx, wallet.PrivateKey,txs)

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

#### 3.2.6 修改`CLI_send.go`文件

打开`day06_08_Update`目录里的 BLC 包。修改`CLI_send.go`文件。

修改步骤：

```go
修改步骤：
step1：修改 send()方法，转账之后更新 utxoSet。 
```

修改完后代码如下：

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

    utxoSet:=&UTXOSet{blockchain}
    //转账成功以后，需要更新
    //utxoSet.ResetUTXOSet()
    utxoSet.Update()
} 
```

#### 3.2.7 修改`BlockChain.go`文件

打开`day06_08_Update`目录里的 BLC 包。修改`BlockChain.go`文件。

修改步骤：

```go
修改步骤：
step1：添加 FindUnSpentOutputMap()方法
    查询未花费的 Output
step2：修改 MineNewBlock(),
    添加 utxoset 对象 
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

    //奖励
    tx := NewCoinBaseTransaction(from[0])
    txs = append(txs, tx)

    utxoSet:=&UTXOSet{bc}

    for i := 0; i < len(from); i++ {

        amountInt, _ := strconv.ParseInt(amount[i], 10, 64)
        tx := NewSimpleTransaction(from[i], to[i], amountInt, utxoSet, txs)

        txs = append(txs, tx)
    }

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

//新增方法
/*
查询未花费的 Output
[string] *TxOutputs
 */
func (bc *BlockChain) FindUnSpentOutputMap() map[string]*TxOutputs {
    iterator := bc.Iterator()

    //存储已经花费：·[txID], txInput
    spentUTXOsMap := make(map[string][]*TXInput)

    //存储未花费
    unSpentOutputMaps := make(map[string]*TxOutputs)

    for {
        block := iterator.Next()

        for i := len(block.Txs) - 1; i >= 0; i-- {
            txOutputs := &TxOutputs{[]*UTXO{}}
            tx := block.Txs[i]

            if !tx.IsCoinbaseTransaction() {
                for _, txInput := range tx.Vins {
                    key := hex.EncodeToString(txInput.TxID)
                    spentUTXOsMap[key] = append(spentUTXOsMap[key], txInput)
                }
            }

            txID := hex.EncodeToString(tx.TxID)
        work:
            for index, out := range tx.Vouts {
                txInputs := spentUTXOsMap[txID]
                if len(txInputs) > 0 {
                    var isSpent bool
                    for _, input := range txInputs {
                        inputPubKeyHash := PubKeyHash(input.PublicKey)
                        if bytes.Compare(inputPubKeyHash, out.PubKeyHash) == 0 {
                            if input.Vout == index {
                                isSpent = true
                                continue work
                            }
                        }
                    }
                    if isSpent == false {
                        utxo:=&UTXO{tx.TxID,index,out}
                        txOutputs.UTXOS = append(txOutputs.UTXOS, utxo)
                    }

                } else {
                    utxo:=&UTXO{tx.TxID,index,out}
                    txOutputs.UTXOS = append(txOutputs.UTXOS, utxo)
                }
            }
            //设置
            unSpentOutputMaps[txID] = txOutputs
        }

        //停止迭代
        var hashInt big.Int
        hashInt.SetBytes(block.PrevBlockHash)

        if hashInt.Cmp(big.NewInt(0)) == 0 {
            break
        }
    }
    return unSpentOutputMaps
} 
```

#### 3.2.8 `main.go`无需修改

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

## 4\. 优化内容讲解

### 4.1 奖励 Reward

在上一篇文章中，我们略过的一个小细节是挖矿奖励。现在，我们已经可以来完善这个细节了。

挖矿奖励，实际上就是一笔`coinbase` 交易。当一个挖矿节点开始挖出一个新块时，它会将交易从队列中取出，并在前面附加一笔`coinbase`交易。`coinbase` 交易只有一个输出，里面包含了矿工的公钥哈希。

实现奖励，非常简单，更新 `MineNewBlock()` 即可：

```go
//挖掘新的区块
func (bc *BlockChain) MineNewBlock(from, to, amount []string) {

    var txs []*Transaction

    //奖励
    tx := NewCoinBaseTransaction(from[0])
    txs = append(txs, tx)

    utxoSet:=&UTXOSet{bc}

    for i := 0; i < len(from); i++ {

        amountInt, _ := strconv.ParseInt(amount[i], 10, 64)
        tx := NewSimpleTransaction(from[i], to[i], amountInt, utxoSet, txs)

        txs = append(txs, tx)
    }

...

} 
```

在我们的实现中，创建交易的第一个人同时挖出了新块，所以会得到一笔奖励。

在项目添加了奖励之后，我们进行测试，首先在终端创建 3 个地址：

```go
hanru:day06_08_Update ruby$ go build -o bc main.go
hanru:day06_08_Update ruby$ ls
hanru:day06_08_Update ruby$ ./bc
hanru:day06_08_Update ruby$ ./bc createwallet
hanru:day06_08_Update ruby$ ./bc createwallet
hanru:day06_08_Update ruby$ ./bc createwallet
hanru:day06_08_Update ruby$ ./bc addresslists 
```

运行结果如下：

![`img.kongyixueyuan.com/0903_%E8%BF%90%E8%A1%8C%E7%BB%93%E6%9E%9C1.png`](img/75ec47c36fb15986c8c60c2939ea64c9.jpg)

接下来我们创建创世区块，并进行一次转账：

```go
hanru:day06_08_Update ruby$ ./bc createblockchain -address '1EAQKaYrgEsLEvpwbXcVKjsMdwSBfgkB1w'
hanru:day06_08_Update ruby$ ./bc getbalance -address '1EAQKaYrgEsLEvpwbXcVKjsMdwSBfgkB1w'
hanru:day06_08_Update ruby$ ./bc send -from '["1EAQKaYrgEsLEvpwbXcVKjsMdwSBfgkB1w"]' -to '["1JbzMa6gWJecePEyXNHADt9zVcYQy7wZYY"]' -amount '["4"]'
hanru:day06_08_Update ruby$ ./bc getbalance -address '1EAQKaYrgEsLEvpwbXcVKjsMdwSBfgkB1w'hanru:day06_08_Update ruby$ ./bc getbalance -address '1JbzMa6gWJecePEyXNHADt9zVcYQy7wZYY' 
```

运行结果如下：

![`img.kongyixueyuan.com/0904_%E8%BF%90%E8%A1%8C%E7%BB%93%E6%9E%9C2.png`](img/624afa1f4b637959eb8014f8f56ca2d7.jpg)

> 说明：
> 
> 1.  比如创世区块中 1EAQKaYrgEsLEvpwbXcVKjsMdwSBfgkB1w 地址有 10 个 Token，
> 2.  转账给 1JbzMa6gWJecePEyXNHADt9zVcYQy7wZYY 地址 4 个 Token，
> 3.  那么 1EAQKaYrgEsLEvpwbXcVKjsMdwSBfgkB1w 的余额是 16(转账找零 6 个，加上得到奖励 10 个），
> 4.  1JbzMa6gWJecePEyXNHADt9zVcYQy7wZYY 的余额是 4。

接下来我们测试一下，一次转账多笔交易：

```go
hanru:day06_08_Update ruby$ ./bc send -from '["1EAQKaYrgEsLEvpwbXcVKjsMdwSBfgkB1w","1JbzMa6gWJecePEyXNHADt9zVcYQy7wZYY"]' -to '["1JbzMa6gWJecePEyXNHADt9zVcYQy7wZYY","19qfQaPjbDJf8V1Leb7ELDMErSiCCMQ6cL"]' -amount '["9","6"]'
hanru:day06_08_Update ruby$ ./bc getbalance -address '1EAQKaYrgEsLEvpwbXcVKjsMdwSBfgkB1w'
hanru:day06_08_Update ruby$ ./bc getbalance -address '1JbzMa6gWJecePEyXNHADt9zVcYQy7wZYY'
hanru:day06_08_Update ruby$ ./bc getbalance -address '19qfQaPjbDJf8V1Leb7ELDMErSiCCMQ6cL' 
```

运行结果如下：

![`img.kongyixueyuan.com/0905_%E8%BF%90%E8%A1%8C%E7%BB%93%E6%9E%9C3.png`](img/4469323bd1b01b59c5171152052728a7.jpg)

> 说明：
> 
> 1.  1EAQKaYrgEsLEvpwbXcVKjsMdwSBfgkB1w 地址的余额是 16，
>     
>     
> 2.  1JbzMa6gWJecePEyXNHADt9zVcYQy7wZYY 地址的余额是 4。
>     
>     
> 3.  本次转账，1EAQKaYrgEsLEvpwbXcVKjsMdwSBfgkB1w 地址转账给 1JbzMa6gWJecePEyXNHADt9zVcYQy7wZYY 地址 9 个，同时 1JbzMa6gWJecePEyXNHADt9zVcYQy7wZYY 地址转账给 19qfQaPjbDJf8V1Leb7ELDMErSiCCMQ6cL 地址 6 个。
>     
>     
> 4.  余额：
>     
>     
>     
>     1EAQKaYrgEsLEvpwbXcVKjsMdwSBfgkB1w 地址余额：转账后剩余 7，加系统奖励 10，一共 17 个 Token
>     
>     
>     
>     1JbzMa6gWJecePEyXNHADt9zVcYQy7wZYY 地址余额：原来余额 4，加上接收转来的 9，转出 6，剩余 7 个 Token
>     
>     
>     
>     19qfQaPjbDJf8V1Leb7ELDMErSiCCMQ6cL 地址余额：9 个 Token。

### 4.2 UTXO 集

在持久化的章节中，我们研究了 Bitcoin Core 是如何在一个数据库中存储块的，并且了解到区块被存储在 `blocks` 数据库，交易输出被存储在 `chainstate` 数据库。会回顾一下 `chainstate` 的结构：

1.  `c` + 32 字节的交易哈希 -> 该笔交易的未花费交易输出记录
2.  `B` + 32 字节的块哈希 -> 未花费交易输出的块哈希

在之前那篇文章中，虽然我们已经实现了交易，但是并没有使用 `chainstate` 来存储交易的输出。所以，接下来我们继续完成这部分。

`chainstate` 不存储交易。它所存储的是 UTXO 集，也就是未花费交易输出的集合。除此以外，它还存储了“数据库表示的未花费交易输出的块哈希”，不过我们会暂时略过块哈希这一点，因为我们还没有用到块高度（但是我们会在接下来的文章中继续改进）。

那么，我们为什么需要 UTXO 集呢？

来思考一下我们早先实现的 `Blockchain.UnUTXOs()` 方法：

```go
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
```

这个函数找到有未花费输出的交易。由于交易被保存在区块中，所以它会对区块链里面的每一个区块进行迭代，检查里面的每一笔交易。截止 2017 年 9 月 18 日，在比特币中已经有 485，860 个块，整个数据库所需磁盘空间超过 140 Gb。这意味着一个人如果想要验证交易，必须要运行一个全节点。此外，验证交易将会需要在许多块上进行迭代。

整个问题的解决方案是有一个仅有未花费输出的索引，这就是 UTXO 集要做的事情：这是一个从所有区块链交易中构建（对区块进行迭代，但是只须做一次）而来的缓存，然后用它来计算余额和验证新的交易。截止 2017 年 9 月，UTXO 集大概有 2.7 Gb。

接下来，我们个通过代码实现一下：

为了在 UTXO 集中存储未花费的 utxo，我们可以创建一个新的结构体`TxOutputs`，通过对象的序列化和反序列化，便于持久化存储。

在 BLC 包下，新建`TxOutputs.go`文件，并编写代码如下：

```go
package BLC

import (
    "bytes"
    "encoding/gob"
    "log"
)

type TxOutputs struct {
    UTXOS []*UTXO
}

//序列化
func (outs *TxOutputs) Serilalize()[]byte{
    //1.创建一个 buffer
    var result bytes.Buffer
    //2.创建一个编码器
    encoder := gob.NewEncoder(&result)
    //3.编码--->打包
    err := encoder.Encode(outs)
    if err != nil {
        log.Panic(err)
    }
    return result.Bytes()
}

//反序列化
func DeserializeTXOutputs(txOutputsBytes [] byte) *TxOutputs {
    var txOutputs TxOutputs
    var reader = bytes.NewReader(txOutputsBytes)
    //1.创建一个解码器
    decoder := gob.NewDecoder(reader)
    //解包
    err := decoder.Decode(&txOutputs)
    if err != nil {
        log.Panic(err)
    }
    return &txOutputs
} 
```

接下来，在`BlockChain.go`文件中添加`FindUnSpentOutputMap()`方法，用于获取每个`TxOutputs`。

```go
 //新增方法
/*
查询未花费的 Output
[string] *TxOutputs
 */
func (bc *BlockChain) FindUnSpentOutputMap() map[string]*TxOutputs {
    iterator := bc.Iterator()

    //存储已经花费：·[txID], txInput
    spentUTXOsMap := make(map[string][]*TXInput)

    //存储未花费
    unSpentOutputMaps := make(map[string]*TxOutputs)

    for {
        block := iterator.Next()

        for i := len(block.Txs) - 1; i >= 0; i-- {
            txOutputs := &TxOutputs{[]*UTXO{}}
            tx := block.Txs[i]

            if !tx.IsCoinbaseTransaction() {
                for _, txInput := range tx.Vins {
                    key := hex.EncodeToString(txInput.TxID)
                    spentUTXOsMap[key] = append(spentUTXOsMap[key], txInput)
                }
            }

            txID := hex.EncodeToString(tx.TxID)
        work:
            for index, out := range tx.Vouts {
                txInputs := spentUTXOsMap[txID]
                if len(txInputs) > 0 {
                    var isSpent bool
                    for _, input := range txInputs {
                        inputPubKeyHash := PubKeyHash(input.PublicKey)
                        if bytes.Compare(inputPubKeyHash, out.PubKeyHash) == 0 {
                            if input.Vout == index {
                                isSpent = true
                                continue work
                            }
                        }
                    }
                    if isSpent == false {
                        utxo:=&UTXO{tx.TxID,index,out}
                        txOutputs.UTXOS = append(txOutputs.UTXOS, utxo)
                    }

                } else {
                    utxo:=&UTXO{tx.TxID,index,out}
                    txOutputs.UTXOS = append(txOutputs.UTXOS, utxo)
                }
            }
            //设置
            unSpentOutputMaps[txID] = txOutputs
        }

        //停止迭代
        var hashInt big.Int
        hashInt.SetBytes(block.PrevBlockHash)

        if hashInt.Cmp(big.NewInt(0)) == 0 {
            break
        }
    }
    return unSpentOutputMaps
} 
```

该方法得到一个`map`，我们将一笔交易的`TxID`作为`key`，将该交易中的`TxOutput`构建的 utxo，再通过 utxo 构建`TxOutputs`作为 value 值。

接下来，我们在 BLC 下再创建一个 go 文件，命名为：`UTXO_Set.go`。添加一个结构体：

```go
type UTXOSet struct {
    BlockChain *BlockChain
} 
```

我们将会使用一个单一数据库，但是我们会将 UTXO 集从存储在不同的 bucket 中。因此，`UTXOSet` 跟 `Blockchain` 一起。

接下类继续定义一个`ResetUTXOSet()`方法：

```go
 const utxoTableName = "utxoTable"

//重置数据库表
func (utxoSet *UTXOSet) ResetUTXOSet() {
    err := utxoSet.BlockChain.DB.Update(func(tx *bolt.Tx) error {
        b := tx.Bucket([]byte(utxoTableName))
        if b != nil {
            err := tx.DeleteBucket([]byte(utxoTableName))
            if err != nil {
                log.Panic("重置中，删除表失败。。")
            }

        }
        b, err := tx.CreateBucket([]byte(utxoTableName))
        if err != nil {
            log.Panic("重置中，创建新表失败。。")
        }
        if b != nil {
            txOutputMap := utxoSet.BlockChain.FindUnSpentOutputMap()
            //fmt.Println("未花费 outputmap：",txOutputMap)
            for txIDStr, outputs := range txOutputMap {
                txID, _ := hex.DecodeString(txIDStr)
                b.Put(txID, outputs.Serilalize())
            }
        }

        return nil
    })
    if err != nil {
        log.Panic(err)
    }
} 
```

这个方法初始化了 UTXO 集。首先，如果 bucket 存在就先移除，然后从区块链中获取所有的未花费输出，最终将输出保存到 bucket 中。该方法也用于重置 UTXO 集。

接下来，修改`CLI_createBlockChain.go`文件，代码如下:

```go
package BLC

func (cli *CLI) createGenesisBlockchain(address string){
    //fmt.Println(data)
    CreateBlockChainWithGenesisBlock(address)

    bc:=GetBlockchainObject()
    defer bc.DB.Close()
    if bc != nil{
        utxoSet:=&UTXOSet{bc}
        utxoSet.ResetUTXOSet()
    }

} 
```

当创建创世区块后，就会立刻进行初始化 UTXO 集。目前，即使这里看起来有点“杀鸡用牛刀”，因为一条链开始的时候，只有一个块，里面只有一笔交易。

写到此处，我们可以先进行一次测试，修改`CLI.go`文件在，添加一个 test 命令。

代码如下：

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
func (cli *CLI) Run() {
    //判断命令行参数的长度
    isValidArgs()

    //1.创建 flagset 标签对象
    ...
    testCmd:=flag.NewFlagSet("test",flag.ExitOnError)

    ...

    //3.解析
    switch os.Args[1] {
    ...
    case "test":
        err := testCmd.Parse(os.Args[2:])
        if err != nil {
            log.Panic(err)
        }
    default:
        printUsage()
        os.Exit(1) //退出
    }

    ...

    if testCmd.Parsed(){
        cli.TestMethod()
    }

}

...
func printUsage() {
    fmt.Println("Usage:")
    fmt.Println("\tcreatewallet -- 创建钱包")
    fmt.Println("\taddresslists -- 输出所有钱包地址")
    fmt.Println("\tcreateblockchain -address DATA -- 创建创世区块")
    fmt.Println("\tsend -from From -to To -amount Amount - 交易数据")
    fmt.Println("\tprintchain - 输出信息:")
    fmt.Println("\tgetbalance -address DATA -- 查询账户余额")
    fmt.Println("\ttest -- 测试")
} 
```

然后在 BLC 包下创建一个`CLI_testmethod.go`文件，用于测试程序，代码如下：

```go
package BLC

import "fmt"

func (cli *CLI) TestMethod(){
    blockchain:=GetBlockchainObject()
    //defer blockchain.DB.Close()

    unSpentOutputMap:=blockchain.FindUnSpentOutputMap()
    fmt.Println(unSpentOutputMap)
    for key,value:=range unSpentOutputMap{
        fmt.Println(key)
        for _,utxo:=range value.UTXOS{
            fmt.Println("金额：",utxo.Output.Value)
            fmt.Printf("地址：%v\n",utxo.Output.PubKeyHash)
            fmt.Println("---------------------")
        }
    }

    utxoSet:=&UTXOSet{blockchain}
    utxoSet.ResetUTXOSet()
} 
```

> 注意，命名文件时，不要命名为`xxxtest.go`，因为这是测试文件的命名格式。

打开终端，删除之前的`blockchain.db`文件，并执行以下命令：

```go
hanru:day06_08_Update ruby$ rm -rf blockchain.db 
hanru:day06_08_Update ruby$ go build -o bc main.go
hanru:day06_08_Update ruby$ ./bc
hanru:day06_08_Update ruby$ ./bc addresslists
hanru:day06_08_Update ruby$ ./bc createblockchain -address '19qfQaPjbDJf8V1Leb7ELDMErSiCCMQ6cL'
hanru:day06_08_Update ruby$ ./bc test 
```

运行效果如图：

![`img.kongyixueyuan.com/0906_%E8%BF%90%E8%A1%8C%E7%BB%93%E6%9E%9C4.png`](img/bccbc5d6962d583e31bb65d941385c77.jpg)

因为创建了创世区块，有`CoinBase`交易的 10 个 Token，这是一个未花费的`TxOutput`，找到之后可以存储到 UTXO 集中。

### 4.3 优化查询余额

有了 UTXO 集，我们想要查询余额，可以不用遍历整个区块链的所有区块，而是直接查找 UTXO 集，找出对应地址的 utxo，进行累加即可。

在 UtxoSet 中，添加`GetBalance()`方法，和`FindUnspentOutputsForAddress()`方法，代码如下：

```go
 func (utxoSet *UTXOSet) GetBalance(address string) int64 {
    utxos := utxoSet.FindUnspentOutputsForAddress(address)
    var amount int64
    for _, utxo := range utxos {
        amount += utxo.Output.Value
        fmt.Println(address,"余额：",utxo.Output.Value)
    }
    //fmt.Printf("%s 账户，有%d 个 Token\n",address,amount)
    return amount

}

func (utxoSet *UTXOSet) FindUnspentOutputsForAddress(address string) []*UTXO {
    var utxos [] *UTXO
    //查询数据，遍历所有的未花费
    err := utxoSet.BlockChain.DB.View(func(tx *bolt.Tx) error {
        b := tx.Bucket([]byte(utxoTableName))
        if b != nil {
            c := b.Cursor()
            for k, v := c.First(); k != nil; k, v = c.Next() {
                //fmt.Printf("key=%s,value=%v\n", k, v)
                txOutputs := DeserializeTXOutputs(v)
                for _, utxo := range txOutputs.UTXOS {
                    if utxo.Output.UnLockWithAddress(address) {
                        utxos = append(utxos, utxo)
                    }
                }
            }
        }

        return nil
    })
    if err != nil {
        log.Panic(err)
    }
    return utxos
} 
```

接下来修改，修改`CLI_getBalance.go`文件，不再通过 bc 对象调用原来的查询方法了，改用 utxoSet 对象进行查询余额，代码如下：

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
    //balance:=bc.GetBalance(address,[]*Transaction{})
    utxoSet:=&UTXOSet{bc}
    balance:= utxoSet.GetBalance(address)
    fmt.Printf("%s,一共有%d 个 Token\n",address,balance)
} 
```

接下来，我们测试一下，先进行一次转账(转账我们还没有使用 UTXO 集优化)，然后再查询余额。在终端中输入以下命令：

```go
hanru:day06_08_Update ruby$ ./bc send -from '["19qfQaPjbDJf8V1Leb7ELDMErSiCCMQ6cL"]' -to '["1EAQKaYrgEsLEvpwbXcVKjsMdwSBfgkB1w"]' -amount '["4"]'
hanru:day06_08_Update ruby$ ./bc test 
```

运行结果如下：

![`img.kongyixueyuan.com/0907_%E6%9F%A5%E8%AF%A2%E4%BD%99%E9%A2%9D.png`](img/dc8f4bc33e31d589803c5092cd4bca14.jpg)

注意，转账后一定要调用 test，表示重置 UTXO 集，否则 UTXO 集并没有存储最新的 utxo。稍后我们事先转账优化，转账后自动更新 UTXO 集即可。

接下来，我们查询余额：

```go
hanru:day06_08_Update ruby$ ./bc getbalance -address '19qfQaPjbDJf8V1Leb7ELDMErSiCCMQ6cL'
hanru:day06_08_Update ruby$ ./bc getbalance -address '1EAQKaYrgEsLEvpwbXcVKjsMdwSBfgkB1w' 
```

运行效果如图：

![`img.kongyixueyuan.com/0908_%E6%9F%A5%E8%AF%A2%E4%BD%99%E9%A2%9D.png`](img/31b095f6784a4c5969678d09dd7ea7a7.jpg)

### 4.4 优化转账交易

因为我们已经将区块链中的 utxo 都存储到 UTXO 集中，所以 UTXO 集可以用于转账发送币。

在`UTXO_Set.go`文件中，添加`FindSpendableUTXOs()`方法，和`FindUnPackageSpentableUTXOs()`方法。代码如下：

```go
 //添加一个方法：用于查询给定地址下的，要转账使用的可以使用的 utxo
func (utxoSet *UTXOSet) FindSpendableUTXOs(from string, amount int64, txs [] *Transaction) (int64, map[string][]int) {
    spentableUTXO := make(map[string][]int)
    var total int64 = 0
    //1.找出未打包的 Transaction 中未花费的
    unPackageUTXOs := utxoSet.FindUnPackageSpentableUTXOs(from, txs)
    for _, utxo := range unPackageUTXOs {
        total += utxo.Output.Value
        txIDStr := hex.EncodeToString(utxo.TxID)
        spentableUTXO[txIDStr] = append(spentableUTXO[txIDStr], utxo.Index)
        fmt.Println(amount,",未打包，转账花费：",utxo.Output.Value)
        if total >= amount {
            return total, spentableUTXO
        }
    }

    //钱不够
    //2.找出已经存在数据库中的未花费的
    err := utxoSet.BlockChain.DB.View(func(tx *bolt.Tx) error {
        b := tx.Bucket([]byte(utxoTableName))
        if b != nil {
            c := b.Cursor()
        dbLoop:
            for k, v := c.First(); k != nil; k, v = c.Next() {
                txOutpus := DeserializeTXOutputs(v)
                for _, utxo := range txOutpus.UTXOS {
                    if utxo.Output.UnLockWithAddress(from) {
                        total += utxo.Output.Value
                        txIDStr := hex.EncodeToString(utxo.TxID)
                        spentableUTXO[txIDStr] = append(spentableUTXO[txIDStr], utxo.Index)
                        fmt.Println(amount,",数据库，转账花费：",utxo.Output.Value)
                        if total >= amount {
                            break dbLoop
                        }
                    }

                }
            }
        }
        return nil
    })
    if err != nil {
        log.Panic(err)
    }

    if total < amount {
        fmt.Printf("%s,账户余额不足，不能转账。。", from)
        os.Exit(1)
    }
    return total, spentableUTXO

}
//查找未打包中的 utxo
func (utxoSet *UTXOSet) FindUnPackageSpentableUTXOs(from string, txs []*Transaction) []*UTXO {
    var unUTXOs []*UTXO
    //存储已经花费
    spentTxOutput := make(map[string][]int)

    for i := len(txs) - 1; i >= 0; i-- {
        unUTXOs = caculate(txs[i], from, spentTxOutput, unUTXOs)
    }
    return unUTXOs
} 
```

以上方法用于找出转账需要用的 utxo。先查找未打包交易中的 utxo，然后再查找 UTXO 集中。

接下来，在`BlockChain.go`文件中，修改`MineNewBlock()`方法。添加 utxoSet 对象。修改代码如下：

```go
 //挖掘新的区块
func (bc *BlockChain) MineNewBlock(from, to, amount []string) {

    var txs []*Transaction

    //奖励
    tx := NewCoinBaseTransaction(from[0])
    txs = append(txs, tx)

    utxoSet:=&UTXOSet{bc}

    for i := 0; i < len(from); i++ {

        amountInt, _ := strconv.ParseInt(amount[i], 10, 64)
        tx := NewSimpleTransaction(from[i], to[i], amountInt, utxoSet, txs)

        txs = append(txs, tx)
    }

    ...

} 
```

接下来需要修改`Transaction.go`文件中的，`NewSimpleTransaction()`方法，改为调用通过 UTXO 集获取转账需要的 utxo。代码如下：

```go
 func NewSimpleTransaction(from, to string, amount int64, utxoSet *UTXOSet, txs []*Transaction) *Transaction {
    var txInputs [] *TXInput
    var txOutputs [] *TXOuput

    //balance, spendableUTXO := bc.FindSpendableUTXOs(from, amount, txs)
    balance, spendableUTXO := utxoSet.FindSpendableUTXOs(from, amount, txs)

    ...
} 
```

通过转账产生交易，一定会花费之前的 utxo，再产生新的 utxo。有了 UTXO 集，也就意味着我们的数据（交易）现在已经被分开存储：实际交易被存储在区块链中，未花费输出被存储在 UTXO 集中。这样一来，我们就需要一个良好的同步机制，因为我们想要 UTXO 集时刻处于最新状态，并且存储最新交易的输出。但是我们不想每生成一个新块，就重新生成索引，因为这正是我们要极力避免的频繁区块链扫描。因此，我们需要一个机制来更新 UTXO 集。接下来，我们在`UTXO_Set.go`文件中，添加一个`Update()`方法，代码如下：

```go
 //每次创建区块后，更新未花费的表
func (utxoSet *UTXOSet) Update() {
    /*
    每当创建新区块后，都会花掉一些原来的 utxo，产生新的 utxo。
    删除已经花费的，增加新产生的未花费
    表中存储的数据结构：
    key：交易 ID
    value：TxInputs
        TxInputs 里是 UTXO 数组

     */

    //1.获取最新的区块，由于该 block 的产生，
    newBlock := utxoSet.BlockChain.Iterator().Next()

    //2.遍历该区块的交易
    inputs := []*TXInput{}
    //spentUTXOMap := make(map[string]*TxOutputs)
    //未花费
    outsMap := make(map[string]*TxOutputs)

    //获取已经花费的
    for _, tx := range newBlock.Txs {
        if tx.IsCoinbaseTransaction(){
            continue
        }
        for _, in := range tx.Vins {
            inputs = append(inputs, in)
        }
    }
    fmt.Println("inputs 的长度：", len(inputs), inputs)
    //以上是找出新添加的区块中的所有的 Input

    //以下是找到新添加的区块中的未花费了的 Output
    for _, tx := range newBlock.Txs {

        utoxs := []*UTXO{}

    outLoop:
        for index, out := range tx.Vouts {
            isSpent := false
            for _, in := range inputs {
                if bytes.Compare(in.TxID, tx.TxID) == 0 && in.Vout == index && bytes.Compare(out.PubKeyHash, PubKeyHash(in.PublicKey)) == 0 {
                    isSpent = true
                    continue outLoop
                }
            }
            if isSpent == false {
                utxo := &UTXO{tx.TxID, index, out}
                utoxs = append(utoxs, utxo)
                fmt.Println("outsMaps,",out.Value)
            }
        }
        if len(utoxs) > 0 {
            txIDStr := hex.EncodeToString(tx.TxID)
            outsMap[txIDStr] = &TxOutputs{utoxs}
        }

    }
    fmt.Println("outsMap 的长度：", len(outsMap), outsMap)
    /*
    outsMap[7788] = utxo
    outsMap[5555] = utxo,utxo
    outsMap[4444] = utxo

     */

    //删除已经花费了的
    err := utxoSet.BlockChain.DB.Update(func(tx *bolt.Tx) error {
        b := tx.Bucket([]byte(utxoTableName))
        if b != nil {
            //删除 ins 中
            for i:=0;i<len(inputs);i++{

                in:=inputs[i]
                fmt.Println(i,"=========================")

                //txIDStr:=hex.EncodeToString(in.TxID)
                //outputs:=outsMap[txIDStr]
                //if len(outputs.UTXOS) > 0{
                //    utxos:=[]*UTXO{}
                //    for _,utxo:=range outputs.UTXOS{
                //        if bytes.Compare(utxo.Output.PubKeyHash, PubKeyHash(in.PublicKey)) == 0 && in.Vout ==utxo.Index {
                //
                //        }else{
                //            utxos = append(utxos,utxo)
                //        }
                //    }
                //
                //    outsMap[txIDStr] = &TxOutputs{utxos}
                //
                //
                //}else{
                txOutputsBytes := b.Get(in.TxID)
                if len(txOutputsBytes) == 0 {
                    //fmt.Println("break",i)
                    continue
                }

                txOutputs := DeserializeTXOutputs(txOutputsBytes)

                //根据 IxID，如果该 txOutputs 中已经有 output 被新区块花掉了，那么将未花掉的添加到 utxos 里，并标记该 txouputs 要删除
                // 判断是否需要
                isNeedDelete := false
                utxos := []*UTXO{} //存储未花费
                for _, utxo := range txOutputs.UTXOS {
                    if bytes.Compare(utxo.Output.PubKeyHash, PubKeyHash(in.PublicKey)) == 0 && in.Vout == utxo.Index {
                        isNeedDelete = true
                    } else {
                        utxos = append(utxos, utxo)
                    }
                }
                fmt.Println(len(utxos))
                if isNeedDelete == true {
                    b.Delete(in.TxID)
                    if len(utxos) > 0 {
                        //该删除的 txoutputs 中还有其他未花费的
                        //preTXOutputs := outsMap[hex.EncodeToString(in.TxID)]
                        //preTXOutputs.UTXOS = append(preTXOutputs.UTXOS,utxos...)
                        //outsMap[hex.EncodeToString(in.TxID)] = preTXOutputs

                        txOutputs := &TxOutputs{utxos}
                        b.Put(in.TxID, txOutputs.Serilalize())
                        fmt.Println("删除时：map：", len(outsMap), outsMap)
                    }

                }

                //}

            }

            //增加
            for keyID, outPuts := range outsMap {
                keyHashBytes, _ := hex.DecodeString(keyID)
                b.Put(keyHashBytes, outPuts.Serilalize())
            }

        }

        return nil
    })

    if err != nil {
        log.Panic(err)
    }
} 
```

虽然这个方法看起来有点复杂，但是它所要做的事情非常直观。当挖出一个新块时，应该更新 UTXO 集。更新意味着移除已花费输出，并从新挖出来的交易中加入未花费输出。如果一笔交易的输出被移除，并且不再包含任何输出，那么这笔交易也应该被移除。相当简单！

最后，修改`CLI_send.go`文件，通过 utxoSet 进行转账并且转账后更新。

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

    utxoSet:=&UTXOSet{blockchain}
    //转账成功以后，需要更新
    //utxoSet.ResetUTXOSet()
    utxoSet.Update()
} 
```

现在，让我们来进行最后的测试，在终端输入一下命令：

```go
hanru:day06_08_Update ruby$ ./bc send -from '["1EAQKaYrgEsLEvpwbXcVKjsMdwSBfgkB1w","19qfQaPjbDJf8V1Leb7ELDMErSiCCMQ6cL"]' -to '["1JbzMa6gWJecePEyXNHADt9zVcYQy7wZYY","1JbzMa6gWJecePEyXNHADt9zVcYQy7wZYY"]' -amount '["3","5"]' 
```

执行结果如下：

![`img.kongyixueyuan.com/0909_%E8%BD%AC%E8%B4%A6.png`](img/7633f99f098d2a6ba0863294d6729fd2.jpg)

我们来查询一下余额：

```go
hanru:day06_08_Update ruby$ ./bc getbalance -address '19qfQaPjbDJf8V1Leb7ELDMErSiCCMQ6cL'
hanru:day06_08_Update ruby$ ./bc getbalance -address '1EAQKaYrgEsLEvpwbXcVKjsMdwSBfgkB1w'
hanru:day06_08_Update ruby$ ./bc getbalance -address '1JbzMa6gWJecePEyXNHADt9zVcYQy7wZYY' 
```

运行结果如下：

![`img.kongyixueyuan.com/0910_%E8%BD%AC%E8%B4%A6.png`](img/2928ea24a00f7bee1201d8dd65395791.jpg)

## 5\. 总结

本章节中，我们并没有在项目中新增功能，只是做了一些优化。首先加入了奖励 Reward 机制。虽然程序中我们实现的比较建议，仅仅是给发起转账的人奖励 10 个 Token，(如果一次转账多笔交易，只奖励给第一个转账人)。然后我们引入了 UTXO 集，进行项目代码优化。UTXO 集的原理，就是我们将所有区块的未花费的 utxo，单独存储到一个 bucket 中。无论是进行余额查询，还是转账操作，都无需遍历查询所有的区块，查询所有的交易去找未花费 utxo 了，只需要查询 UTXO 集即可。如果是转账操作，转账后需要及时更新 UTXO 集。

[项目源代码](https://github.com/rubyhan1314/PublicChain)