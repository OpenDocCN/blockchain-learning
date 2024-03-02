# 第十一章 实现简易版区块链的网络

# 网络(network)

到目前为止，我们所构建的原型已经具备了区块链所有的关键特性：匿名，安全，随机生成的地址；区块链数据存储；工作量证明系统；可靠地存储交易。尽管这些特性都不可或缺，但是仍有不足。能够使得这些特性真正发光发热，使得加密货币成为可能的，是**网络（network）**。如果实现的这样一个区块链仅仅运行在单一节点上，有什么用呢？如果只有一个用户，那么这些基于密码学的特性，又有什么用呢？正是由于网络，才使得整个机制能够运转和发光发热。

你可以将这些区块链特性认为是规则（rule），类似于人类在一起生活，繁衍生息建立的规则，一种社会安排。区块链网络就是一个程序社区，里面的每个程序都遵循同样的规则，正是由于遵循着同一个规则，才使得网络能够长存。类似的，当人们都有着同样的想法，就能够将拳头攥在一起构建一个更好的生活。如果有人遵循着不同的规则，那么他们就将生活在一个分裂的社区（州，公社，等等）中。同样的，如果有区块链节点遵循不同的规则，那么也会形成一个分裂的网络。

**重点在于**：如果没有网络，或者大部分节点都不遵守同样的规则，那么规则就会形同虚设，毫无用处！

> 声明：不幸的是，本文并没有实现一个真实的 P2P 网络原型。但是会展示一个最常见的场景，这个场景涉及不同类型的节点。继续改进这个场景，将它实现为一个 P2P 网络，对你来说是一个很好的挑战和实践！除了本文的场景，我们也无法保证在其他场景将会正常工作。抱歉！
> 
> 本文的代码实现变化很大。

## 1\. 课程目标

1.  学会通过动态设置环境变量设置 NODE_ID
2.  了解节点之间的工作和消息传递的原理
3.  主节点和钱包节点以及矿工节点之间通信的流程
4.  项目中通过代码实现消息传递

## 2\. 项目代码及效果展示

### 2.1 项目代码结构

![`img.kongyixueyuan.com/1101_%E9%A1%B9%E7%9B%AE%E7%BB%93%E6%9E%84.gif`](img/6829ae95f361a7f50bfad7c668e3266a.jpg)

### 2.2 项目运行结果

钱包节点同步数据，效果图 1：

![`img.kongyixueyuan.com/1102_%E8%BF%90%E8%A1%8C%E6%95%88%E6%9E%9C1.gif`](img/47ae277f7a80c19420dc63c29775a6a3.jpg)

矿工节点挖矿，效果图 2：

![`img.kongyixueyuan.com/1102_%E8%BF%90%E8%A1%8C%E6%95%88%E6%9E%9C2.gif`](img/c46837e3cfb2452b2893ab69c456fe3f.jpg)

## 3\. 创建项目

### 3.1 创建工程

首先打开 Goland 开发工具

打开工程：`mypublicchain`

创建项目：将上一次的项目代码，`day07_09_Merkle`，复制为`day08_10_Net`

> 说明：我们每一章节的项目代码，都是在上一个章节上进行添加。所以拷贝上一次的项目代码，然后进行新内容的添加或修改。

### 3.2 代码实现

#### 3.2.1 修改`CLI.go`文件

打开`day08_10_Net`目录里的 BLC 包。修改`CLI.go`文件。

修改步骤：

```go
修改步骤：
step1：修改 Run()方法，
    设置 NODE_ID
    添加新的命令，用于启动节点，
    以及修改转账 send 命令。 
```

修改完后代码如下：

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

    /*
    获取节点 ID
    解释：返回当前进程的环境变量 varname 的值,若变量没有定义时返回 nil
    export NODE_ID=8888

    每次打开一个终端，都需要设置 NODE_ID 的值。
    变量名 NODE_ID，可以更改别的。
     */

    nodeID :=os.Getenv("NODE_ID")
    if nodeID == ""{
        fmt.Printf("NODE_ID 环境变量没有设置。。\n")
        os.Exit(1)
    }
    fmt.Printf("NODE_ID:%s\n",nodeID)

    //1.创建 flagset 标签对象
    createWalletCmd := flag.NewFlagSet("createwallet", flag.ExitOnError)
    addressListsCmd := flag.NewFlagSet("addresslists",flag.ExitOnError)

    sendBlockCmd := flag.NewFlagSet("send", flag.ExitOnError)
    //fmt.Printf("%T\n",addBlockCmd) //*FlagSet
    printChainCmd := flag.NewFlagSet("printchain", flag.ExitOnError)
    createBlockChainCmd := flag.NewFlagSet("createblockchain", flag.ExitOnError)
    getBalanceCmd := flag.NewFlagSet("getbalance", flag.ExitOnError)

    testCmd:=flag.NewFlagSet("test",flag.ExitOnError)

    startNodeCmd := flag.NewFlagSet("startnode",flag.ExitOnError)

    //2.设置标签后的参数
    //flagAddBlockData:= addBlockCmd.String("data","helloworld..","交易数据")
    flagFromData := sendBlockCmd.String("from", "", "转帐源地址")
    flagToData := sendBlockCmd.String("to", "", "转帐目标地址")
    flagAmountData := sendBlockCmd.String("amount", "", "转帐金额")
    flagCreateBlockChainData := createBlockChainCmd.String("address", "", "创世区块交易地址")
    flagGetBalanceData := getBalanceCmd.String("address", "", "要查询的某个账户的余额")

    flagMiner := startNodeCmd.String("miner","","定义挖矿奖励的地址......")
    flagMine := sendBlockCmd.Bool("mine",false,"是否在当前节点中立即验证....")

    //3.解析
    switch os.Args[1] {
    case "send":
        err := sendBlockCmd.Parse(os.Args[2:])
        if err != nil {
            log.Panic(err)
        }
        //fmt.Println("----",os.Args[2:])

    case "printchain":
        err := printChainCmd.Parse(os.Args[2:])
        if err != nil {
            log.Panic(err)
        }
        //fmt.Println("====",os.Args[2:])

    case "createblockchain":
        err := createBlockChainCmd.Parse(os.Args[2:])
        if err != nil {
            log.Panic(err)
        }
    case "getbalance":
        err := getBalanceCmd.Parse(os.Args[2:])
        if err != nil {
            log.Panic(err)
        }

    case "createwallet":
        err := createWalletCmd.Parse(os.Args[2:])
        if err != nil {
            log.Panic(err)
        }
    case "addresslists":
        err := addressListsCmd.Parse(os.Args[2:])
        if err != nil {
            log.Panic(err)
        }
    case "test":
        err := testCmd.Parse(os.Args[2:])
        if err != nil {
            log.Panic(err)
        }

    case "startnode":
        err := startNodeCmd.Parse(os.Args[2:])
        if err != nil {
            log.Panic(err)
        }

    default:
        printUsage()
        os.Exit(1) //退出
    }

    if sendBlockCmd.Parsed() {
        if *flagFromData == "" || *flagToData == "" || *flagAmountData == "" {
            printUsage()
            os.Exit(1)
        }
        //cli.addBlock([]*Transaction{})
        //fmt.Println(*flagFromData)
        //fmt.Println(*flagToData)
        //fmt.Println(*flagAmountData)
        //fmt.Println(JSONToArray(*flagFrom))
        //fmt.Println(JSONToArray(*flagTo))
        //fmt.Println(JSONToArray(*flagAmount))
        from := JSONToArray(*flagFromData)
        to := JSONToArray(*flagToData)
        amount := JSONToArray(*flagAmountData)

        for i := 0; i < len(from); i++ {
            if !IsValidForAddress([]byte(from[i])) || !IsValidForAddress([]byte(to[i])) {
                fmt.Println("钱包地址无效")
                printUsage()
                os.Exit(1)
            }
        }

        cli.send(from, to, amount,nodeID,*flagMine)
    }
    if printChainCmd.Parsed() {
        cli.printChains(nodeID)
    }

    if createBlockChainCmd.Parsed() {
        //if *flagCreateBlockChainData == "" {
        if !IsValidForAddress([]byte(*flagCreateBlockChainData)){
            fmt.Println("创建地址无效")
            printUsage()
            os.Exit(1)
        }
        cli.createGenesisBlockchain(*flagCreateBlockChainData,nodeID)
    }

    if getBalanceCmd.Parsed() {
        //if *flagGetBalanceData == "" {
        if !IsValidForAddress([]byte(*flagGetBalanceData)){
            fmt.Println("查询地址无效")
            printUsage()
            os.Exit(1)
        }
        cli.getBalance(*flagGetBalanceData,nodeID)

    }

    if createWalletCmd.Parsed() {
        //创建钱包
        cli.createWallet(nodeID)
    }

    //获取所有的钱包地址
    if addressListsCmd.Parsed(){
        cli.addressLists(nodeID)
    }

    if testCmd.Parsed(){
        cli.TestMethod(nodeID)
    }

    if startNodeCmd.Parsed() {
        cli.startNode(nodeID,*flagMiner)
    }

}

func isValidArgs() {
    if len(os.Args) < 2 {
        printUsage()
        os.Exit(1)
    }
}
func printUsage() {
    fmt.Println("Usage:")
    fmt.Println("\tcreatewallet -- 创建钱包")
    fmt.Println("\taddresslists -- 输出所有钱包地址")
    fmt.Println("\tcreateblockchain -address DATA -- 创建创世区块")
    fmt.Println("\tsend -from FROM -to TO -amount AMOUNT -mine -- 交易明细.")
    fmt.Println("\tprintchain - 输出信息:")
    fmt.Println("\tgetbalance -address DATA -- 查询账户余额")
    fmt.Println("\ttest -- 测试")
    fmt.Println("\tstartnode -miner ADDRESS -- 启动节点服务器，并且指定挖矿奖励的地址.")
} 
```

#### 3.2.2 修改`Wallets.go`文件

打开`day08_10_Net`目录里的 BLC 包。修改`Wallets.go`文件。

修改步骤：

```go
修改步骤：
step1：修改 walletFile 变量
step2：修改 NewWallets()方法，添加 NODE_ID
step3：修改 CreateNewWallet()方法，添加 NODE_ID
step4：修改 SaveWallets()方法，添加 NODE_ID 
```

修改完后代码如下：

```go
package BLC

import (
    "fmt"
    "os"
    "io/ioutil"
    "log"
    "encoding/gob"
    "crypto/elliptic"
    "bytes"
)

//1.创建钱包
type Wallets struct {
    WalletsMap map[string]*Wallet
}

//2.创建一个钱包集合
//创建钱包集合:文件中存在从文件中读取，否则新建一个
const walletFile = "Wallets_%s.dat"

func NewWallets(nodeID string) *Wallets {
    //wallets := &WalletsMap{}
    //wallets.WalletsMap = make(map[string]*Wallet)
    //return wallets

    walletFile := fmt.Sprintf(walletFile,nodeID)

    //1.判断钱包文件是否存在
    if _, err := os.Stat(walletFile); os.IsNotExist(err) {
        fmt.Println("文件不存在")
        wallets := &Wallets{}
        wallets.WalletsMap = make(map[string]*Wallet)
        return wallets
    }
    //2.否则读取文件中的数据
    fileContent, err := ioutil.ReadFile(walletFile)
    if err != nil {
        log.Panic(err)
    }
    var wallets Wallets
    gob.Register(elliptic.P256())
    decoder := gob.NewDecoder(bytes.NewReader(fileContent))
    err = decoder.Decode(&wallets)
    if err != nil {
        log.Panic(err)
    }
    return &wallets
}

//3.创建一个新钱包
func (ws *Wallets) CreateNewWallet(nodeID string) {
    wallet := NewWallet()
    fmt.Printf("创建钱包地址：%s\n", wallet.GetAddress())
    ws.WalletsMap[string(wallet.GetAddress())] = wallet

    //将钱包保存
    ws.SaveWallets(nodeID)
}

