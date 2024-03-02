# 第七章 比特币地址生成解析

# 地址(Address)

在上一篇文章中，我们已经初步实现了交易。相信你应该了解了交易中的一些天然属性，这些属性没有丝毫“个人”色彩的存在：在比特币中，没有用户账户，不需要也不会在任何地方存储个人数据（比如姓名，护照号码或者 SSN）。但是，我们总要有某种途径识别出你是交易输出的所有者（也就是说，你拥有在这些输出上锁定的币）。这就是比特币地址（address）需要完成的使命。在上一篇中，我们把一个由用户定义的任意字符串当成是地址，现在我们将要实现一个跟比特币一样的真实地址。

## 1\. 课程目标

1.  学会什么是 Base58 编码和解码

2.  学会创建私钥和公钥

3.  学会使用公钥生成钱包地址

4.  学会将钱包地址进行持久化存储

## 2\. 项目代码及效果展示

### 2.1 项目代码结构

![`img.kongyixueyuan.com/0701_%E9%A1%B9%E7%9B%AE%E7%BB%93%E6%9E%84.png`](img/7b49a78ed985250ed3a3aaaa9fa0781a.jpg)

### 2.2 项目运行结果

![`img.kongyixueyuan.com/0702_%E8%BF%90%E8%A1%8C%E6%95%88%E6%9E%9C.gif`](img/1be1775dbb7fb1bca9763697378525f5.jpg)

## 3\. 创建项目

### 3.1 创建工程

首先打开 Goland 开发工具

打开工程：`mypublicchain`

创建项目：将上一次的项目代码，`day03_05_Transaction`，复制为`day04_06_Address`

> 说明：我们每一章节的项目代码，都是在上一个章节上进行添加。所以拷贝上一次的项目代码，然后进行新内容的添加或修改。

### 3.2 代码实现

#### 3.2.1 新建 go 文件：`base58.go`

打开`day04_06_Address`目录里的 BLC 包，新建`base58.go`文件。添加 Base58 的编码和解码方法，代码如下:

```go
package BLC
import (
    "math/big"
    "bytes"
)

//base64

/*
ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/
0(零)，O(大写的 o)，I(大写的 i)，l(小写的 L),+,/
 */

var b58Alphabet = []byte("123456789ABCDEFGHJKLMNPQRSTUVWXYZabcdefghijkmnopqrstuvwxyz")

//字节数组转 Base58，编码
func Base58Encode(input []byte)[]byte{
    var result [] byte
    x := big.NewInt(0).SetBytes(input)

    base :=big.NewInt(int64(len(b58Alphabet)))
    zero:=big.NewInt(0)
    mod:= &big.Int{}
    for x.Cmp(zero) !=0{
        x.DivMod(x,base,mod)
        result = append(result,b58Alphabet[mod.Int64()])
    }
    ReverseBytes(result)
    for b:=range input{
        if b == 0x00{
            result = append([]byte{b58Alphabet[0]},result...)
        }else {
            break
        }
    }

    return result

}

//Base58 转字节数组，解码
func Base58Decode(input[] byte)[]byte{
    result:=big.NewInt(0)
    zeroBytes := 0
    for b:=range input{
        if b == 0x00{
            zeroBytes++
        }
    }
    payload := input[zeroBytes:]
    for _, b := range payload {
        charIndex := bytes.IndexByte(b58Alphabet, b)
        result.Mul(result, big.NewInt(58))
        result.Add(result, big.NewInt(int64(charIndex)))
    }

    decoded := result.Bytes()
    decoded = append(bytes.Repeat([]byte{byte(0x00)}, zeroBytes), decoded...)

    return decoded
} 
```

#### 3.2.2 创建 go 文件：`Wallet.go`

打开`day04_06_Address`目录里的 BLC 包，创建`Wallet.go`文件。在`Wallet.go`文件中编写代码如下：