/*
要让数据对象能在网络上传输或存储，我们需要进行编码和解码。现在比较流行的编码方式有 JSON,XML 等。然而，Go 在 gob 包中为我们提供了另一种方式，该方式编解码效率高于 JSON。
gob 是 Golang 包自带的一个数据结构序列化的编码/解码工具
 */
func (ws *Wallets) SaveWallets(nodeID string) {

    walletFile := fmt.Sprintf(walletFile,nodeID)

    var content bytes.Buffer
    //注册的目的，为了可以序列化任何类型，wallet 结构体中有接口类型。将接口进行注册
    gob.Register(elliptic.P256()) //gob 是 Golang 包自带的一个数据结构序列化的编码/解码工具
    encoder := gob.NewEncoder(&content)
    err := encoder.Encode(ws)
    if err != nil {
        log.Panic(err)
    }
    //将序列化后的数据写入到文件，原来的文件中的内容会被覆盖掉
    err = ioutil.WriteFile(walletFile, content.Bytes(), 0644)
    if err != nil {
        log.Panic(err)
    }
} 
```

#### 3.2.3 修改`CLI_getBalance`文件

打开`day08_10_Net`目录里的 BLC 包。修改`Wallets.go`文件。

修改步骤：

```go
修改步骤：
step1：修改 getBalance()方法，添加 NODE_ID 
```

修改完后代码如下：

```go
package BLC

import (
    "fmt"
    "os"
)

//查询余额
func (cli *CLI)getBalance(address string,nodeID string){
    fmt.Println("查询余额：",address)
    bc := GetBlockchainObject(nodeID)

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

#### 3.2.4 修改`CLI_printChains.go`文件

打开`day08_10_Net`目录里的 BLC 包。修改`CLI_printChains.go`文件。

修改步骤：

```go
修改步骤：
step1：修改 printChains()方法，添加 NODE_ID 
```

修改完后代码如下：

```go
package BLC

import (
    "fmt"
    "os"
)

func (cli *CLI)printChains(nodeID string){
    bc:=GetBlockchainObject(nodeID)
    if bc == nil{
        fmt.Println("没有区块可以打印。。")
        os.Exit(1)
    }
    defer bc.DB.Close()
    bc.PrintChains()
} 
```

#### 3.2.5 修改`CLI_createBlockChain.go`文件

打开`day08_10_Net`目录里的 BLC 包。修改`CLI_createBlockChain.go`文件。

修改步骤：

```go
修改步骤：
step1：修改 createGenesisBlockchain()方法，添加 NODE_ID 
```

修改完后代码如下：

```go
package BLC

func (cli *CLI) createGenesisBlockchain(address string,nodeID string){
    //fmt.Println(data)
    CreateBlockChainWithGenesisBlock(address,nodeID)

    bc:=GetBlockchainObject(nodeID)
    defer bc.DB.Close()
    if bc != nil{
        utxoSet:=&UTXOSet{bc}
        utxoSet.ResetUTXOSet()
    }

} 
```

#### 3.2.6 修改`CLI_testmethod.go`文件

打开`day08_10_Net`目录里的 BLC 包。修改`CLI_testmethod.go`文件。

修改步骤：

```go
修改步骤：
step1：修改 TestMethod()方法，添加 NODE_ID 
```

修改完后代码如下：

```go
package BLC

import "fmt"

func (cli *CLI) TestMethod(nodeID string){
    blockchain:=GetBlockchainObject(nodeID)
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

#### 3.2.7 修改`CLI_addresslists.go`文件

打开`day08_10_Net`目录里的 BLC 包。修改`CLI_addresslists.go`文件。

修改步骤：

```go
修改步骤：
step1：修改 addressLists()方法，添加 NODE_ID 
```

修改完后代码如下：

```go
 package BLC

import "fmt"

func (cli *CLI)addressLists(nodeID string){
    fmt.Println("打印所有的钱包地址。。")
    //获取
    Wallets:=NewWallets(nodeID)
    for address,_ := range Wallets.WalletsMap{
        fmt.Println("address:",address)
    }
} 
```

#### 3.2.8 修改`CLI_send.go`文件

打开`day08_10_Net`目录里的 BLC 包。修改`CLI_send.go`文件。

修改步骤：

```go
修改步骤：
step1：修改 send()方法，添加 NODE_ID 
```

修改完后代码如下：

```go
package BLC

import (
    "strconv"
    "fmt"
)

// 转账
func (cli *CLI) send(from []string, to []string, amount []string, nodeID string, mineNow bool) {

    blockchain := GetBlockchainObject(nodeID)
    utxoSet := &UTXOSet{blockchain}
    defer blockchain.DB.Close()

    if mineNow {
        blockchain.MineNewBlock(from, to, amount, nodeID)

        //转账成功以后，需要更新一下
        utxoSet.Update()
    } else {
        // 把交易发送到矿工节点去进行验证
        fmt.Println("由矿工节点处理......")
        value, _ := strconv.Atoi(amount[0])
        tx := NewSimpleTransaction(from[0], to[0], int64(value), utxoSet, []*Transaction{}, nodeID)

        sendTx(knowNodes[0], tx)
    }

} 
```

#### 3.2.9 修改`Transaction.go`文件

打开`day08_10_Net`目录里的 BLC 包。修改`Transaction.go`文件。

修改步骤：

```go
修改步骤：
step1：修改 NewSimpleTransaction()方法，添加 NODE_ID 
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
    "encoding/json"
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

func NewSimpleTransaction(from, to string, amount int64, utxoSet *UTXOSet, txs []*Transaction,nodeID string) *Transaction {
    var txInputs [] *TXInput
    var txOutputs [] *TXOuput

    //balance, spendableUTXO := bc.FindSpendableUTXOs(from, amount, txs)
    balance, spendableUTXO := utxoSet.FindSpendableUTXOs(from, amount, txs)

    //代表消费

    //txInput := &TXInput{bytes, 0, from}
    //txInputs = append(txInputs, txInput)

    //获取钱包
    wallets := NewWallets(nodeID)
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

#### 3.2.10 修改`BlockChain.go`文件

打开`day08_10_Net`目录里的 BLC 包。修改`BlockChain.go`文件。

修改步骤：

```go
修改步骤：
step1：修改 dbExists()方法
step2：修改 CreateBlockchainWithGenesisBlock()，添加 NODE_ID
step3：修改 GetBlockchainObject()，添加 NODE_ID
step4：添加 GetBestHeight(),用于获取区块的高度
step5：添加 GetBlockHashes(),获取区块的 hash
step6：添加 GetBlock()，获取区块
step7：添加 AddBlock(),添加区块 
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
func CreateBlockChainWithGenesisBlock(address string,nodeID string) {
    /*
    格式化数据库的名字
        1.修改数据库的名字："blockchain_%s.db"
        2.根据节点生成数据库的名字

     */
    DBNAME:= fmt.Sprintf(DBNAME,nodeID)

    if dbExists(DBNAME) {
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

//提供一个方法，用于判断数据库是否存在
func dbExists(DBName string) bool {
    if _, err := os.Stat(DBName); os.IsNotExist(err) {
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
func GetBlockchainObject(nodeID string) *BlockChain {
    DBNAME:= fmt.Sprintf(DBNAME,nodeID)

    /*
    1.如果数据库不存在，直接返回 nil
    2.读取数据库
     */
    if !dbExists(DBNAME) {
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
func (bc *BlockChain) MineNewBlock(from, to, amount []string,nodeID string) {
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
        tx := NewSimpleTransaction(from[i], to[i], amountInt, utxoSet, txs,nodeID)

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

//----------
//获取最新区块的高度
func (bc *BlockChain) GetBestHeight() int64 {

    block := bc.Iterator().Next()

    return block.Height
}

//获取所有区块的 hash
func (bc *BlockChain) GetBlockHashes() [][]byte {

    blockIterator := bc.Iterator()

    var blockHashs [][]byte

    for {
        block := blockIterator.Next()

        blockHashs = append(blockHashs,block.Hash)

        var hashInt big.Int
        hashInt.SetBytes(block.PrevBlockHash)

        if hashInt.Cmp(big.NewInt(0)) == 0 {
            break;
        }
    }

    return blockHashs
}

//根据 hash 获取区块
func (bc *BlockChain) GetBlock(blockHash []byte) ([]byte ,error) {

    var blockBytes []byte

    err := bc.DB.View(func(tx *bolt.Tx) error {

        b := tx.Bucket([]byte(BLOCKTABLENAME))

        if b != nil {

            blockBytes = b.Get(blockHash)

        }

        return nil
    })

    return blockBytes,err
}

//添加区块到数据库
func (bc *BlockChain) AddBlock(block *Block)  {

    err := bc.DB.Update(func(tx *bolt.Tx) error {

        b := tx.Bucket([]byte(BLOCKTABLENAME))

        if b != nil {

            blockExist := b.Get(block.Hash)

            if blockExist != nil {
                // 如果存在，不需要做任何过多的处理
                return nil
            }

            err := b.Put(block.Hash,block.Serilalize())

            if err != nil {
                log.Panic(err)
            }

            // 最新的区块链的 Hash
            blockHash := b.Get([]byte("l"))

            blockBytes := b.Get(blockHash)

            blockInDB := DeserializeBlock(blockBytes)

            if blockInDB.Height < block.Height {

                b.Put([]byte("l"),block.Hash)
                bc.Tip = block.Hash
            }
        }

        return nil
    })

    if err != nil {
        log.Panic(err)
    }
} 
```

#### 3.2.11 新建`Server_var.go`文件

打开`day08_10_Net`目录里的 BLC 包。修改`Server_var.go`文件。并添加代码如下:

```go
package BLC

//存储节点全局变量

//localhost:3000 主节点的地址
var knowNodes = []string{"localhost:3000"}
var nodeAddress string //全局变量，节点地址
// 存储 hash 值
var transactionArray [][]byte
var minerAddress string
var memoryTxPool = make(map[string]*Transaction) 
```

#### 3.2.12 新建`Server_Version.go`文件

打开`day08_10_Net`目录里的 BLC 包。修改`Server_Version.go`文件。并添加代码如下:

```go
package BLC

type Version struct {
    Version    int64 // 版本
    BestHeight int64 // 当前节点区块的高度
    AddrFrom   string //当前节点的地址
} 
```

#### 3.2.13 新建`Server.go`文件

打开`day08_10_Net`目录里的 BLC 包。修改`Server.go`文件。并添加代码如下:

```go
package BLC

import (
    "fmt"
    "net"
    "log"
    "io/ioutil"
)

func startServer(nodeID string, minerAdd string) {
    //""
    // 当前节点的 IP 地址
    nodeAddress = fmt.Sprintf("localhost:%s", nodeID)

    minerAddress = minerAdd

    fmt.Printf("nodeAddress:%s,minerAddress:%s\n",nodeAddress,minerAddress)
    ln, err := net.Listen(PROTOCOL, nodeAddress)

    if err != nil {
        log.Panic(err)
    }

    defer ln.Close()

    bc := GetBlockchainObject(nodeID)

    //defer bc.DB.Close()

    // 第一个终端：端口为 3000,启动的就是主节点
    // 第二个终端：端口为 3001，钱包节点
    // 第三个终端：端口号为 3002，矿工节点
    if nodeAddress != knowNodes[0] {
        // 此节点是钱包节点或者矿工节点，需要向主节点发送请求同步数据
        fmt.Printf("knowNodes:%s\n",knowNodes[0])
        sendVersion(knowNodes[0], bc)
    }

    for {
        // 收到的数据的格式是固定的，12 字节+结构体字节数组

        // 接收客户端发送过来的数据
        conn, err := ln.Accept()
        if err != nil {
            log.Panic(err)
        }

        go handleConnection(conn, bc)

    }

}

func handleConnection(conn net.Conn, bc *BlockChain) {

    // 读取客户端发送过来的所有的数据
    request, err := ioutil.ReadAll(conn)
    if err != nil {
        log.Panic(err)
    }

    fmt.Printf("Receive a Message:%s\n", request[:COMMANDLENGTH])

    //version
    command := bytesToCommand(request[:COMMANDLENGTH])

    // 12 字节 + 某个结构体序列化以后的字节数组

    switch command {
    case COMMAND_VERSION:
        handleVersion(request, bc)

    case COMMAND_GETBLOCKS:
        handleGetblocks(request, bc)

    case COMMAND_INV:
        handleInv(request, bc)

    case COMMAND_ADDR:
        handleAddr(request, bc)
    case COMMAND_BLOCK:
        handleBlock(request, bc)

    case COMMAND_GETDATA:
        handleGetData(request, bc)

    case COMMAND_TX:
        handleTx(request, bc)
    default:
        fmt.Println("Unknown command!")
    }

    conn.Close()
}

func nodeIsKnown(addr string) bool {
    for _, node := range knowNodes {
        if node == addr {
            return true
        }
    }

    return false
} 
```

#### 3.2.14 新建`Server_handle.go`文件

打开`day08_10_Net`目录里的 BLC 包。修改`Server_handle.go`文件。并添加代码如下:

```go
package BLC

import (
    "bytes"
    "log"
    "encoding/gob"
    "fmt"
    "encoding/hex"
    "github.com/boltdb/bolt"
)

func handleVersion(request []byte, bc *BlockChain) {

    var buff bytes.Buffer
    var payload Version

    dataBytes := request[COMMANDLENGTH:]

    // 反序列化
    buff.Write(dataBytes)
    dec := gob.NewDecoder(&buff)
    err := dec.Decode(&payload)
    if err != nil {
        log.Panic(err)
    }

    //Version
    //1\. Version
    //2\. BestHeight
    //3\. 节点地址

    bestHeight := bc.GetBestHeight()          //3 1
    foreignerBestHeight := payload.BestHeight // 1 3

    if bestHeight > foreignerBestHeight {
        sendVersion(payload.AddrFrom, bc)
    } else if bestHeight < foreignerBestHeight {
        // 去向主节点要信息
        sendGetBlocks(payload.AddrFrom)
    }

    if !nodeIsKnown(payload.AddrFrom) {
        knowNodes = append(knowNodes, payload.AddrFrom)
    }

}

func handleAddr(request []byte, bc *BlockChain) {

}

func handleGetblocks(request []byte, bc *BlockChain) {

    var buff bytes.Buffer
    var payload GetBlocks

    dataBytes := request[COMMANDLENGTH:]

    // 反序列化
    buff.Write(dataBytes)
    dec := gob.NewDecoder(&buff)
    err := dec.Decode(&payload)
    if err != nil {
        log.Panic(err)
    }

    blocks := bc.GetBlockHashes()

    //txHash blockHash
    sendInv(payload.AddrFrom, BLOCK_TYPE, blocks)

}

func handleInv(request []byte, bc *BlockChain) {

    var buff bytes.Buffer
    var payload Inv

    dataBytes := request[COMMANDLENGTH:]

    // 反序列化
    buff.Write(dataBytes)
    dec := gob.NewDecoder(&buff)
    err := dec.Decode(&payload)
    if err != nil {
        log.Panic(err)
    }

    // Ivn 3000 block hashes [][]

    if payload.Type == BLOCK_TYPE {

        //tansactionArray = payload.Items

        //payload.Items

        blockHash := payload.Items[0]
        sendGetData(payload.AddrFrom, BLOCK_TYPE, blockHash)

        if len(payload.Items) >= 1 {
            transactionArray = payload.Items[1:]
        }
    }

    if payload.Type == TX_TYPE {

        txHash := payload.Items[0]
        if memoryTxPool[hex.EncodeToString(txHash)] == nil {
            sendGetData(payload.AddrFrom, TX_TYPE, txHash)
        }

    }

}

func handleGetData(request []byte, bc *BlockChain) {

    var buff bytes.Buffer
    var payload GetData

    dataBytes := request[COMMANDLENGTH:]

    // 反序列化
    buff.Write(dataBytes)
    dec := gob.NewDecoder(&buff)
    err := dec.Decode(&payload)
    if err != nil {
        log.Panic(err)
    }

    if payload.Type == BLOCK_TYPE {

        block, err := bc.GetBlock([]byte(payload.Hash))
        if err != nil {
            return
        }

        sendBlock(payload.AddrFrom, block)
    }

    if payload.Type == TX_TYPE {

        tx := memoryTxPool[hex.EncodeToString(payload.Hash)]

        sendTx(payload.AddrFrom, tx)

    }
}

func handleBlock(request []byte, bc *BlockChain) {
    var buff bytes.Buffer
    var payload BlockData

    dataBytes := request[COMMANDLENGTH:]

    // 反序列化
    buff.Write(dataBytes)
    dec := gob.NewDecoder(&buff)
    err := dec.Decode(&payload)
    if err != nil {
        log.Panic(err)
    }

    blockBytes := payload.Block

    block := DeserializeBlock(blockBytes)

    fmt.Println("Recevied a new block!")
    bc.AddBlock(block)
    UTXOSet := &UTXOSet{bc}
    UTXOSet.Update()

    fmt.Printf("Added block %x\n", block.Hash)

    if len(transactionArray) > 0 {
        blockHash := transactionArray[0]
        sendGetData(payload.AddrFrom, "block", blockHash)

        transactionArray = transactionArray[1:]
    } else {

        //fmt.Println("数据库重置......")
        //UTXOSet := &UTXOSet{bc}
        //UTXOSet.ResetUTXOSet()

    }

}

func handleTx(request []byte, bc *BlockChain) {

    var buff bytes.Buffer
    var payload Tx

    dataBytes := request[COMMANDLENGTH:]

    // 反序列化
    buff.Write(dataBytes)
    dec := gob.NewDecoder(&buff)
    err := dec.Decode(&payload)
    if err != nil {
        log.Panic(err)
    }

    //-----

    tx := payload.Tx
    memoryTxPool[hex.EncodeToString(tx.TxID)] = tx

    // 说明主节点自己
    if nodeAddress == knowNodes[0] {
        // 给矿工节点发送交易 hash
        for _, nodeAddr := range knowNodes {

            if nodeAddr != nodeAddress && nodeAddr != payload.AddrFrom {
                sendInv(nodeAddr, TX_TYPE, [][]byte{tx.TxID})
            }
        }
    }

    // 矿工进行挖矿验证
    // "" | 1DVFvyCK8qTQkLBTZ5fkh5eDSbcZVoHAsj
    if len(memoryTxPool) >= 1 && len(minerAddress) > 0 {

    MineTransactions:

        utxoSet := &UTXOSet{bc}

        txs := []*Transaction{tx}

        //奖励
        coinbaseTx := NewCoinBaseTransaction(minerAddress)
        txs = append(txs, coinbaseTx)

        _txs := []*Transaction{}

        //fmt.Println("开始进行数字签名验证.....")

        for _, tx := range txs {

            //fmt.Printf("开始第%d 次验证...\n",index)

            // 数字签名失败
            if bc.VerifyTransaction(tx, _txs) != true {
                log.Panic("ERROR: Invalid transaction")
            }

            //fmt.Printf("第%d 次验证成功\n",index)
            _txs = append(_txs, tx)
        }

        //fmt.Println("数字签名验证成功.....")

        //1\. 通过相关算法建立 Transaction 数组
        var block *Block

        bc.DB.View(func(tx *bolt.Tx) error {

            b := tx.Bucket([]byte(BLOCKTABLENAME))
            if b != nil {

                hash := b.Get([]byte("l"))

                blockBytes := b.Get(hash)

                block = DeserializeBlock(blockBytes)

            }

            return nil
        })

        //2\. 建立新的区块
        block = NewBlock(txs, block.Hash, block.Height+1)

        //将新区块存储到数据库
        bc.DB.Update(func(tx *bolt.Tx) error {
            b := tx.Bucket([]byte(BLOCKTABLENAME))
            if b != nil {

                b.Put(block.Hash, block.Serilalize())

                b.Put([]byte("l"), block.Hash)

                bc.Tip = block.Hash

            }
            return nil
        })
        utxoSet.Update()
        sendBlock(knowNodes[0], block.Serilalize())

        for _, tx := range txs {
            txID := hex.EncodeToString(tx.TxID)
            delete(memoryTxPool, txID)
        }
        for _, node := range knowNodes {
            if node != nodeAddress {
                sendInv(node, "block", [][]byte{block.Hash})
            }
        }

        if len(memoryTxPool) > 0 {
            goto MineTransactions
        }
    }
} 
```

#### 3.2.15 新建`Server_send.go`文件

打开`day08_10_Net`目录里的 BLC 包。修改`Server_send.go`文件。并添加代码如下:

```go
package BLC

import (
    "io"
    "bytes"
    "log"
    "net"
    "fmt"
)

//COMMAND_VERSION
func sendVersion(toAddress string,bc *BlockChain)  {

    bestHeight := bc.GetBestHeight()

    payload := gobEncode(Version{NODE_VERSION, bestHeight, nodeAddress})

    //version
    request := append(commandToBytes(COMMAND_VERSION), payload...)

    sendData(toAddress,request)

}

//COMMAND_GETBLOCKS
func sendGetBlocks(toAddress string)  {

    payload := gobEncode(GetBlocks{nodeAddress})

    request := append(commandToBytes(COMMAND_GETBLOCKS), payload...)

    fmt.Printf("toAddress:%s",toAddress)
    sendData(toAddress,request)

}

// 主节点将自己的所有的区块 hash 发送给钱包节点
//COMMAND_BLOCK
//
func sendInv(toAddress string, kind string, hashes [][]byte) {

    payload := gobEncode(Inv{nodeAddress,kind,hashes})

    request := append(commandToBytes(COMMAND_INV), payload...)

    sendData(toAddress,request)

}

func sendGetData(toAddress string, kind string ,blockHash []byte) {

    payload := gobEncode(GetData{nodeAddress,kind,blockHash})

    request := append(commandToBytes(COMMAND_GETDATA), payload...)

    sendData(toAddress,request)
}

func sendBlock(toAddress string, block []byte)  {

    payload := gobEncode(BlockData{nodeAddress,block})

    request := append(commandToBytes(COMMAND_BLOCK), payload...)

    sendData(toAddress,request)

}

func sendTx(toAddress string,tx *Transaction)  {

    payload := gobEncode(Tx{nodeAddress,tx})

    request := append(commandToBytes(COMMAND_TX), payload...)

    sendData(toAddress,request)

}

func sendData(to string,data []byte)  {

    conn, err := net.Dial(PROTOCOL, to)
    fmt.Println(err)
    if err != nil {
        panic("error")
    }
    defer conn.Close()

    // 附带要发送的数据
    _, err = io.Copy(conn, bytes.NewReader(data))
    if err != nil {
        log.Panic(err)
    }
} 
```

#### 3.2.16 新建`Server_getblocks.go`文件

打开`day08_10_Net`目录里的 BLC 包。修改`Server_getblocks.go`文件。并添加代码如下:

```go
package BLC

//getblocks 意为 “给我看一下你有什么区块”（在比特币中，这会更加复杂）
type GetBlocks struct {
    AddrFrom string
} 
```

#### 3.2.17 新建`Server_Inv.go`文件

打开`day08_10_Net`目录里的 BLC 包。修改`Server_Inv.go`文件。并添加代码如下:

```go
package BLC

type Inv struct {
    AddrFrom string  //自己的地址
    Type     string  //类型 block tx
    Items    [][]byte //hash 二维数组
} 
```

#### 3.2.18 新建`Server_Inv.go`文件

打开`day08_10_Net`目录里的 BLC 包。修改`Server_Inv.go`文件。并添加代码如下:

```go
package BLC

type Inv struct {
    AddrFrom string  //自己的地址
    Type     string  //类型 block tx
    Items    [][]byte //hash 二维数组
} 
```

#### 3.2.19 新建`Server_GetData.go`文件

打开`day08_10_Net`目录里的 BLC 包。修改`Server_GetData.go`文件。并添加代码如下:

```go
package BLC
//用于某个块或交易的请求，它可以仅包含一个块或交易的 ID。
type GetData struct {
    AddrFrom string
    Type     string
    Hash       []byte
} 
```

#### 3.2.20 新建`Server_block.go`文件

打开`day08_10_Net`目录里的 BLC 包。修改`Server_block.go`文件。并添加代码如下:

```go
package BLC

type BlockData struct {
    AddrFrom string
    Block []byte
} 
```

#### 3.2.21 新建`Server_Tx.go`文件

打开`day08_10_Net`目录里的 BLC 包。修改`Server_Tx.go`文件。并添加代码如下:

```go
package BLC

type Tx struct {
    AddrFrom string
    Tx *Transaction
} 
```

## 4\. 网络讲解

### 4.1 区块链网络

区块链网络是去中心化的，这意味着没有服务器，客户端也不需要依赖服务器来获取或处理数据。在区块链网络中，有的是节点，每个节点是网络的一个完全（full-fledged）成员。节点就是一切：它既是一个客户端，也是一个服务器。这一点需要牢记于心，因为这与传统的网页应用非常不同。

区块链网络是一个 P2P（Peer-to-Peer，端到端）的网络，即节点直接连接到其他节点。它的拓扑是扁平的，因为在节点的世界中没有层级之分。下面是它的示意图：

![`img.kongyixueyuan.com/1108_p2p.png`](img/e596235fd74ec651b91feec11013f591.jpg)

要实现这样一个网络节点更加困难，因为它们必须执行很多操作。每个节点必须与很多其他节点进行交互，它必须请求其他节点的状态，与自己的状态进行比较，当状态过时时进行更新。

### 4.2 节点角色

尽管节点具有完备成熟的属性，但是它们也可以在网络中扮演不同角色。比如：

1.  矿工 这样的节点运行于强大或专用的硬件（比如 ASIC）之上，它们唯一的目标是，尽可能快地挖出新块。矿工是区块链中唯一可能会用到工作量证明的角色，因为挖矿实际上意味着解决 PoW 难题。在权益证明 PoS 的区块链中，没有挖矿。
2.  全节点 这些节点验证矿工挖出来的块的有效性，并对交易进行确认。为此，他们必须拥有区块链的完整拷贝。同时，全节点执行路由操作，帮助其他节点发现彼此。对于网络来说，非常重要的一段就是要有足够多的全节点。因为正是这些节点执行了决策功能：他们决定了一个块或一笔交易的有效性。
3.  SPV SPV 表示 Simplified Payment Verification，简单支付验证。这些节点并不存储整个区块链副本，但是仍然能够对交易进行验证（不过不是验证全部交易，而是一个交易子集，比如，发送到某个指定地址的交易）。一个 SPV 节点依赖一个全节点来获取数据，可能有多个 SPV 节点连接到一个全节点。SPV 使得钱包应用成为可能：一个人不需要下载整个区块链，但是仍能够验证他的交易。

### 4.3 网络简化

为了在目前的区块链原型中实现网络，我们不得不简化一些事情。因为我们没有那么多的计算机来模拟一个多节点的网络。当然，我们可以使用虚拟机或是 Docker 来解决这个问题，但是这会使一切都变得更复杂：你将不得不先解决可能出现的虚拟机或 Docker 问题，而我的目标是将全部精力都放在区块链实现上。所以，我们想要在一台机器上运行多个区块链节点，同时希望它们有不同的地址。为了实现这一点，我们将使用**端口号作为节点标识符**，而不是使用 IP 地址，比如将会有这样地址的节点：**127.0.0.1:3000**，**127.0.0.1:3001**，**127.0.0.1:3002** 等等。我们叫它端口节点（port node） ID，并使用环境变量 `NODE_ID` 对它们进行设置。故而，你可以打开多个终端窗口，设置不同的 `NODE_ID` 运行不同的节点。

这个方法也需要有不同的区块链和钱包文件。它们现在必须依赖于节点 ID 进行命名，比如 `blockchain_3000.db`, `blockchain_30001.db` , `wallet_3000.db`, `wallet_30001.db` 等等。

所以，当你下载 Bitcoin Core 并首次运行时，到底发生了什么呢？它必须连接到某个节点下载最新状态的区块链。考虑到你的电脑并没有意识到所有或是部分的比特币节点，那么连接到的“某个节点”到底是什么？

在 Bitcoin Core 中硬编码一个地址，已经被证实是一个错误：因为节点可能会被攻击或关机，这会导致新的节点无法加入到网络中。在 Bitcoin Core 中，硬编码了 [DNS seeds](https://bitcoin.org/en/glossary/dns-seed)。虽然这些并不是节点，但是 DNS 服务器知道一些节点的地址。当你启动一个全新的 Bitcoin Core 时，它会连接到一个种子节点，获取全节点列表，随后从这些节点中下载区块链。

不过在我们目前的实现中，无法做到完全的去中心化，因为会出现中心化的特点。我们会有三个节点：

1.  一个中心节点。所有其他节点都会连接到这个节点，这个节点会在其他节点之间发送数据。
2.  一个矿工节点。这个节点会在内存池中存储新的交易，当有足够的交易时，它就会打包挖出一个新块。
3.  一个钱包节点。这个节点会被用作在钱包之间发送币。但是与 SPV 节点不同，它存储了区块链的一个完整副本。

### 4.4 场景

本文的目标是实现如下场景：

1.  中心节点创建一个区块链。
2.  一个其他（钱包）节点连接到中心节点并下载区块链。
3.  另一个（矿工）节点连接到中心节点并下载区块链。
4.  钱包节点创建一笔交易。
5.  矿工节点接收交易，并将交易保存到内存池中。
6.  当内存池中有足够的交易时，矿工开始挖一个新块。
7.  当挖出一个新块后，将其发送到中心节点。
8.  钱包节点与中心节点进行同步。
9.  钱包节点的用户检查他们的支付是否成功。

这就是比特币中的一般流程。尽管我们不会实现一个真实的 P2P 网络，但是我们会实现一个真实，也是比特币最常见最重要的用户场景。

##

### 4.5 代码实现

#### 4.5.1 设置`NODE_ID`

1.如何设置`NODE_ID`

我们可以通过`os.Getenv()`方法获取 环境变量的值。

在 BLC 包下，修改`CLI.go`文件中的`Run()`方法，代码如下：

```go
//step2：添加 Run 方法
func (cli *CLI) Run() {
    //判断命令行参数的长度
    isValidArgs()

    /*
    获取节点 ID
    解释：返回当前进程的环境变量 varname 的值,若变量没有定义时返回 nil
    export NODE_ID=8888

    每次打开一个终端，都需要设置 NODE_ID 的值。
    变量名 NODE_ID，可以更改别的。
     */

    nodeID :=os.Getenv("NODE_ID")
    if nodeID == ""{
        fmt.Printf("NODE_ID 环境变量没有设置。。\n")
        os.Exit(1)
    }
    fmt.Printf("NODE_ID:%s\n",nodeID)

    //1.创建 flagset 标签对象
    createWalletCmd := flag.NewFlagSet("createwallet", flag.ExitOnError)
    addressListsCmd := flag.NewFlagSet("addresslists",flag.ExitOnError)

    ...

} 
```

接下来我们测试一下`NODE_ID`，首先打开一个终端，输入以下命令：

```go
hanru:mypublicchain ruby$ cd day08_10_Net/
hanru:day08_10_Net ruby$ go build -o bc main.go
hanru:day08_10_Net ruby$ export NODE_ID=9527
hanru:day08_10_Net ruby$ ./bc haha 
```

运行结果如下：

![`img.kongyixueyuan.com/1103_nodeid.png`](img/1402a5d854e713a37d2bb814ef8a158a.jpg)

#### 4.5.2 更新项目中结合`NODE_ID`

现在我们已经在程序中设置了`NODE_ID`，接下来我们需要调整程序，使用`NODE_ID`模拟不同的节点，分别创建自己的数据库文件和钱包文件等。我们需要一点一点修改：

首先修改数据库的名字：

打开 BLC 包下，`Constant.go`文件，修改如下：

```go
package BLC

const DBNAME = "blockchain_%s.db"  //数据库名
const BLOCKTABLENAME = "blocks" //表名 
```

我们可以利用`fmt`包下的`func Sprintf(format string, a ...interface{}) string`方法，来动态设置数据库文件名称和钱包文件名称。

**1\. 接下来修改创建创世区块的功能：**

step1：修改`CLI.go 中的 Run()`方法，修改`cli.createGenesisBlockchain(*flagCreateBlockChainData,nodeID)`，添加`nodeID`，代码如下：

```go
func (cli *CLI) Run() {
    ...
    if createBlockChainCmd.Parsed() {
            //if *flagCreateBlockChainData == "" {
            if !IsValidForAddress([]byte(*flagCreateBlockChainData)){
                fmt.Println("创建地址无效")
                printUsage()
                os.Exit(1)
            }
            cli.createGenesisBlockchain(*flagCreateBlockChainData,nodeID)
       }
    ...
} 
```

step2：修改`CLI_createBlockChain.go`文件中，修改方法声明，添加`nodeID`，代码如下：

```go
package BLC

func (cli *CLI) createGenesisBlockchain(address string,nodeID string){
    //fmt.Println(data)
    CreateBlockChainWithGenesisBlock(address,nodeID)

    bc:=GetBlockchainObject(nodeID)
    defer bc.DB.Close()
    if bc != nil{
        utxoSet:=&UTXOSet{bc}
        utxoSet.ResetUTXOSet()
    }

} 
```

step3：修改`BlockChain.go`文件中的`dbExists()`方法，用于判断给定的数据库是否存在，修改后代码如下：

```go
//提供一个方法，用于判断数据库是否存在
func dbExists(DBName string) bool {
    if _, err := os.Stat(DBName); os.IsNotExist(err) {
        return false
    }
    return true
} 
```

step4：接下来修改`BlockChain.go`文件中的`CreateBlockChainWithGenesisBlock()`方法，添加`nodeID`，修改后代码如下：

```go
//修改该方法
/*
1.仅仅用来创建区块链
如果数据库存在，证明区块链存在，直接结束该方法
否则进行创建创世区块，并存入数据库中
 */
func CreateBlockChainWithGenesisBlock(address string,nodeID string) {
    /*
    格式化数据库的名字
        1.修改数据库的名字："blockchain_%s.db"
        2.根据节点生成数据库的名字

     */
    DBNAME:= fmt.Sprintf(DBNAME,nodeID)

    if dbExists(DBNAME) {
        fmt.Println("数据库已经存在。。。")
        return
    }

    //
    fmt.Println("创建创世区块：")
    //2.数据库不存在，说明第一次创建，然后存入到数据库中
    fmt.Println("数据库不存在。。")

    ...
} 
```

step5：修改`BlockChain.go`文件中的`GetBlockchainObject()`方法，添加`nodeID`，修改后代码如下：

```go
//新增方法，用于获取区块链
func GetBlockchainObject(nodeID string) *BlockChain {
    DBNAME:= fmt.Sprintf(DBNAME,nodeID)

    /*
    1.如果数据库不存在，直接返回 nil
    2.读取数据库
     */
    if !dbExists(DBNAME) {
        fmt.Println("数据库不存在，无法获取区块链。。")
        return nil
    }
    db, err := bolt.Open(DBNAME, 0600, nil)
    if err != nil {
        log.Fatal(err)
    }
    ...
} 
```

**2\. 修改查询余额功能：**

step1：修改`CLI.go`中的`Run()`方法，修改`cli.getBalance(*flagGetBalanceData,nodeID)`，添加`nodeID`，代码如下：

```go
func (cli *CLI) Run() {
    ...
    if getBalanceCmd.Parsed() {
        //if *flagGetBalanceData == "" {
        if !IsValidForAddress([]byte(*flagGetBalanceData)){
            fmt.Println("查询地址无效")
            printUsage()
            os.Exit(1)
        }
        cli.getBalance(*flagGetBalanceData,nodeID)

    }
    ...
} 
```

step2：修改`CLI_getBalance.go`文件中，修改`getBalance()`方法声明，添加`nodeID`，代码如下：

```go
package BLC

import (
    "fmt"
    "os"
)

//查询余额
func (cli *CLI)getBalance(address string,nodeID string){
    fmt.Println("查询余额：",address)
    bc := GetBlockchainObject(nodeID)

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

**3\. 修改打印区块功能：**

step1：修改`CLI.go`中的`Run()`方法，修改`cli.printChains(nodeID)`，添加`nodeID`，代码如下：

```go
func (cli *CLI) Run() {
    ...
    if printChainCmd.Parsed() {
        cli.printChains(nodeID)
    }
    ...
} 
```

step2：修改`CLI_printChains.go`文件中，修改`printChains()`方法声明，添加`nodeID`，代码如下：

```go
package BLC

import (
    "fmt"
    "os"
)

func (cli *CLI)printChains(nodeID string){
    bc:=GetBlockchainObject(nodeID)
    if bc == nil{
        fmt.Println("没有区块可以打印。。")
        os.Exit(1)
    }
    defer bc.DB.Close()
    bc.PrintChains()
} 
```

**4\. 修改测试方法：**

step1：修改`CLI.go`中的`Run()`方法，修改`cli.TestMethod(nodeID)`，添加`nodeID`，代码如下：

```go
func (cli *CLI) Run() {
    ...
    if testCmd.Parsed(){
        cli.TestMethod(nodeID)
    }
    ...
} 
```

step2：修改`CLI_testmethod.go`文件中，修改`TestMethod()`方法声明，添加`nodeID`，代码如下：

```go
package BLC

import "fmt"

func (cli *CLI) TestMethod(nodeID string){
    blockchain:=GetBlockchainObject(nodeID)
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

**5\. 修改创建钱包功能：**

step1：修改`CLI.go`中的`Run()`方法，修改`cli.createWallet(nodeID)`，添加`nodeID`，代码如下：

```go
func (cli *CLI) Run() {
    ...
    if createWalletCmd.Parsed() {
        //创建钱包
        cli.createWallet(nodeID)
    }
    ...
} 
```

step2：修改`CLI_createwallet.go`文件中，修改`createWallet()`方法声明，添加`nodeID`，代码如下：

```go
package BLC

func (cli *CLI) createWallet(nodeID string){
    wallets:= NewWallets(nodeID)
    wallets.CreateNewWallet(nodeID)
} 
```

step3：修改`CLI_createwallet.go`文件中，

先修改`walletFile`命名：

```go
const walletFile = "Wallets_%s.dat" 
```

然后修改`createWallet()`方法声明，添加`nodeID`，代码如下：

```go
func NewWallets(nodeID string) *Wallets {
    //wallets := &WalletsMap{}
    //wallets.WalletsMap = make(map[string]*Wallet)
    //return wallets

    walletFile := fmt.Sprintf(walletFile,nodeID)

    //1.判断钱包文件是否存在
    ...
} 
```

接下来，修改`CreateNewWallet()`方法声明，添加`nodeID`，代码如下:

```go
//3.创建一个新钱包
func (ws *Wallets) CreateNewWallet(nodeID string) {
    wallet := NewWallet()
    fmt.Printf("创建钱包地址：%s\n", wallet.GetAddress())
    ws.WalletsMap[string(wallet.GetAddress())] = wallet

    //将钱包保存
    ws.SaveWallets(nodeID)
} 
```

再然后，修改`SaveWallets()`方法声明，添加`nodeID`，代码如下:

```go
func (ws *Wallets) SaveWallets(nodeID string) {

    walletFile := fmt.Sprintf(walletFile,nodeID)

    var content bytes.Buffer
    //注册的目的，为了可以序列化任何类型，wallet 结构体中有接口类型。将接口进行注册
    ...
} 
```

**6\. 修改打印钱包地址功能：**

step1：修改`CLI.go`中的`Run()`方法，修改`cli.addressLists(nodeID)`，添加`nodeID`，代码如下：

```go
func (cli *CLI) Run() {
    ...
    //获取所有的钱包地址
    if addressListsCmd.Parsed(){
        cli.addressLists(nodeID)
    }
    ...
} 
```

step2：修改`CLI_addresslists.go`文件中，修改`addressLists()`方法声明，添加`nodeID`，代码如下：

```go
 package BLC

import "fmt"

func (cli *CLI)addressLists(nodeID string){
    fmt.Println("打印所有的钱包地址。。")
    //获取
    Wallets:=NewWallets(nodeID)
    for address,_ := range Wallets.WalletsMap{
        fmt.Println("address:",address)
    }
} 
```

**7\. 修改转账交易功能：**

step1：修改`CLI.go`中的`Run()`方法，修改`cli.send(from, to, amount,nodeID)`，添加`nodeID`，代码如下：

```go
func (cli *CLI) Run() {
    ...
    if sendBlockCmd.Parsed() {
        ...
        for i := 0; i < len(from); i++ {
            if !IsValidForAddress([]byte(from[i])) || !IsValidForAddress([]byte(to[i])) {
                fmt.Println("钱包地址无效")
                printUsage()
                os.Exit(1)
            }
        }

        cli.send(from, to, amount,nodeID)
    }
    ...
} 
```

step2：修改`CLI_send.go`文件中，修改`send()`方法声明，添加`nodeID`，代码如下：

```go
package BLC

//转账
func (cli *CLI) send(from, to, amount [] string,nodeID string) {
    //if !dbExists() {
    //    fmt.Println("数据库不存在。。。")
    //    os.Exit(1)
    //}
    blockchain := GetBlockchainObject(nodeID)

    blockchain.MineNewBlock(from, to, amount,nodeID)
    defer blockchain.DB.Close()

    utxoSet:=&UTXOSet{blockchain}
    //转账成功以后，需要更新
    //utxoSet.ResetUTXOSet()
    utxoSet.Update()
} 
```

step3：在`BlockChain.go`文件中，修改`MineNewBlock()`方法声明，添加`nodeID`，代码如下：

```go
 //挖掘新的区块
func (bc *BlockChain) MineNewBlock(from, to, amount []string,nodeID string) {

    var txs []*Transaction
...

    for i := 0; i < len(from); i++ {

        amountInt, _ := strconv.ParseInt(amount[i], 10, 64)
        tx := NewSimpleTransaction(from[i], to[i], amountInt, utxoSet, txs,nodeID)

        txs = append(txs, tx)
    }

...

} 
```

step4：在`Transaction.go`文件中，`NewSimpleTransaction()`方法，添加`nodeID`，代码如下:

```go
 func NewSimpleTransaction(from, to string, amount int64, utxoSet *UTXOSet, txs []*Transaction,nodeID string) *Transaction {
...
    //获取钱包
    wallets := NewWallets(nodeID)
    wallet := wallets.WalletsMap[from]

    ...
} 
```

最后，我们进行代码测试，看一下是否可以不同的`NODE_ID`，可以创建出不同的数据库和钱包文件，打开一个终端并输入以下命令：

```go
hanru:mypublicchain ruby$ cd day08_10_Net/
hanru:day08_10_Net ruby$ go build -o bc main.go
hanru:day08_10_Net ruby$ export NODE_ID=3000
hanru:day08_10_Net ruby$ ./bc haha
hanru:day08_10_Net ruby$ ./bc createwallet 
```

运行结果如下：

![`img.kongyixueyuan.com/1104_%E8%BF%90%E8%A1%8C%E7%BB%93%E6%9E%9C.png`](img/198679dcb9861e3369758afccb214800.jpg)

接下来我们继续输入以下命令：

```go
hanru:day08_10_Net ruby$ ./bc createwallet
hanru:day08_10_Net ruby$ ./bc addresslists
hanru:day08_10_Net ruby$ ./bc createblockchain -address '16s8TuPn9PeoSGLMB395nsZ7taEAFjuu3L'
hanru:day08_10_Net ruby$ ./bc getbalance -address '16s8TuPn9PeoSGLMB395nsZ7taEAFjuu3L' 
```

运行效果如下：

![`img.kongyixueyuan.com/1105_%E8%BF%90%E8%A1%8C%E7%BB%93%E6%9E%9C2.png`](img/3d1136827be4b571a38972a6df81ed72.jpg)

接下来我们实现转账以及查询余额，输入终端命令如下:

```go
hanru:day08_10_Net ruby$ ./bc send -from '["16s8TuPn9PeoSGLMB395nsZ7taEAFjuu3L"]' -to '["1KcTkHkW2UhtCLjQCzaYYz6XhDc4LKcACx"]' -amount '["4"]'
hanru:day08_10_Net ruby$ ./bc getbalance -address '16s8TuPn9PeoSGLMB395nsZ7taEAFjuu3L'
hanru:day08_10_Net ruby$ ./bc getbalance -address '1KcTkHkW2UhtCLjQCzaYYz6XhDc4LKcACx' 
```

运行效果如下：

![`img.kongyixueyuan.com/1106_%E8%BD%AC%E8%B4%A6.png`](img/9a33c7bb9598104f9bfb23fb6fcfdae8.jpg)

最后，我们再尝试以下其他的`NODE_ID`，再打开另一个终端，输入以下命令：

```go
hanru:mypublicchain ruby$ cd day08_10_Net/
hanru:day08_10_Net ruby$ export NODE_ID=3001
hanru:day08_10_Net ruby$ ./bc createwallet
hanru:day08_10_Net ruby$ ./bc createblockchain -address '13RHpxgnDXXtyPR67LdWUTGuQj4F6wgr8Y'
hanru:day08_10_Net ruby$ ./bc getbalance -address '13RHpxgnDXXtyPR67LdWUTGuQj4F6wgr8Y' 
```

运行结果：

![`img.kongyixueyuan.com/1107_%E5%8F%A6%E4%B8%80%E4%B8%AA%E8%8A%82%E7%82%B9.png`](img/e29bf79590606fed506f33a5136d0c0a.jpg)

至此，我们已经将项目中结合了`NODE_ID`，可以模拟不同的节点工作了。

#### 4.5.3 版本`Version`

节点通过消息（message）进行交流。当一个新的节点开始运行时，它会从一个 DNS 种子获取几个节点，给它们发送 `version` 消息，在我们的实现看起来就像是这样：

```go
package BLC

type Version struct {
    Version    int64 // 版本
    BestHeight int64 // 当前节点区块的高度
    AddrFrom   string //当前节点的地址
} 
```

由于我们仅有一个区块链版本，所以 `Version` 字段实际并不会存储什么重要信息。`BestHeight` 存储区块链中节点的高度。`AddFrom` 存储发送者的地址。

接收到 `version` 消息的节点应该做什么呢？它会响应自己的 `version` 消息。这是一种握手：如果没有事先互相问候，就不可能有其他交流。不过，这并不是出于礼貌：`version` 用于找到一个更长的区块链。当一个节点接收到 `version` 消息，它会检查本节点的区块链是否比 `BestHeight` 的值更大。如果不是，节点就会请求并下载缺失的块。

为了接收消息，我们需要一个服务器：

首先修改`CLI.go`文件中的`Run()`方法，添加启动节点命令，修改后代码如下:

```go
 //step2：添加 Run 方法
func (cli *CLI) Run() {
    //判断命令行参数的长度
    isValidArgs()

    /*
    获取节点 ID
    解释：返回当前进程的环境变量 varname 的值,若变量没有定义时返回 nil
    export NODE_ID=8888

    每次打开一个终端，都需要设置 NODE_ID 的值。
    变量名 NODE_ID，可以更改别的。
     */

    nodeID :=os.Getenv("NODE_ID")
    if nodeID == ""{
        fmt.Printf("NODE_ID 环境变量没有设置。。\n")
        os.Exit(1)
    }
    fmt.Printf("NODE_ID:%s\n",nodeID)

    //1.创建 flagset 标签对象
    ...

    startNodeCmd := flag.NewFlagSet("startnode",flag.ExitOnError)

    //2.设置标签后的参数
    ...
    flagMiner := startNodeCmd.String("miner","","定义挖矿奖励的地址......")

    //3.解析
    switch os.Args[1] {
    ...

    case "startnode":
        err := startNodeCmd.Parse(os.Args[2:])
        if err != nil {
            log.Panic(err)
        }

    default:
        printUsage()
        os.Exit(1) //退出
    }

    ...

    if startNodeCmd.Parsed() {
        cli.startNode(nodeID,*flagMiner)
    }

} 
```

然后在 BLC 包下，新建一个`CLI_startnode.go`文件，添加启动节点功能方法，代码如下:

```go
package BLC

import (
    "fmt"
    "os"
)

func (cli *CLI) startNode(nodeID string,minerAdd string)  {

    // 启动服务器

    if minerAdd == "" || IsValidForAddress([]byte(minerAdd))  {
        //  启动服务器
        fmt.Printf("启动服务器:localhost:%s\n",nodeID)
        startServer(nodeID,minerAdd)

    } else {

        fmt.Println("指定的地址无效....")
        os.Exit(0)
    }

} 
```

先新建`Server_var.go`文件，存储节点全局变量，代码如下：

```go
package BLC

//存储节点全局变量

//localhost:3000 主节点的地址
var knowNodes = []string{"localhost:3000"}
var nodeAddress string //全局变量，节点地址
// 存储 hash 值
var transactionArray [][]byte
var minerAddress string
var memoryTxPool = make(map[string]*Transaction) 
```

修改`Constant.go`文件，添加几个常量，代码如下：

```go
package BLC

const DBNAME = "blockchain_%s.db"  //数据库名
const BLOCKTABLENAME = "blocks" //表名

const PROTOCOL  = "tcp"
const COMMANDLENGTH  = 12
const NODE_VERSION  = 1

//12 个字节 + 结构体序列化的字节数组

// 命令
const COMMAND_VERSION  = "version"
const COMMAND_ADDR  = "addr"
const COMMAND_BLOCK  = "block"
const COMMAND_INV  = "inv"
const COMMAND_GETBLOCKS  = "getblocks"
const COMMAND_GETDATA  = "getdata"
const COMMAND_TX  = "tx"

// 类型
const BLOCK_TYPE  = "block"
const TX_TYPE  = "tx" 
```

再新建`Server.go`文件，用于表示节点的服务端，添加`startServer()`方法，代码如下：

```go
func startServer(nodeID string,minerAdd string)  {
    //""
    // 当前节点的 IP 地址
    nodeAddress = fmt.Sprintf("localhost:%s",nodeID)

    minerAddress = minerAdd

    ln,err := net.Listen(PROTOCOL,nodeAddress)

    if err != nil {
        log.Panic(err)
    }

    defer ln.Close()

    bc := GetBlockchainObject(nodeID)

    //defer bc.DB.Close()

    // 第一个终端：端口为 3000,启动的就是主节点
    // 第二个终端：端口为 3001，钱包节点
    // 第三个终端：端口号为 3002，矿工节点
    if nodeAddress != knowNodes[0]{
         // 此节点是钱包节点或者矿工节点，需要向主节点发送请求同步数据

         sendVersion(knowNodes[0],bc)
    }

    for {
        // 收到的数据的格式是固定的，12 字节+结构体字节数组

        // 接收客户端发送过来的数据
        conn, err := ln.Accept()
        if err != nil {
            log.Panic(err)
        }

        go handleConnection(conn,bc)

    }

} 
```

首先，我们对中心节点的地址进行硬编码：因为每个节点必须知道从何处开始初始化。`minerAddress` 参数指定了接收挖矿奖励的地址。代码片段：

```go
if nodeAddress != knowNodes[0]{
         // 此节点是钱包节点或者矿工节点，需要向主节点发送请求同步数据

         sendVersion(knowNodes[0],bc)
} 
```

这意味着如果当前节点不是中心节点，它必须向中心节点发送 `version` 消息来查询是否自己的区块链已过时。所以新建`Server_send.go`文件，用于表示发送消息，并添加方法：

```go
//COMMAND_VERSION
func sendVersion(toAddress string,bc *BlockChain)  {

    bestHeight := bc.GetBestHeight()

    payload := gobEncode(Version{NODE_VERSION, bestHeight, nodeAddress})

    //version
    request := append(commandToBytes(COMMAND_VERSION), payload...)

    sendData(toAddress,request)

} 
```

修改`BlockChain.go`文件，添加方法，用于获取最新区块的高度，代码如下：

```go
//----------
//获取最新区块的高度
func (bc *BlockChain) GetBestHeight() int64 {

    block := bc.Iterator().Next()

    return block.Height
} 
```

我们的消息，在底层就是字节序列。前 12 个字节指定了命令名（比如这里的 `version`），后面的字节会包含 **gob** 编码的消息结构。所以我们可以添加一个工具方法。

修改 utils.go 文件，添加以下工具方法：

```go
//version 转字节数组
func commandToBytes(command string) []byte {
    var bytes [COMMANDLENGTH]byte

    for i, c := range command {
        bytes[i] = byte(c)
    }

    return bytes[:]
} 
```

它创建一个 12 字节的缓冲区，并用命令名进行填充，将剩下的字节置为空。

接下来再添加一个方法，用于将字节数据转为命令：

```go
//字节数组转 command
func bytesToCommand(bytes []byte) string {
    var command []byte

    for _, b := range bytes {
        if b != 0x0 {
            command = append(command, b)
        }
    }

    return fmt.Sprintf("%s", command)
} 
```

再添加一个序列化的工具方法，代码如下：

```go
 // 将结构体序列化成字节数组
func gobEncode(data interface{}) []byte {
    var buff bytes.Buffer

    enc := gob.NewEncoder(&buff)
    err := enc.Encode(data)
    if err != nil {
        log.Panic(err)
    }

    return buff.Bytes()
} 
```

当一个节点接收到一个命令，它会运行 `bytesToCommand` 来提取命令名，并选择正确的处理器处理命令主体。

所以接下来在`Server.go`文件中，添加`handleConnection()`方法，代码如下：

```go
 func handleConnection(conn net.Conn,bc *BlockChain) {

    // 读取客户端发送过来的所有的数据
    request, err := ioutil.ReadAll(conn)
    if err != nil {
        log.Panic(err)
    }

    fmt.Printf("Receive a Message:%s\n",request[:COMMANDLENGTH])

    //version
    command := bytesToCommand(request[:COMMANDLENGTH])

    // 12 字节 + 某个结构体序列化以后的字节数组

    switch command {
        case COMMAND_VERSION:
            handleVersion(request, bc)

            ...

        default:
            fmt.Println("Unknown command!")
    }

    conn.Close()
} 
```

接下来，新建一个`Server_handle.go`文件，用于处理接收到的命令。

添加 `version` 命令处理方法，代码如下：

```go
 func handleVersion(request []byte,bc *BlockChain)  {

    var buff bytes.Buffer
    var payload Version

    dataBytes := request[COMMANDLENGTH:]

    // 反序列化
    buff.Write(dataBytes)
    dec := gob.NewDecoder(&buff)
    err := dec.Decode(&payload)
    if err != nil {
        log.Panic(err)
    }

    //Version
    //1\. Version
    //2\. BestHeight
    //3\. 节点地址

    bestHeight := bc.GetBestHeight() //3 1
    foreignerBestHeight := payload.BestHeight // 1 3

    if bestHeight > foreignerBestHeight {
        sendVersion(payload.AddrFrom,bc)
    } else if bestHeight < foreignerBestHeight {
        // 去向主节点要信息
        sendGetBlocks(payload.AddrFrom)
    }

    if !nodeIsKnown(payload.AddrFrom) {
        knowNodes = append(knowNodes, payload.AddrFrom)
    }

} 
```

首先，我们需要对请求进行解码，提取有效信息。所有的处理器在这部分都类似，所以我们会下面的代码片段中略去这部分。

然后节点将从消息中提取的 `BestHeight` 与自身进行比较。如果自身节点的区块链更长，它会回复 `version` 消息；否则，它会发送 `getblocks` 消息。

#### 4.5.4 getblocks

`getblocks` 意为 “给我看一下你有什么区块”（在比特币中，这会更加复杂）。注意，它并没有说“把你全部的区块给我”，而是请求了一个块哈希的列表。这是为了减轻网络负载，因为区块可以从不同的节点下载，并且我们不想从一个单一节点下载数十 GB 的数据。

新建`Server_getblocks.go`文件，添加结构体代码如下：

```go
package BLC

//getblocks 意为 “给我看一下你有什么区块”
type GetBlocks struct {
    AddrFrom string
} 
```

处理命令十分简单，在`Server_handle.go`文件中，添加`handleGetblocks()`方法，代码如下：

```go
func handleGetblocks(request []byte,bc *BlockChain)  {

    var buff bytes.Buffer
    var payload GetBlocks

    dataBytes := request[COMMANDLENGTH:]

    // 反序列化
    buff.Write(dataBytes)
    dec := gob.NewDecoder(&buff)
    err := dec.Decode(&payload)
    if err != nil {
        log.Panic(err)
    }

    blocks := bc.GetBlockHashes()

    //txHash blockHash
    sendInv(payload.AddrFrom, BLOCK_TYPE, blocks)

} 
```

在我们简化版的实现中，它会返回 **所有块哈希**。

修改`BlockChain.go`文件，添加`GetBlockHashes()`方法，用于获取所有区块的`hash`。代码如下：

```go
 //获取所有区块的 hash
func (bc *BlockChain) GetBlockHashes() [][]byte {

    blockIterator := bc.Iterator()

    var blockHashs [][]byte

    for {
        block := blockIterator.Next()

        blockHashs = append(blockHashs,block.Hash)

        var hashInt big.Int
        hashInt.SetBytes(block.PrevBlockHash)

        if hashInt.Cmp(big.NewInt(0)) == 0 {
            break;
        }
    }

    return blockHashs
} 
```

#### 4.5.5 `inv`

比特币使用 `inv` 来向其他节点展示当前节点有什么块和交易。再次提醒，它没有包含完整的区块链和交易，仅仅是哈希而已。`Type` 字段表明了这是块还是交易。

新建`Server_Inv.go`文件，添加结构体，代码如下：

```go
package BLC

type Inv struct {
    AddrFrom string  //自己的地址
    Type     string  //类型 block tx
    Items    [][]byte //hash 二维数组
} 
```

处理 `inv` 稍显复杂，在`Server_handle.go`文件中，添加`handleInv()`方法，代码如下：

```go
 func handleInv(request []byte,bc *BlockChain)  {

    var buff bytes.Buffer
    var payload Inv

    dataBytes := request[COMMANDLENGTH:]

    // 反序列化
    buff.Write(dataBytes)
    dec := gob.NewDecoder(&buff)
    err := dec.Decode(&payload)
    if err != nil {
        log.Panic(err)
    }

    // Ivn 3000 block hashes [][]

    if payload.Type == BLOCK_TYPE {

        //tansactionArray = payload.Items

        //payload.Items

        blockHash := payload.Items[0]
        sendGetData(payload.AddrFrom, BLOCK_TYPE , blockHash)

        if len(payload.Items) >= 1 {
            transactionArray = payload.Items[1:]
        }
    }

    if payload.Type == TX_TYPE {

        txHash := payload.Items[0]
        if memoryTxPool[hex.EncodeToString(txHash)] == nil  {
            sendGetData(payload.AddrFrom, TX_TYPE , txHash)
        }

    }

} 
```

如果收到块哈希，我们想要将它们保存在 `blocksInTransit` 变量来跟踪已下载的块。这能够让我们从不同的节点下载块。在将块置于传送状态时，我们给 `inv` 消息的发送者发送 `getdata` 命令并更新 `blocksInTransit`。在一个真实的 P2P 网络中，我们会想要从不同节点来传送块。

在我们的实现中，我们永远也不会发送有多重哈希的 `inv`。这就是为什么当 `payload.Type == "tx"` 时，只会拿到第一个哈希。然后我们检查是否在内存池中已经有了这个哈希，如果没有，发送 `getdata` 消息。

#### 4.5.6 getdata

`getdata` 用于某个块或交易的请求，它可以仅包含一个块或交易的 ID。

新建`Server_GetData.go`文件，添加结构体：

```go
package BLC
//用于某个块或交易的请求，它可以仅包含一个块或交易的 ID。
type GetData struct {
    AddrFrom string
    Type     string
    Hash       []byte
} 
```

在`Server_handle.go`文件中，添加`handleGetData()`方法，代码如下：

```go
 func handleGetData(request []byte,bc *BlockChain)  {

    var buff bytes.Buffer
    var payload GetData

    dataBytes := request[COMMANDLENGTH:]

    // 反序列化
    buff.Write(dataBytes)
    dec := gob.NewDecoder(&buff)
    err := dec.Decode(&payload)
    if err != nil {
        log.Panic(err)
    }

    if payload.Type == BLOCK_TYPE {

        block, err := bc.GetBlock([]byte(payload.Hash))
        if err != nil {
            return
        }

        sendBlock(payload.AddrFrom, block)
    }

    if payload.Type == TX_TYPE {

        tx := memoryTxPool[hex.EncodeToString(payload.Hash)]

        sendTx(payload.AddrFrom,tx)

    }
} 
```

在`BlockChain.go`文件中，添加`GetBlock()`方法，用于根据指定的`hash`获取对应的`block`数据，代码如下：

```go
 //根据 hash 获取区块
func (bc *BlockChain) GetBlock(blockHash []byte) ([]byte ,error) {

    var blockBytes []byte

    err := bc.DB.View(func(tx *bolt.Tx) error {

        b := tx.Bucket([]byte(BLOCKTABLENAME))

        if b != nil {

            blockBytes = b.Get(blockHash)

        }

        return nil
    })

    return blockBytes,err
} 
```

这个处理也比较地直观：如果它们请求一个块，则返回块；如果它们请求一笔交易，则返回交易。注意，我们并不检查实际上是否已经有了这个块或交易。(这是一个缺陷)。

#### 4.5.7 `block` 和 `tx`

实际完成数据转移的正是这些消息：区块和交易。

新建`Server_block.go`文件，添加结构体：

```go
package BLC

type BlockData struct {
    AddrFrom string
    Block []byte
} 
```

再新建`Server_Tx.go`文件，添加结构体：

```go
package BLC

type Tx struct {
    AddrFrom string
    Tx *Transaction
} 
```

处理 `block` 消息十分简单，在`Server_handle.go`文件中，添加`handleBlock()`方法，代码如下：

```go
 func handleBlock(request []byte,bc *BlockChain)  {
    var buff bytes.Buffer
    var payload BlockData

    dataBytes := request[COMMANDLENGTH:]

    // 反序列化
    buff.Write(dataBytes)
    dec := gob.NewDecoder(&buff)
    err := dec.Decode(&payload)
    if err != nil {
        log.Panic(err)
    }

    blockBytes := payload.Block

    block := DeserializeBlock(blockBytes)

    fmt.Println("Recevied a new block!")
    bc.AddBlock(block)
    UTXOSet := &UTXOSet{bc}
    UTXOSet.Update()

    fmt.Printf("Added block %x\n", block.Hash)

    if len(transactionArray) > 0 {
        blockHash := transactionArray[0]
        sendGetData(payload.AddrFrom, "block", blockHash)

        transactionArray = transactionArray[1:]
    } else {

        //fmt.Println("数据库重置......")
        //UTXOSet := &UTXOSet{bc}
        //UTXOSet.ResetUTXOSet()

    }

} 
```

在`BlockChain.go`文件中，添加`AddBlock()`方法，将获取到的区块添加到数据库中。代码如下：

```go
 //添加区块到数据库
func (bc *BlockChain) AddBlock(block *Block)  {

    err := bc.DB.Update(func(tx *bolt.Tx) error {

        b := tx.Bucket([]byte(BLOCKTABLENAME))

        if b != nil {

            blockExist := b.Get(block.Hash)

            if blockExist != nil {
                // 如果存在，不需要做任何过多的处理
                return nil
            }

            err := b.Put(block.Hash,block.Serilalize())

            if err != nil {
                log.Panic(err)
            }

            // 最新的区块链的 Hash
            blockHash := b.Get([]byte("l"))

            blockBytes := b.Get(blockHash)

            blockInDB := DeserializeBlock(blockBytes)

            if blockInDB.Height < block.Height {

                b.Put([]byte("l"),block.Hash)
                bc.Tip = block.Hash
            }
        }

        return nil
    })

    if err != nil {
        log.Panic(err)
    }
} 
```

当接收到一个新块时，我们把它放到区块链里面。如果还有更多的区块需要下载，我们继续从上一个下载的块的那个节点继续请求。当最后把所有块都下载完后，对 UTXO 集进行重新索引。

处理 `tx` 消息是最困难的部分，我们一步一步来实现：

首先在`CLI.go 中`修改转账命令：

```go
//fmt.Println("\tsend -from FROM -to TO -amount AMOUNT -mine -- 交易明细.")

flagMine := sendBlockCmd.Bool("mine",false,"是否在当前节点中立即验证....") 
```

修改`CLI_send.go`文件，修改转账方法:

```go
package BLC

import (
    "strconv"
    "fmt"
)

// 转账
func (cli *CLI) send(from []string, to []string, amount []string, nodeID string, mineNow bool) {

    blockchain := GetBlockchainObject(nodeID)
    utxoSet := &UTXOSet{blockchain}
    defer blockchain.DB.Close()

    if mineNow {
        blockchain.MineNewBlock(from, to, amount, nodeID)

        //转账成功以后，需要更新一下
        utxoSet.Update()
    } else {
        // 把交易发送到矿工节点去进行验证
        fmt.Println("由矿工节点处理......")
        value, _ := strconv.Atoi(amount[0])
        tx := NewSimpleTransaction(from[0], to[0], int64(value), utxoSet, []*Transaction{}, nodeID)

        sendTx(knowNodes[0], tx)
    }

} 
```

如果转账时没有直接挖矿创建区块，可以转交给矿工节点进行挖矿，那么就需要将转账交易发送给矿工节点，接下来我们实现转账进入交易消息的处理部分，在`Server_handle.go`文件中，添加`handleTx()`方法，代码如下：

```go
 func handleTx(request []byte, bc *BlockChain) {

    var buff bytes.Buffer
    var payload Tx

    dataBytes := request[COMMANDLENGTH:]

    // 反序列化
    buff.Write(dataBytes)
    dec := gob.NewDecoder(&buff)
    err := dec.Decode(&payload)
    if err != nil {
        log.Panic(err)
    }

    //-----

    tx := payload.Tx
    memoryTxPool[hex.EncodeToString(tx.TxID)] = tx

    // 说明主节点自己
    if nodeAddress == knowNodes[0] {
        // 给矿工节点发送交易 hash
        for _, nodeAddr := range knowNodes {

            if nodeAddr != nodeAddress && nodeAddr != payload.AddrFrom {
                sendInv(nodeAddr, TX_TYPE, [][]byte{tx.TxID})
            }

        }
    }

    // 矿工进行挖矿验证
    // "" | 1DVFvyCK8qTQkLBTZ5fkh5eDSbcZVoHAsj
    if len(memoryTxPool) >= 1 && len(minerAddress) > 0 {

    MineTransactions:

        utxoSet := &UTXOSet{bc}

        txs := []*Transaction{tx}

        //奖励
        coinbaseTx := NewCoinBaseTransaction(minerAddress)
        txs = append(txs, coinbaseTx)

        _txs := []*Transaction{}

        //fmt.Println("开始进行数字签名验证.....")

        for _, tx := range txs {

            //fmt.Printf("开始第%d 次验证...\n",index)

            // 数字签名失败
            if bc.VerifyTransaction(tx, _txs) != true {
                log.Panic("ERROR: Invalid transaction")
            }

            //fmt.Printf("第%d 次验证成功\n",index)
            _txs = append(_txs, tx)
        }

        //fmt.Println("数字签名验证成功.....")

        //1\. 通过相关算法建立 Transaction 数组
        var block *Block

        bc.DB.View(func(tx *bolt.Tx) error {

            b := tx.Bucket([]byte(BLOCKTABLENAME))
            if b != nil {

                hash := b.Get([]byte("l"))

                blockBytes := b.Get(hash)

                block = DeserializeBlock(blockBytes)

            }

            return nil
        })

        //2\. 建立新的区块
        block = NewBlock(txs, block.Hash, block.Height+1)

        //将新区块存储到数据库
        bc.DB.Update(func(tx *bolt.Tx) error {
            b := tx.Bucket([]byte(BLOCKTABLENAME))
            if b != nil {

                b.Put(block.Hash, block.Serilalize())

                b.Put([]byte("l"), block.Hash)

                bc.Tip = block.Hash

            }
            return nil
        })
        utxoSet.Update()
        sendBlock(knowNodes[0], block.Serilalize())

        for _, tx := range txs {
            txID := hex.EncodeToString(tx.TxID)
            delete(memoryTxPool, txID)
        }
        for _, node := range knowNodes {
            if node != nodeAddress {
                sendInv(node, "block", [][]byte{block.Hash})
            }
        }

        if len(memoryTxPool) > 0 {
            goto MineTransactions
        }
    }
} 
```

首先要做的事情是将新交易放到内存池中（再次提醒，在将交易放到内存池之前，必要对其进行验证）。

```go
tx := payload.Tx
memoryTxPool[hex.EncodeToString(tx.TxID)] = tx 
```

下个片段，检查当前节点是否是中心节点。在我们的实现中，中心节点并不会挖矿。它只会将新的交易推送给网络中的其他节点。

```go
 // 说明主节点自己
    if nodeAddress == knowNodes[0] {
        // 给矿工节点发送交易 hash
        for _,nodeAddr := range knowNodes {

            if nodeAddr != nodeAddress && nodeAddr != payload.AddrFrom {
                sendInv(nodeAddr,TX_TYPE,[][]byte{tx.TxID})
            }

        }
    } 
```

下一个很大的代码片段是矿工节点“专属”。让我们对它进行一下分解:

`miningAddress` 只会在矿工节点上设置。如果当前节点（矿工）的内存池中有 1 笔或更多的交易，开始挖矿：

```go
for _,tx := range txs  {

            //fmt.Printf("开始第%d 次验证...\n",index)

            // 数字签名失败
            if bc.VerifyTransaction(tx,_txs) != true {
                log.Panic("ERROR: Invalid transaction")
            }

            //fmt.Printf("第%d 次验证成功\n",index)
            _txs = append(_txs,tx)
        } 
```

首先，内存池中所有交易都是通过验证的。无效的交易会被忽略，如果没有有效交易，则挖矿中断。

```go
//奖励
coinbaseTx := NewCoinBaseTransaction(minerAddress)
txs = append(txs,coinbaseTx)

_txs := []*Transaction{}
...
var block *Block

bc.DB.View(func(tx *bolt.Tx) error {

    b := tx.Bucket([]byte(BLOCKTABLENAME))
        if b != nil {

            hash := b.Get([]byte("l"))
            blockBytes := b.Get(hash)
            block = DeserializeBlock(blockBytes)

        }
        return nil
})

//2\. 建立新的区块
block = NewBlock(txs, block.Hash,block.Height+1 )

//将新区块存储到数据库
bc.DB.Update(func(tx *bolt.Tx) error {
    b := tx.Bucket([]byte(BLOCKTABLENAME))
    if b != nil {

        b.Put(block.Hash, block.Serilalize())

        b.Put([]byte("l"), block.Hash)

        bc.Tip = block.Hash

    }
    return nil
})
utxoSet.Update()
sendBlock(knowNodes[0],block.Serilalize()) 
```

验证后的交易被放到一个块里，同时还有附带奖励的 `coinbase` 交易。当块被挖出来以后，UTXO 集会被更新。

#### 4.5.8 结果

让我们来回顾一下上面定义的场景。

首先，在第一个终端窗口中将 `NODE_ID` 设置为 3000（`export NODE_ID=3000`）。为了让你知道什么节点执行什么操作，我会使用像 **NODE 3000** 或 **NODE 3001** 进行标识。

**NODE 3000**

首先我们打开一个终端，模拟主节点，输入以下命令，创建一个钱包地址：

```go
hanru:mypublicchain ruby$ cd day08_10_Net/
hanru:day08_10_Net ruby$ ls
hanru:day08_10_Net ruby$ go build -o bc main.go
hanru:day08_10_Net ruby$ ls
hanru:day08_10_Net ruby$ export NODE_ID=3000
hanru:day08_10_Net ruby$ ./bc
hanru:day08_10_Net ruby$ ./bc createwallet 
```

运行结果如下：

![`img.kongyixueyuan.com/1109_%E4%B8%BB%E8%8A%82%E7%82%B9.png`](img/065098d74a41bffd0016a644b8c42368.jpg)

继续输入以下命令，一个新的区块链：

```go
 hanru:day08_10_Net ruby$ ./bc createblockchain -address '1XiTWdg7EFaP9pns6JrGTaoiZp8J8eQXy'
hanru:day08_10_Net ruby$ ./bc printchain
hanru:day08_10_Net ruby$ cp blockchain_3000.db blockchain_genesis.db 
```

运行结果如下：

![`img.kongyixueyuan.com/1110_%E4%B8%BB%E8%8A%82%E7%82%B92.png`](img/37c2088a5f1eddb8430c703c7958f0a7.jpg)

然后，会生成一个仅包含创世块的区块链。我们需要保存块，并在其他节点使用。创世块承担了一条链标识符的角色（在 Bitcoin Core 中，创世块是硬编码的），所以我们备份了创世区块。

**NODE 3001**

接下来，打开一个新的终端窗口，将 `node ID` 设置为 3001。这会作为一个钱包节点。

输入以下命令：

```go
hanru:mypublicchain ruby$ cd day08_10_Net/
hanru:day08_10_Net ruby$ ./bc
hanru:day08_10_Net ruby$ export NODE_ID=3001
hanru:day08_10_Net ruby$ ./bc createwallet
hanru:day08_10_Net ruby$ ./bc createwallet
hanru:day08_10_Net ruby$ cp blockchain_genesis.db blockchain_3001.db 
```

运行结果如下 ：

![`img.kongyixueyuan.com/1111_%E9%92%B1%E5%8C%85%E8%8A%82%E7%82%B9.png`](img/3f36e8e6950e814831b2caef3c328a5f.jpg)

**NODE 3000**

向钱包地址发送一些币，在主节点终端输入以下命令：

```go
hanru:day08_10_Net ruby$ ./bc
hanru:day08_10_Net ruby$ ./bc send -from '["1XiTWdg7EFaP9pns6JrGTaoiZp8J8eQXy"]' -to '["15bEARdUzej9QWNc2th55LWhjq2vhVbtok"]' -amount '["8"]' -mine
hanru:day08_10_Net ruby$ ./bc printchain 
```

`-mine` 标志指的是块会立刻被同一节点挖出来。我们必须要有这个标志，因为初始状态时，网络中没有矿工节点。

运行结果如下：

![`img.kongyixueyuan.com/1112_%E8%BD%AC%E8%B4%A6.png`](img/ac8eee451f13fa0d10acd085699463e5.jpg)

继续输入命令，查询余额：

```go
hanru:day08_10_Net ruby$ ./bc getbalance -address '1XiTWdg7EFaP9pns6JrGTaoiZp8J8eQXy'
hanru:day08_10_Net ruby$ ./bc getbalance -address '15bEARdUzej9QWNc2th55LWhjq2vhVbtok' 
```

运行效果如下：

![`img.kongyixueyuan.com/1113_%E6%9F%A5%E8%AF%A2%E4%BD%99%E9%A2%9D.png`](img/251e006952001e9705b00297dfaffbca.jpg)

**NODE 3001**

切换到钱包节点，输入命令查询余额：

```go
hanru:day08_10_Net ruby$ ./bc getbalance -address '1XiTWdg7EFaP9pns6JrGTaoiZp8J8eQXy'
hanru:day08_10_Net ruby$ ./bc getbalance -address '15bEARdUzej9QWNc2th55LWhjq2vhVbtok' 
```

运行结果如下：

![`img.kongyixueyuan.com/1114_%E6%9F%A5%E8%AF%A2%E4%BD%99%E9%A2%9D.png`](img/c361dda4254f2d3654a18ca1c28a5985.jpg)

我们发现钱包节点的查询到的余额和主节点中的数据不一致，接下来我们启动节点进行数据同步

**NODE 3000**

切换到主节点的终端，输入以下命令，启动主节点：

```go
hanru:day08_10_Net ruby$ ./bc startnode 
```

启动主节点后等待钱包节点链接，如果 有钱包节点链接，那么就会有消息传递，进行数据同步，运行效果如下：

![`img.kongyixueyuan.com/1115_%E5%90%AF%E5%8A%A8%E4%B8%BB%E8%8A%82%E7%82%B9.png`](img/fa7fb0865d549c875aacf33f39a5a80a.jpg)

这个节点会持续运行，直到本文定义的场景结束。

**NODE 3001**

接下来切换到钱包节点终端，输入以下命令，启动钱包节点，进行数据同步：

```go
hanru:day08_10_Net ruby$ ./bc startnode 
```

运行效果如下：

![`img.kongyixueyuan.com/1116_%E5%90%AF%E5%8A%A8%E9%92%B1%E5%8C%85%E8%8A%82%E7%82%B9.png`](img/a1fd96a0092699638404e35948db7113.jpg)

同步数据后，输入 ctrl+c 键，强制结束。

然后重新输入以下命令进行查看数据是否同步：

```go
hanru:day08_10_Net ruby$ ./bc printchain
hanru:day08_10_Net ruby$ ./bc getbalance -address '1XiTWdg7EFaP9pns6JrGTaoiZp8J8eQXy'
hanru:day08_10_Net ruby$ ./bc getbalance -address '15bEARdUzej9QWNc2th55LWhjq2vhVbtok' 
```

运行结果如下，打印区块，我们发现 blockchain_3001.db 中已经有了 2 个 block：

![`img.kongyixueyuan.com/1117_%E6%9F%A5%E8%AF%A2.png`](img/24e2b62e1c345a36f9cf43b7fc34b650.jpg)

查询余额，效果如下：

![`img.kongyixueyuan.com/1118_%E6%9F%A5%E8%AF%A2.png`](img/e1da3e77505d49640ed51da9463e8660.jpg)

**NODE 3002**

打开一个新的终端窗口，将它的 `NODE_ID` 设置为 3002，然后生成一个钱包。这会是一个矿工节点。

在终端输入以下命令，初始化区块链：

```go
hanru:mypublicchain ruby$ cd day08_10_Net/
hanru:day08_10_Net ruby$ export NODE_ID=3002
hanru:day08_10_Net ruby$ cp blockchain_genesis.db blockchain_3002.db
hanru:day08_10_Net ruby$ ./bc printchain 
```

运行效果如下：

![`img.kongyixueyuan.com/1119_%E7%9F%BF%E5%B7%A5%E8%8A%82%E7%82%B9.png`](img/1acb78edbe7c819d55ed3f07e699db7a.jpg)

创建地址，启动节点指定奖励地址：

```go
hanru:day08_10_Net ruby$ ./bc createwallet
hanru:day08_10_Net ruby$ ./bc startnode -miner '15PJzTGsHyBmkKG7RvN854TMwJfWdLaUem' 
```

启动矿工节点后，会先同步主节点的数据，效果如下:

![`img.kongyixueyuan.com/1120_%E7%9F%BF%E5%B7%A5%E8%8A%82%E7%82%B9.png`](img/49f52408211b1495edc97149c8f2b085.jpg)

**NODE 3001**

保证主节点和矿工节点启动，然后切换到钱包节点，进行转账，在终端输入命令如下：

```go
hanru:day08_10_Net ruby$ ./bc send -from '["15bEARdUzej9QWNc2th55LWhjq2vhVbtok"]' -to '["19ChjcK8nGq2am6WZQMFmhZs7oEWsjcDxP"]' -amount '["5"]' 
```

本次转账没有理解挖矿，所以会交由矿工节点进行挖矿，效果如下：

![`img.kongyixueyuan.com/1121_%E9%92%B1%E5%8C%85%E8%8A%82%E7%82%B9.png`](img/e8ec891624b6d0a9b0c4d3ff1784df7f.jpg)

**NODE 3002**

迅速切换到矿工节点，你会看到挖出了一个新块！同时查询余额是最新的数据。

```go
hanru:day08_10_Net ruby$ ./bc printchain

// 钱包节点的地址
hanru:day08_10_Net ruby$ ./bc getbalance -address '15bEARdUzej9QWNc2th55LWhjq2vhVbtok'
// 矿工节点的地址，挖矿指定了奖励地址
hanru:day08_10_Net ruby$ ./bc getbalance -address '15PJzTGsHyBmkKG7RvN854TMwJfWdLaUem' 
```

运行效果如下：

![`img.kongyixueyuan.com/1122_%E7%9F%BF%E5%B7%A5%E8%8A%82%E7%82%B9.png`](img/581d6615aa6938a598709b8d985fc4cf.jpg)

查询余额：

![`img.kongyixueyuan.com/1123_%E7%9F%BF%E5%B7%A5%E8%8A%82%E7%82%B9.png`](img/c3012a26f92db6ecd23ea977b994dd14.jpg)

**NODE 3001**

切换到钱包节点并启动：

```go
hanru:day08_10_Net ruby$ ./bc startnode
hanru:day08_10_Net ruby$ ./bc getbalance -address '15bEARdUzej9QWNc2th55LWhjq2vhVbtok'
hanru:day08_10_Net ruby$ ./bc getbalance -address '19ChjcK8nGq2am6WZQMFmhZs7oEWsjcDxP' 
```

它会下载最近挖出来的块，暂停节点并检查余额：

运行效果如下：

![`img.kongyixueyuan.com/1124_%E9%92%B1%E5%8C%85%E8%8A%82%E7%82%B9.png`](img/4b975dea8129e854f376346be6d56f57.jpg)

钱包节点查询余额：

![`img.kongyixueyuan.com/1125_%E9%92%B1%E5%8C%85%E8%8A%82%E7%82%B9.png`](img/d1c229b5c5dd1586b1869280d9161d15.jpg)

就是这么多了！

## 5\. 总结

本章节的学习，我们实现了简易版的网络，能够同步节点之间的数据消息，尽管这个过程比较复杂。我们已经尽可能的简化了，仅仅是通过端口来模拟不同的节点。实现了主节点，钱包节点和矿工节点之间的数据传递。

最后，这是本系列的最后一篇文章了，希望本文已经回答了关于比特币技术的一些问题，也给读者提出了一些问题，这些问题你可以自行寻找答案。在比特币技术中还有隐藏着很多有趣的事情！好运！

[项目源代码](https://github.com/rubyhan1314/PublicChain)