```go
package BLC

import (
    "crypto/ecdsa"
    "crypto/elliptic"
    "crypto/rand"
    "log"
    "crypto/sha256"
    "golang.org/x/crypto/ripemd160"
    "bytes"
)

//step1：创建一个钱包
type Wallet struct {
    //1.私钥
    PrivateKey ecdsa.PrivateKey
    //2.公钥
    PublicKey [] byte
}

//step2：产生一对密钥
func newKeyPair() (ecdsa.PrivateKey, []byte) {
    /*
    1.通过椭圆曲线算法，随机产生私钥
    2.根据私钥生成公钥

    elliptic:椭圆
    curve：曲线
    ecc：椭圆曲线加密
    ecdsa：elliptic curve  digital signature algorithm，椭圆曲线数字签名算法
        比特币使用 SECP256K1 算法，p256 是 ecdsa 算法中的一种

     */
    //椭圆加密
    curve := elliptic.P256() //椭圆加密算法，得到一个椭圆曲线值，全称：SECP256k1
    private, err := ecdsa.GenerateKey(curve, rand.Reader)
    if err != nil {
        log.Panic(err)
    }
    //生成公钥
    pubKey := append(private.PublicKey.X.Bytes(), private.PublicKey.Y.Bytes()...)
    return *private, pubKey
}

//step3：提供一个方法用于获取钱包
func NewWallet() *Wallet {
    privateKey, publicKey := newKeyPair()
    //fmt.Println("privateKey:", privateKey, ",publicKey:", publicKey)
    return &Wallet{privateKey, publicKey}
}

//step4：根据一个公钥获取对应的地址
/*
将公钥 sha2561 次，再 160，1 次
然后 version+hash
 */
func (w *Wallet) GetAddress() [] byte {
    //1.先将公钥进行一次 hash256，一次 160,得到 pubKeyHash
    pubKeyHash := PubKeyHash(w.PublicKey)
    //2.添加版本号
    versioned_payload := append([]byte{version}, pubKeyHash...)
    // 3.获取校验和，将 pubKeyhash，两次 sha256 后，取前 4 位
    checkSumBytes := CheckSum(versioned_payload)
    full_payload := append(versioned_payload, checkSumBytes...)
    //fmt.Println(len(full_payload))
    //4.Base58
    address := Base58Encode(full_payload)
    return address

}

//一次 sha256,再一次 ripemd160,得到 publicKeyHash
func PubKeyHash(publicKey [] byte) []byte {
    //1.sha256
    hasher := sha256.New()
    hasher.Write(publicKey)
    hash := hasher.Sum(nil)

    //2.ripemd160
    ripemder := ripemd160.New()
    ripemder.Write(hash)
    pubKeyHash := ripemder.Sum(nil)

    //返回
    return pubKeyHash
}

const version = byte(0x00)
const addressChecksumLen = 4

//获取验证码：将公钥哈希两次 sha256,取前 4 位，就是校验和
func CheckSum(payload []byte) []byte {
    firstSHA := sha256.Sum256(payload)
    secondSHA := sha256.Sum256(firstSHA[:])
    return secondSHA[:addressChecksumLen]
}

//判断地址是否有效
/*
根据地址，base58 解码后获取 byte[],获取校验和数组
使用
 */
func  IsValidForAddress(address []byte) bool {
    full_payload := Base58Decode(address)
    //fmt.Println("检验 version_public_checksumBytes:",full_payload)
    checkSumBytes := full_payload[len(full_payload)-addressChecksumLen:]
    //fmt.Println("检验 checkSumBytes：",checkSumBytes)
    versioned_payload := full_payload[:len(full_payload)-addressChecksumLen]
    //fmt.Println("检验 version_ripemd160:",versioned_payload)
    checkBytes := CheckSum(versioned_payload)
    //fmt.Println("检验 checkBytes：",checkBytes)
    if bytes.Compare(checkSumBytes, checkBytes) == 0 {
        return true
    }
    return false
} 
```

#### 3.2.3 创建`Wallets.go`文件

打开`day04_06_Address`目录里的 BLC 包，创建`Wallets.go`文件。在`Wallets.go`文件中编写代码如下：

添加代码如下：

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
const walletFile = "Wallets.dat"

func NewWallets() *Wallets {
    //wallets := &WalletsMap{}
    //wallets.WalletsMap = make(map[string]*Wallet)
    //return wallets
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
func (ws *Wallets) CreateNewWallet() {
    wallet := NewWallet()
    fmt.Printf("创建钱包地址：%s\n", wallet.GetAddress())
    ws.WalletsMap[string(wallet.GetAddress())] = wallet

    //将钱包保存
    ws.SaveWallets()
}

/*
要让数据对象能在网络上传输或存储，我们需要进行编码和解码。
现在比较流行的编码方式有 JSON,XML 等。然而，Go 在 gob 包中为我们提供了另一种方式，该方式编解码效率高于 JSON。
gob 是 Golang 包自带的一个数据结构序列化的编码/解码工具
 */
func (ws *Wallets) SaveWallets() {
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

#### 3.2.4 修改`CLI.go`文件

打开`day04_06_Address`目录里的 BLC 包，修改`CLI.go`文件。添加两个命令，创建钱包地址和打印钱包地址。

修改步骤：

```go
修改步骤：
step1：修改 printUsage()方法，添加两个命令说明
step2：修改 Run()方法，增加两个命令，并解析
    createWalletCmd := flag.NewFlagSet("createwallet", flag.ExitOnError)
    addressListsCmd := flag.NewFlagSet("addresslists",flag.ExitOnError) 
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

    //1.创建 flagset 标签对象
    createWalletCmd := flag.NewFlagSet("createwallet", flag.ExitOnError)
    addressListsCmd := flag.NewFlagSet("addresslists",flag.ExitOnError)

    sendBlockCmd := flag.NewFlagSet("send", flag.ExitOnError)
    //fmt.Printf("%T\n",addBlockCmd) //*FlagSet
    printChainCmd := flag.NewFlagSet("printchain", flag.ExitOnError)
    createBlockChainCmd := flag.NewFlagSet("createblockchain", flag.ExitOnError)
    getBalanceCmd := flag.NewFlagSet("getbalance", flag.ExitOnError)

    //2.设置标签后的参数
    //flagAddBlockData:= addBlockCmd.String("data","helloworld..","交易数据")
    flagFromData := sendBlockCmd.String("from", "", "转帐源地址")
    flagToData := sendBlockCmd.String("to", "", "转帐目标地址")
    flagAmountData := sendBlockCmd.String("amount", "", "转帐金额")
    flagCreateBlockChainData := createBlockChainCmd.String("address", "", "创世区块交易地址")
    flagGetBalanceData := getBalanceCmd.String("address", "", "要查询的某个账户的余额")

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
        fmt.Println(*flagFromData)
        fmt.Println(*flagToData)
        fmt.Println(*flagAmountData)
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

        cli.send(from, to, amount)
    }
    if printChainCmd.Parsed() {
        cli.printChains()
    }

    if createBlockChainCmd.Parsed() {
        //if *flagCreateBlockChainData == "" {
        if !IsValidForAddress([]byte(*flagCreateBlockChainData)){
            fmt.Println("创建地址无效")
            printUsage()
            os.Exit(1)
        }
        cli.createGenesisBlockchain(*flagCreateBlockChainData)
    }

    if getBalanceCmd.Parsed() {
        //if *flagGetBalanceData == "" {
        if !IsValidForAddress([]byte(*flagGetBalanceData)){
            fmt.Println("查询地址无效")
            printUsage()
            os.Exit(1)
        }
        cli.getBalance(*flagGetBalanceData)

    }

    if createWalletCmd.Parsed() {
        //创建钱包
        cli.createWallet()
    }

    //获取所有的钱包地址
    if addressListsCmd.Parsed(){
        cli.addressLists()
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
    fmt.Println("\tsend -from From -to To -amount Amount - 交易数据")
    fmt.Println("\tprintchain - 输出信息:")
    fmt.Println("\tgetbalance -address DATA -- 查询账户余额")
} 
```

#### 3.2.5 新建`CLI_createwallet.go`

打开`day04_06_Address`目录里的 BLC 包，新建`CLI_createwallet.go`文件。

并编写代码如下：

```go
package BLC

func (cli *CLI) createWallet(){
    wallets:= NewWallets()
    wallets.CreateNewWallet()
} 
```

#### 3.2.6 新建`CLI_addresslists.go`

打开`day04_06_Address`目录里的 BLC 包，新建`CLI_addresslists.go`文件。

并编写代码如下：

```go
package BLC

import "fmt"

func (cli *CLI)addressLists(){
    fmt.Println("打印所有的钱包地址。。")
    //获取
    Wallets:=NewWallets()
    for address,_ := range Wallets.WalletsMap{
        fmt.Println("address:",address)
    }
} 
```

#### 3.2.7 `main.go`无需修改

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

## 4\. 钱包地址讲解

### 4.1 比特币地址

这就是一个真实的比特币地址：[1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa](https://blockchain.info/address/1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa)。这是史上第一个比特币地址，据说属于中本聪。比特币地址是完全公开的，如果你想要给某个人发送币，只需要知道他的地址就可以了。但是，地址（尽管地址也是独一无二的）并不是用来证明你是一个“钱包”所有者的信物。实际上，所谓的地址，只不过是将公钥表示成人类可读的形式而已，因为原生的公钥人类很难阅读。在比特币中，你的身份（identity）就是一对（或者多对）保存在你的电脑（或者你能够获取到的地方）上的公钥（public key）和私钥（private key）。比特币基于一些加密算法的组合来创建这些密钥，并且保证了在这个世界上没有其他人能够取走你的币，除非拿到你的密钥。下图是创建钱包后到产生地址的流程图。

![`img.kongyixueyuan.com/0703_%E6%B5%81%E7%A8%8B%E5%9B%BE.png`](img/a830e702f3d26da29340acd908693ec5.jpg)

> 1、对于比特币来说，钱不是支付给个人的，而是支付给某一把私钥。这就是交易匿名性的根本原因，因为没有人知道，那些私钥背后的主人是谁。所以，比特币交易的第一件事，就是你必须拥有自己的公钥和私钥。
> 
> 2、去网上那些比特币交易所开户，它们会让你首先生成一个比特币钱包（wallet）。这个钱包不是用来存放比特币，而是存放你的公钥和私钥。软件会帮你生成这两把钥匙，然后放在钱包里面。根据协议，公钥的长度是 512 位。这个长度不太方便传播，因此协议又规定，要为公钥生成一个 160 位的指纹。所谓指纹，就是一个比较短的、易于传播的哈希值。160 位是二进制，写成十六进制，大约是 26 到 35 个字符，比如 1Ko29e6QcUXzhrtHD5aizMa6ustL88RTWR。这个字符串就叫做钱包的地址，它是唯一的，即每个钱包的地址肯定都是不一样的。
> 
> 3、你向别人收钱时，只要告诉对方你的钱包地址即可，对方向这个地址付款。由于你是这个地址的拥有者，所以你会收到这笔钱。由于你是否拥有某个钱包地址，是由私钥证明的，所以一定要保护好私钥。同样的，你向他人支付比特币，千万不能写错他人的钱包地址，否则你的比特币就支付到了另一个不同的人了。

下面，让我们来讨论一下这些算法到底是什么。

### 4.2 公钥加密

公钥加密（public-key cryptography）算法使用的是成对的密钥：公钥和私钥。公钥并不是敏感信息，可以告诉其他人。但是，私钥绝对不能告诉其他人：只有所有者（owner）才能知道私钥，能够识别，鉴定和证明所有者身份的就是私钥。在加密货币的世界中，你的私钥代表的就是你，私钥就是一切。

本质上，比特币钱包也只不过是这样的密钥对而已。当你安装一个钱包应用，或是使用一个比特币客户端来生成一个新地址时，它就会为你生成一对密钥。在比特币中，谁拥有了私钥，谁就可以控制所有发送到这个公钥的币。

私钥和公钥只不过是随机的字节序列，因此它们无法在屏幕上打印，人类也无法通过肉眼去读取。这就是为什么比特币使用了一个转换算法，将公钥转化为一个人类可读的字符串（也就是我们看到的地址）。

> 如果你用过比特币钱包应用，很可能它会为你生成一个助记符。这样的助记符可以用来替代私钥，并且可以被用于生成私钥。[BIP-039](https://github.com/bitcoin/bips/blob/master/bip-0039.mediawiki) 已经实现了这个机制。

### 4.3 椭圆曲线加密

正如之前提到的，公钥和私钥是随机的字节序列。私钥能够用于证明持币人的身份，需要有一个条件：随机算法必须生成真正随机的字节。因为没有人会想要生成一个私钥，而这个私钥意外地也被别人所有。

比特币使用椭圆曲线来产生私钥。椭圆曲线是一个复杂的数学概念，我们并不打算在这里作太多解释（如果你真的十分好奇，可以查看[这篇文章](http://andrea.corbellini.name/2015/05/17/elliptic-curve-cryptography-a-gentle-introduction/)，注意：有很多数学公式！）我们只要知道这些曲线可以生成非常大的随机数就够了。在比特币中使用的曲线可以随机选取在 0 与 2 ^ 2 ^ 56（大概是 10⁷⁷, 而整个可见的宇宙中，原子数在 10⁷⁸ 到 10⁸² 之间） 的一个数。有如此高的一个上限，意味着几乎不可能发生有两次生成同一个私钥的事情。

椭圆曲线算法（ECC，Elliptic Curve Cryptography）

*   椭圆曲线算法是不可逆的，很容易向一个方向计算，但是不可以向相反方向倒推。
*   比特币使用 SECP256K1 算法（椭圆曲线算法的一种），将私钥生成公钥。
*   SECP256K1 是不可逆的，因此公钥公开暴露，也对私钥的安全性不会造成影响。

比特币使用的是 ECDSA（Elliptic Curve Digital Signature Algorithm）算法来对交易进行签名，我们也会使用该算法。

**比特币钱包随机生成私钥的安全性**

每个用户的私钥是由比特币钱包随机生成的。可能有人会有疑问，这样随机生成的私钥安不安全，会不会我随机生成私钥跟别人的私钥恰好重复了？这个你不用过于担心，因为，比特币私钥是一串很长的文本，理论上比特币私钥的总数是：1461501637330902918203684832716283019655932542976

![`img.kongyixueyuan.com/0704_%E9%9A%8F%E6%9C%BA%E6%95%B0.png`](img/58abbe7014a56e1adf3e1211b4c6087b.jpg)

这是个什么概念？地球上的沙子是 7.5 乘以 10 的 18 次方，然后你想象一下，每粒沙子又是一个地球，这时的沙子总数是 56.25 乘以 10 的 36 次方，仍然远小于比特币私钥的总数。所以在这样一个庞大的地址空间下，私钥重复的概率微乎其微。

![`img.kongyixueyuan.com/0705_%E9%9A%8F%E6%9C%BA%E6%95%B02.png`](img/b124b2b6c1e8f918ff045795992c06bd.jpg)

### 4.4 Base64 和 Base58 编码和解码

Base64 就是一种基于 64 个可打印字符来表示二进制数据的方法

*   Base64 使用了 26 个小写字母、26 个大写字母、10 个数字以及两个符号（例如“+”和“/”），用于在电子邮件这样的基于文本的媒介中传输二进制数据。

*   Base64 通常用于编码邮件中的附件。

Base64 字符集：

```go
ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/ 
```

![`img.kongyixueyuan.com/0706_base64.png`](img/f6ac2957f4f6b49771585310a128cb2b.jpg)

Base58 是一种基于文本的二进制编码格式，是用于 Bitcoin 中使用的一种独特的编码方式，主要用于产生 Bitcoin 的钱包地址。

*   相比 Base64，Base58 不使用数字"0"，大写字母"O"，大写字母"I"和小写字母"l"，以及"+"和"/"符号。目的就是去除容易混淆的字符。
*   这种编码格式不仅实现了数据压缩，保持了易读性，还具有错误诊断功能。

Base58 字符集：

```go
ABCDEFGHJKLMNPQRSTUVWXYZabcdefghijkmnopqrstuvwxyz123456789 
```

![`img.kongyixueyuan.com/0707_base58-%E8%A1%A8.jpg`](img/9c3d2c621dcdfba2fa0918c430d52e76.jpg)

**Base58Check**

Base58Check 是一种常用在比特币中的 Base58 编码格式，增加了错误校验码来检查数据在转录中出现的错误。在 Base58Check 中，对数据添加了一个称作“版本字节”的前缀，这个前缀用来明确需要编码的数据的类型。

Base58Check 的作用

*   既然有了 Base58 编码，已经不会搞错 0 和 O, 1 和 l 和 I，也把大整数转换成了可读字符串，为什么还要再有 Base58Check 这个环节呢？
*   假设一种情况，你在程序中输入一个 Base58 编码的地址，尽管你已经不会搞错 0 和 O, 1 和 l 和 I，但是万一你不小心输错一个字符，或者少写多写一个字符，会咋样？你可能会说，没啥大不了的，错个字符而已，这不是很常见嘛，重新输入不就可以了吗？但是当用户给一个比特币地址转账，如果输入错误，那么对方就不会收到资金，更关键的是该笔资金发给了一个根本不存在的比特币地址，那么这笔资金也就永远不可能被交易，也就是说比特币丢失了。
*   校验码长 4 个字节，添加到需要编码的数据之后。
*   校验码是从需要编码的数据的哈希值中得到的，所以可以用来检测并避免转录和输入中产生的错误。
*   使用 Base58check 编码格式时，程序会计算原始数据的校验码并和自带的校验码进行对比，二者若不匹配则表明有错误产生。
*   实际上，在比特币交易中，都会校验比特币地址是否合法，如果经过 Base58Check 的比特币地址被比特币钱包程序判定是无效的，当然会阻止交易继续进行，就避免了资金损失。

### 4.5 生成地址

回到上面提到的比特币地址：1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa 。现在，我们已经知道了这是公钥用人类可读的形式表示而已。如果我们对它进行解码，就会看到公钥的本来面目（16 进制表示的字节）：0062E907B15CBF27D5425399EBF6F0FB50EBB88F18C29B7D93

比特币使用 Base58 算法将公钥转换成人类可读的形式。这个算法跟著名的 Base64 很类似，区别在于它使用了更短的字母表：为了避免一些利用字母相似性的攻击，从字母表中移除了一些字母。也就是，没有这些符号：0(零)，O(大写的 o)，I(大写的 i)，l(小写的 L)，因为这几个字母看着很像。另外，也没有 + 和 / 符号。

比特币地址生成过程中的 Base58Check 的步骤如下：

```go
1\. 获取到公钥
2\. 计算公钥的 SHA-256 哈希值（对公钥进行第一次 hash256 运算）
3\. 计算 RIPEMD-160 哈希值（对第一次 hash256 的结果进行 ripeMD160 运算）
4\. 添加版本前缀（将 ripeMD160 运算的结果前增加版本编号）
5\. 计算两次 hash（对加上版本编号的 hash 值计算两次 hash）
6\. 获取校验码（获取两次 hash 后前四个字节，作为校验码）
7\. 形成比特币地址的 16 进制格式（将校验码作为比特币地址的后缀，与添加版本前缀的 hash 值拼接）
8\. 进行 Base58 编码处理（对比特币地址的 16 进制格式进行 Base58 编码，就形成了比特币地址） 
```

![`img.kongyixueyuan.com/0708_address2.jpg`](img/49c849051fdfa52926012c7766cee29d.jpg)

因此，为了使用 Base58Check 编码格式对数据（数字）进行编码，首先我们要对数据添加一个称作“版本字节”的前缀，这个前缀用来明确需要编码的数据的类型。

> 例如，比特币地址的前缀是 0（十六进制是 0x00），而对私钥编码时前缀是 128（十六进制是 0x80）。

所以公钥解码后包含三个部分：

```go
Version  Public key hash                           Checksum
00       62E907B15CBF27D5425399EBF6F0FB50EBB88F18  C29B7D93 
```

由于哈希函数是单向的（也就说无法逆转回去），所以不可能从一个哈希中提取公钥。不过通过执行哈希函数并进行哈希比较，我们可以检查一个公钥是否被用于哈希的生成。

### 4.6 Base58 代码实现

好了，所有细节都已就绪，来写代码吧。很多概念只有当写代码的时候，才能理解地更透彻。

由于 Base58 是在货币中特有的，并不是像 Base64 那样是通用编码方式，所以 golang 自带的包中没有实现，(Base64 是有的，在"encoding/base64"包下)，那么就需要我们自己写工具方法实现编码和解码。

在 BLC 包下，新建一个 go 文件，命名为 base58.go，并添加代码如下：

```go
package BLC
import (
    "math/big"
    "bytes"
)

//base64

/*
ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/
0(零)，O(大写的 o)，I(大写的 i)，l(小写的 L),+,/
 */

var b58Alphabet = []byte("123456789ABCDEFGHJKLMNPQRSTUVWXYZabcdefghijkmnopqrstuvwxyz")

//字节数组转 Base58，解码
func Base58Encode(input []byte)[]byte{
    var result [] byte
    x := big.NewInt(0).SetBytes(input)

    base :=big.NewInt(int64(len(b58Alphabet)))
    zero:=big.NewInt(0)
    mod:= &big.Int{}
    for x.Cmp(zero) !=0{
        x.DivMod(x,base,mod)
        result = append(result,b58Alphabet[mod.Int64()])
    }
    ReverseBytes(result)
    for b:=range input{
        if b == 0x00{
            result = append([]byte{b58Alphabet[0]},result...)
        }else {
            break
        }
    }

    return result

}

//Base58 转字节数组，解码
func Base58Decode(input[] byte)[]byte{
    result:=big.NewInt(0)
    zeroBytes := 0
    for b:=range input{
        if b == 0x00{
            zeroBytes++
        }
    }
    payload := input[zeroBytes:]
    for _, b := range payload {
        charIndex := bytes.IndexByte(b58Alphabet, b)
        result.Mul(result, big.NewInt(58))
        result.Add(result, big.NewInt(int64(charIndex)))
    }

    decoded := result.Bytes()
    decoded = append(bytes.Repeat([]byte{byte(0x00)}, zeroBytes), decoded...)

    return decoded
} 
```

有一个点需要单独强调一下：

```go
for b:=range input{
        if b == 0x00{
            result = append([]byte{b58Alphabet[0]},result...)
        }else {
            break
        }
    } 
```

对于 Base58 编码，最后总会拼接下标 0 对应的 Base58 字符，就是 1。所以我们所产生的地址的第一位，都是 1。比如：1NH3bAuMAyXHnCrBmykPcKciBG6W3Dc5vY。

### 4.7 创建钱包生成地址

我们先从钱包 `Wallet` 结构开始：

在 BLC 包下，新建`Wallet.go`文件，并添加 Wallet 结构体对象。

```go
//step1：创建一个钱包
type Wallet struct {
    //1.私钥
    PrivateKey ecdsa.PrivateKey
    //2.公钥
    PublicKey [] byte
}
//step2：产生一对密钥
func newKeyPair() (ecdsa.PrivateKey, []byte) {
    /*
    1.通过椭圆曲线算法，随机产生私钥
    2.根据私钥生成公钥

    elliptic:椭圆
    curve：曲线
    ecc：椭圆曲线加密
    ecdsa：elliptic curve  digital signature algorithm，椭圆曲线数字签名算法
        比特币使用 SECP256K1 算法，p256 是 ecdsa 算法中的一种

     */
    //椭圆加密
    curve := elliptic.P256() //椭圆加密算法，得到一个椭圆曲线值，全称：SECP256k1
    private, err := ecdsa.GenerateKey(curve, rand.Reader)
    if err != nil {
        log.Panic(err)
    }
    //生成公钥
    pubKey := append(private.PublicKey.X.Bytes(), private.PublicKey.Y.Bytes()...)
    return *private, pubKey
}

//step3：提供一个方法用于获取钱包
func NewWallet() *Wallet {
    privateKey, publicKey := newKeyPair()
    //fmt.Println("privateKey:", privateKey, ",publicKey:", publicKey)
    return &Wallet{privateKey, publicKey}
} 
```

一个钱包只有一个密钥对而已。我们需要 `Wallets` 类型来保存多个钱包的组合，将它们保存到文件中，或者从文件中进行加载。`Wallet` 的构造函数会生成一个新的密钥对。`newKeyPair` 函数非常直观：ECDSA 基于椭圆曲线，所以我们需要一个椭圆曲线。接下来，使用椭圆生成一个私钥，然后再从私钥生成一个公钥。有一点需要注意：在基于椭圆曲线的算法中，公钥是曲线上的点。因此，公钥是 X，Y 坐标的组合。在比特币中，这些坐标会被连接起来，然后形成一个公钥。

现在，来生成一个地址：

```go
//step4：根据一个公钥获取对应的地址
/*
将公钥 sha2561 次，再 160，1 次
然后 version+hash
 */
func (w *Wallet) GetAddress() [] byte {
    //1.先将公钥进行一次 hash256，一次 160,得到 pubKeyHash
    pubKeyHash := PubKeyHash(w.PublicKey)
    //2.添加版本号
    versioned_payload := append([]byte{version}, pubKeyHash...)
    // 3.获取校验和，将 pubKeyhash，两次 sha256 后，取前 4 位
    checkSumBytes := CheckSum(versioned_payload)
    full_payload := append(versioned_payload, checkSumBytes...)
    //fmt.Println(len(full_payload))
    //4.Base58
    address := Base58Encode(full_payload)
    return address

}

//一次 sha256,再一次 ripemd160,得到 publicKeyHash
func PubKeyHash(publicKey [] byte) []byte {
    //1.sha256
    hasher := sha256.New()
    hasher.Write(publicKey)
    hash := hasher.Sum(nil)

    //2.ripemd160
    ripemder := ripemd160.New()
    ripemder.Write(hash)
    pubKeyHash := ripemder.Sum(nil)

    //返回
    return pubKeyHash
}

const version = byte(0x00)
const addressChecksumLen = 4

//获取验证码：将公钥哈希两次 sha256,取前 4 位，就是校验和
func CheckSum(payload []byte) []byte {
    firstSHA := sha256.Sum256(payload)
    secondSHA := sha256.Sum256(firstSHA[:])
    return secondSHA[:addressChecksumLen]
} 
```

至此，就可以得到一个**真实的比特币地址**，你甚至可以在 [blockchain.info](https://blockchain.info/) 查看它的余额。不过我可以负责任地说，无论生成一个新的地址多少次，检查它的余额都是 0。这就是为什么选择一个合适的公钥加密算法是如此重要：考虑到私钥是随机数，生成同一个数字的概率必须是尽可能地低。理想情况下，必须是低到“永远”不会重复。

我们还可以添加一个方法，用于验证一个地址的有效性。验证原理就是，将地址进行 Base58 解码，得到数据：版本号+hash 数据+checksum，将版本号+hash 数据，重新生成新的 checksum，和原来的 checksum 进行比较，如果不同，说明地址无效。

```go
 //判断地址是否有效
/*
根据地址，base58 解码后获取 byte[],获取校验和数组
使用
 */
func  IsValidForAddress(address []byte) bool {
    full_payload := Base58Decode(address)
    //fmt.Println("检验 version_public_checksumBytes:",full_payload)
    checkSumBytes := full_payload[len(full_payload)-addressChecksumLen:]
    //fmt.Println("检验 checkSumBytes：",checkSumBytes)
    versioned_payload := full_payload[:len(full_payload)-addressChecksumLen]
    //fmt.Println("检验 version_ripemd160:",versioned_payload)
    checkBytes := CheckSum(versioned_payload)
    //fmt.Println("检验 checkBytes：",checkBytes)
    if bytes.Compare(checkSumBytes, checkBytes) == 0 {
        return true
    }
    return false
} 
```

接下来，在`CLI.go`文件，修改 Run()，添加一个命令，用于创建钱包：

```go
func (cli *CLI) Run() {
    //判断命令行参数的长度
    isValidArgs()

    //1.创建 flagset 标签对象
    createWalletCmd := flag.NewFlagSet("createwallet", flag.ExitOnError)

    ...

    //3.解析
    switch os.Args[1] {
    ...

    case "createwallet":
        err := createWalletCmd.Parse(os.Args[2:])
        if err != nil {
            log.Panic(err)
        }

    default:
        printUsage()
        os.Exit(1) //退出
    }

    ...

    if createWalletCmd.Parsed() {
        //创建钱包
        cli.createWallet()
    }

} 
```

然后创建一个 go 文件，`CLI_createwallet.go`，并添加代码如下：

```go
package BLC

func (cli *CLI) createWallet(){
    wallets:= NewWallets()
    wallets.CreateNewWallet()
} 
```

接下来，我们代码测试一下：

在终端中输入以下命令：

```go
hanru:day04_06_Address ruby$ ./bc createwallet 
```

运行效果如下：

![`img.kongyixueyuan.com/0709_%E5%9C%B0%E5%9D%80%E8%BF%90%E8%A1%8C%E7%BB%93%E6%9E%9C.png`](img/e5e75a190a8723cfd289dab905c58a11.jpg)

另外，注意：你并不需要连接到一个比特币节点来获得一个地址。地址生成算法使用的多种开源算法可以通过很多编程语言和库实现。

### 4.8 打印钱包地址

一个钱包对象，存储了一对私钥和公钥，公钥生成一个地址。现在我们可以创建一个钱包集合，可以存储多个地址。

新建一个`Wallets.go`文件，并添加 Wallets 结构体：

```go
//1.创建钱包
type Wallets struct {
    WalletsMap map[string]*Wallet
} 
```

对于 map 集合，一个钱包地址，对应了一个钱包对象。

目前的钱包地址创建后，程序结束后内存就销毁了，那么我们需要将创建好的钱包地址，存储到本地文件中，进行持久化保存。

```go
 //2.创建一个钱包集合
//创建钱包集合:文件中存在从文件中读取，否则新建一个
const walletFile = "Wallets.dat"

func NewWallets() *Wallets {
    //wallets := &WalletsMap{}
    //wallets.WalletsMap = make(map[string]*Wallet)
    //return wallets
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
func (ws *Wallets) CreateNewWallet() {
    wallet := NewWallet()
    fmt.Printf("创建钱包地址：%s\n", wallet.GetAddress())
    ws.WalletsMap[string(wallet.GetAddress())] = wallet

    //将钱包保存
    ws.SaveWallets()
}

/*
要让数据对象能在网络上传输或存储，我们需要进行编码和解码。
现在比较流行的编码方式有 JSON,XML 等。然而，Go 在 gob 包中为我们提供了另一种方式，该方式编解码效率高于 JSON。
gob 是 Golang 包自带的一个数据结构序列化的编码/解码工具
 */
func (ws *Wallets) SaveWallets() {
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

接下来，我们一步一步讲解钱包地址持久化的过程。

首先，我们需要先定义一个文件：`Wallets.dat`，用于存储钱包集合中的数据。然后创建了一个`NewWallets()`方法，用于获取一个 Wallets 对象，而 Wallets 里存储钱包地址对应钱包对象。那么我们首先判断钱包文件是否存在，如果不存在，我们就直接创建一个空的 Wallets 钱包集合对象。

```go
if _, err := os.Stat(walletFile); os.IsNotExist(err) {
        fmt.Println("文件不存在")
        wallets := &Wallets{}
        wallets.WalletsMap = make(map[string]*Wallet)
        return wallets
    } 
```

如果文件存在，那么我们应该从文件中读取钱包数据。

```go
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
```

当获取到一个 Wallets 钱包集合对象后，可以创建新的钱包对象以及地址，并进行存储。所以创建一个`CreateNewWallet()`方法，并添加代码如下：

```go
//3.创建一个新钱包
func (ws *Wallets) CreateNewWallet() {
    wallet := NewWallet()
    fmt.Printf("创建钱包地址：%s\n", wallet.GetAddress())
    ws.WalletsMap[string(wallet.GetAddress())] = wallet

    //将钱包保存
    ws.SaveWallets()
} 
```

每当创建一个钱包对象后，都应该将该对象存入到文件中，所以接下来我们再创建一个`SaveWallets()`方法，用于保存数据，代码如下：

```go
/*
要让数据对象能在网络上传输或存储，我们需要进行编码和解码。
现在比较流行的编码方式有 JSON,XML 等。然而，Go 在 gob 包中为我们提供了另一种方式，该方式编解码效率高于 JSON。
gob 是 Golang 包自带的一个数据结构序列化的编码/解码工具
 */
func (ws *Wallets) SaveWallets() {
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

最后，我们在`CLI.go`中，添加一个命令，用于打印出 Wallets 钱包集合中的所有的地址。

修改 Run()方法代码如下：

```go
//step2：添加 Run 方法
func (cli *CLI) Run() {
    //判断命令行参数的长度
    isValidArgs()

    //1.创建 flagset 标签对象

    addressListsCmd := flag.NewFlagSet("addresslists",flag.ExitOnError)

    ...

    //3.解析
    switch os.Args[1] {
    ...
    case "addresslists":
        err := addressListsCmd.Parse(os.Args[2:])
        if err != nil {
            log.Panic(err)
        }
    default:
        printUsage()
        os.Exit(1) //退出
    }

    //获取所有的钱包地址
    if addressListsCmd.Parsed(){
        cli.addressLists()
    }

} 
```

在 BLC 包下，创建一个 go 文件，命名为`CLI_addresslists.go`，并添加代码如下：

```go
package BLC

import "fmt"

func (cli *CLI)addressLists(){
    fmt.Println("打印所有的钱包地址。。")
    //获取
    Wallets:=NewWallets()
    for address,_ := range Wallets.WalletsMap{
        fmt.Println("address:",address)
    }
} 
```

最后，我们测试一下程序，在终端输入以下命令：

```go
hanru:day04_06_Address ruby$ ./bc addresslists 
```

运行效果如下:

![`img.kongyixueyuan.com/0710_%E6%89%80%E6%9C%89%E7%9A%84%E5%9C%B0%E5%9D%80.png`](img/4fd22a29ca0ae482e0ed9fcae77d94a8.jpg)

## 5\. 总结

通过本章节的学习，我们知道了如何创建一个钱包，钱包中如何创建一对秘钥。我们可以根据公钥生成钱包地址，这个过程虽然有点繁琐，但是不难理解。首先将公钥，进行一次 sha256，一次 ripemd160，进行 hash 散列，生成公钥 hash(也叫指纹)。再用公钥 hash 前加 1 个 byte 的版本号，一般都是 0x00，然后进行两次 sha256，获取前 4 位，作为 checksum，然后就得到了版本号+公钥 hash+checksum 的数据。最后进行一次 Base58 编码，就得到了钱包地址。

程序中的钱包地址都是存储在 map 中，为了进行持久化，我们还需要将创建好的钱包数据，保存到本地文件中。

[项目源代码](https://github.com/rubyhan1314/PublicChain)