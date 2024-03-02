# 第八章 用 go 语言实现一个 POS 共识机制

## POS 工程简述

在 PoW 中，节点之间通过 hash 的计算力来竞赛以获取下一个区块的记账权，而在 PoS 中，块是已经铸造好的，铸造的过程是基于每个节点(Node)愿意作为抵押的令牌(Token)数量。如果验证者愿意提供更多的令牌作为抵押品，他们就有更大的机会记账下一个区块并获得奖励。

## 实现 POS 主要功能点

*   我们将有一个中心化的 TCP 服务节点，其他节点可以连接该服务器
*   最新的区块链状态将定期广播到每个节点
*   每个节点都能提议建立新的区块
*   基于每个节点的令牌数量，其中一个节点将随机地(以令牌数作为加权值)作为获胜者，并且将该区块添加到区块链中

## 实现 POS

### 设置 TCP 服务器的端口

新建 `.env`，添加如下内容 `PORT=9000`

### 安装依赖软件

```go
$ go get github.com/davecgh/go-spew/spew

$ go get github.com/joho/godotenv 
```

*   `spew` 在控制台中格式化输出相应的结果。

*   `godotenv` 可以从我们项目的根目录的 `.env` 文件中读取数据。

### 引入相应的包

新建 `main.go`，引入相应的包

```go
package main

import (
    "bufio"
    "crypto/sha256"
    "encoding/hex"
    "encoding/json"
    "fmt"
    "io"
    "log"
    "math/rand"
    "net"
    "os"
    "strconv"
    "sync"
    "time"

    "github.com/davecgh/go-spew/spew"
    "github.com/joho/godotenv"
) 
```

### 全局变量

```go
// Block represents each 'item' in the blockchain
type Block struct {
    Index     int
    Timestamp string
    BPM       int
    Hash      string
    PrevHash  string
    Validator string
}

// Blockchain is a series of validated Blocks
var Blockchain []Block
var tempBlocks []Block

// candidateBlocks handles incoming blocks for validation
var candidateBlocks = make(chan Block)

// announcements broadcasts winning validator to all nodes
var announcements = make(chan string)

var mutex = &sync.Mutex{}

// validators keeps track of open validators and balances
var validators = make(map[string]int) 
```

*   `Block` 是每个区块的内容
*   `Blockchain` 是我们的官方区块链，它只是一串经过验证的区块集合。每个区块中的 `PrevHash` 与前面块的 `Hash` 相比较，以确保我们的链是正确的。 `tempBlocks` 是临时存储单元，在区块被选出来并添加到 `BlockChain` 之前，临时存储在这里
*   `candidateBlocks` 是 `Block` 的通道，任何一个节点在提出一个新块时都将它发送到这个通道
*   `announcements` 也是一个通道，我们的主 Go TCP 服务器将向所有节点广播最新的区块链
*   `mutex`是一个标准变量，允许我们控制读/写和防止数据竞争
*   `validators` 是节点的存储 map，同时也会保存每个节点持有的令牌数

### 生成区块

```go
func generateBlock(oldBlock Block, BPM int, address string) (Block, error) {

    var newBlock Block

    t := time.Now()

    newBlock.Index = oldBlock.Index + 1
    newBlock.Timestamp = t.String()
    newBlock.BPM = BPM
    newBlock.PrevHash = oldBlock.Hash
    newBlock.Hash = calculateBlockHash(newBlock)
    newBlock.Validator = address

    return newBlock, nil
} 
```

`generateBlock` 是用来创建新块的。
`newBlock.PrevHash` 存储的是上一个区块的 `Hash`
`newBlock.Hash` 是通过 `calculateBlockHash(newBlock)` 生成的 Hash 。
`newBlock.Validator` 存储的是获取记账权的节点地址

```go
// SHA256 hasing
// calculateHash is a simple SHA256 hashing function
func calculateHash(s string) string {
    h := sha256.New()
    h.Write([]byte(s))
    hashed := h.Sum(nil)
    return hex.EncodeToString(hashed)
}

//calculateBlockHash returns the hash of all block information
func calculateBlockHash(block Block) string {
    record := string(block.Index) + block.Timestamp + string(block.BPM) + block.PrevHash
    return calculateHash(record)
} 
```

`calculateHash` 函数会接受一个 `string` ，并且返回一个`SHA256 hash` 。

`calculateBlockHash` 是对一个 `block` 进行 `hash`，将一个 `block` 的所有字段连接到一起后，再调用 `calculateHash` 将字符串转为 `SHA256 hash` 。

### 验证区块

我们通过检查 `Index` 来确保它们按预期递增。我们也检查以确保我们 `PrevHash` 的确与 `Hash` 前一个区块相同。最后，我们希望通过在当前块上 `calculateBlockHash` 再次运行该函数来检查当前块的散列。

```go
// isBlockValid makes sure block is valid by checking index
// and comparing the hash of the previous block
func isBlockValid(newBlock, oldBlock Block) bool {
    if oldBlock.Index+1 != newBlock.Index {
        return false
    }

    if oldBlock.Hash != newBlock.PrevHash {
        return false
    }

    if calculateBlockHash(newBlock) != newBlock.Hash {
        return false
    }

    return true
} 
```

### 验证者

当一个验证者连接到我们的 TCP 服务，我们需要提供一些函数达到以下目标：

*   输入令牌的余额（之前提到过，我们不做钱包等逻辑)
*   接收区块链的最新广播
*   接收验证者赢得区块的广播信息
*   将自身节点添加到全局的验证者列表中（validators)
*   输入 Block 的 BPM 数据- BPM 是每个验证者的人体脉搏值
*   提议创建一个新的区块

```go
func handleConn(conn net.Conn) {
    defer conn.Close()

    go func() {
        for {
            msg := <-announcements
            io.WriteString(conn, msg)
        }
    }()
    // 验证者地址
    var address string

    // 验证者输入他所拥有的 tokens，tokens 的值越大，越容易获得新区块的记账权
    io.WriteString(conn, "Enter token balance:")
    scanBalance := bufio.NewScanner(conn)
    for scanBalance.Scan() {
        // 获取输入的数据，并将输入的值转为 int
        balance, err := strconv.Atoi(scanBalance.Text())
        if err != nil {
            log.Printf("%v not a number: %v", scanBalance.Text(), err)
            return
        }
        t := time.Now()
        // 生成验证者的地址
        address = calculateHash(t.String())
        // 将验证者的地址和 token 存储到 validators
        validators[address] = balance
        fmt.Println(validators)
        break
    }

    io.WriteString(conn, "\nEnter a new BPM:")

    scanBPM := bufio.NewScanner(conn)

    go func() {
        for {
            // take in BPM from stdin and add it to blockchain after conducting necessary validation
            for scanBPM.Scan() {
                bpm, err := strconv.Atoi(scanBPM.Text())
                // 如果验证者试图提议一个被污染（例如伪造）的 block，例如包含一个不是整数的 BPM，那么程序会抛出一个错误，我们会立即从我们的验证器列表 validators 中删除该验证者，他们将不再有资格参与到新块的铸造过程同时丢失相应的抵押令牌。
                if err != nil {
                    log.Printf("%v not a number: %v", scanBPM.Text(), err)
                    delete(validators, address)
                    conn.Close()
                }

                mutex.Lock()
                oldLastIndex := Blockchain[len(Blockchain)-1]
                mutex.Unlock()

                // 创建新的区块，然后将其发送到 candidateBlocks 通道
                newBlock, err := generateBlock(oldLastIndex, bpm, address)
                if err != nil {
                    log.Println(err)
                    continue
                }
                if isBlockValid(newBlock, oldLastIndex) {
                    candidateBlocks <- newBlock
                }
                io.WriteString(conn, "\nEnter a new BPM:")
            }
        }
    }()

    // 循环会周期性的打印出最新的区块链信息
    for {
        time.Sleep(time.Minute)
        mutex.Lock()
        output, err := json.Marshal(Blockchain)
        mutex.Unlock()
        if err != nil {
            log.Fatal(err)
        }
        io.WriteString(conn, string(output)+"\n")
    }

} 
```

*   `io.WriteString(conn, "Enter token balance:")`允许验证者输入他持有的令牌数量，然后，该验证者被分配一个 `SHA256`地址，随后该验证者地址和验证者的令牌数被添加到验证者列表`validators` 中。

*   接着我们输入 BPM，验证者的脉搏值，并创建一个单独的 Go 协程来处理这块儿逻辑

*   `delete(validators, address)` 如果验证者试图提议一个被污染（例如伪造）的 `block`，例如包含一个不是整数的 BPM，那么程序会抛出一个错误，我们会立即从我们的验证器列表 `validators` 中删除该验证者，他们将不再有资格参与到新块的铸造过程同时丢失相应的抵押令牌。

*   正是因为这种抵押令牌的机制，使得 PoS 协议是一种更加可靠的机制。如果一个人试图伪造和破坏，那么他将被抓住，并且失去所有抵押和未来的权益，因此对于恶意者来说，是非常大的威慑。

*   接着，我们用 `generateBlock` 函数创建一个新的 `block`，然后将其发送到 `candidateBlocks` 通道进行进一步处理。将`Block` 发送到通道使用的语法: `candidateBlocks <- newBlock`

*   最后会循环打印出最新的区块链，这样每个验证者都能获知最新的状态。

### 选择获取记账权的节点

下面是 PoS 的主要逻辑。我们需要编写代码以实现获胜验证者的选择;他们所持有的令牌数量越高，他们就越有可能被选为胜利者。

```go
// pickWinner creates a lottery pool of validators and chooses the validator who gets to forge a block to the blockchain
// by random selecting from the pool, weighted by amount of tokens staked
func pickWinner() {
    time.Sleep(30 * time.Second)
    mutex.Lock()
    temp := tempBlocks
    mutex.Unlock()

    lotteryPool := []string{}
    if len(temp) > 0 {

        // slightly modified traditional proof of stake algorithm
        // from all validators who submitted a block, weight them by the number of staked tokens
        // in traditional proof of stake, validators can participate without submitting a block to be forged
    OUTER:
        for _, block := range temp {
            // if already in lottery pool, skip
            for _, node := range lotteryPool {
                if block.Validator == node {
                    continue OUTER
                }
            }

            // lock list of validators to prevent data race
            mutex.Lock()
            setValidators := validators
            mutex.Unlock()

            // 获取验证者的 tokens
            k, ok := setValidators[block.Validator]
            if ok {
                // 向 lotteryPool 追加 k 条数据，k 代表的是当前验证者的 tokens
                for i := 0; i < k; i++ {
                    lotteryPool = append(lotteryPool, block.Validator)
                }
            }
        }

        // 通过随机获得获胜节点的地址
        s := rand.NewSource(time.Now().Unix())
        r := rand.New(s)
        lotteryWinner := lotteryPool[r.Intn(len(lotteryPool))]

        // 把获胜者的区块添加到整条区块链上，然后通知所有节点关于胜利者的消息
        for _, block := range temp {
            if block.Validator == lotteryWinner {
                mutex.Lock()
                Blockchain = append(Blockchain, block)
                mutex.Unlock()
                for _ = range validators {
                    announcements <- "\nwinning validator: " + lotteryWinner + "\n"
                }
                break
            }
        }
    }

    mutex.Lock()
    tempBlocks = []Block{}
    mutex.Unlock()
} 
```

*   每隔 30 秒，我们选出一个胜利者，这样对于每个验证者来说，都有时间提议新的区块，参与到竞争中来。接着创建一个`lotteryPool`，它会持有所有验证者的地址，这些验证者都有机会成为一个胜利者。然后，对于提议块的暂存区域，我们会通过`if len(temp) > 0`来判断是否已经有了被提议的区块。

*   在`OUTER FOR`循环中，要检查暂存区域是否和 `lotteryPool` 中存在同样的验证者，如果存在，则跳过。

*   在以 `k, ok := setValidators[block.Validator]`开始的代码块中，我们确保了从`temp`中取出来的验证者都是合法的，即这些验证者在验证者列表`validators`已存在。若合法，则把该验证者加入到`lotteryPool`中。

*   那么我们怎么根据这些验证者持有的令牌数来给予他们合适的随机权重呢？

    *   首先，用验证者的令牌填充`lotteryPool`数组，例如一个验证者有 100 个令牌，那么在`lotteryPool`中就将有 100 个元素填充；如果有 1 个令牌，那么将仅填充 1 个元素。

    *   然后，从`lotteryPool`中随机选择一个元素，元素所属的验证者即是胜利者，把胜利验证者的地址赋值给 lotteryWinner。这里能够看出来，如果验证者持有的令牌越多，那么他在数组中的元素也越多，他获胜的概率就越大；同时，持有令牌很少的验证者，也是有概率获胜的。

*   接着我们把获胜者的区块添加到整条区块链上，然后通知所有节点关于胜利者的消息：`announcements <- "\nwinning validator: " + lotteryWinner + "\n"`。

*   最后，清空 tempBlocks，以便下次提议的进行。

### 主函数

```go
func main() {
    err := godotenv.Load()
    if err != nil {
        log.Fatal(err)
    }

    // 创建初始区块
    t := time.Now()
    genesisBlock := Block{}
    genesisBlock = Block{0, t.String(), 0, calculateBlockHash(genesisBlock), "", ""}
    spew.Dump(genesisBlock)
    Blockchain = append(Blockchain, genesisBlock)

    httpPort := os.Getenv("PORT")

    // 启动 TCP 服务
    server, err := net.Listen("tcp", ":"+httpPort)
    if err != nil {
        log.Fatal(err)
    }
    log.Println("HTTP Server Listening on port :", httpPort)
    defer server.Close()

    // 启动了一个 Go routine 从 candidateBlocks 通道中获取提议的区块，然后填充到临时缓冲区 tempBlocks 中
    go func() {
        for candidate := range candidateBlocks {
            mutex.Lock()
            tempBlocks = append(tempBlocks, candidate)
            mutex.Unlock()
        }
    }()

    // 启动了一个 Go routine 完成 pickWinner 函数
    go func() {
        for {
            pickWinner()
        }
    }()

    // 接收验证者节点的连接
    for {
        conn, err := server.Accept()
        if err != nil {
            log.Fatal(err)
        }
        go handleConn(conn)
    }
} 
```

*   `godotenv.Load()` 会解析 `.env` 文件并将相应的 Key/Value 对都放到环境变量中，通过 `os.Getenv` 获取
*   然后创建一个创世区块 genesisBlock，形成了区块链。
*   接着启动了 Tcp 服务，等待所有验证者的连接。
*   启动了一个 Go 协程从 `candidateBlocks` 通道中获取提议的区块，然后填充到临时缓冲区 `tempBlocks` 中，最后启动了另外一个 Go 协程来完成 `pickWinner` 函数。
*   最后的 for 循环，用来接收验证者节点的连接。

## 运行

`go run main.go` 启动您的 Go 程序和 TCP 服务器，并会打印出初始区块的信息。

```go
$ go run main.go
(main.Block) {
 Index: (int) 0,
 Timestamp: (string) (len=50) "2018-05-08 16:45:27.14287 +0800 CST m=+0.000956793",
 BPM: (int) 0,
 Hash: (string) (len=64) "96a296d224f285c67bee93c30f8a309157f0daa35dc5b87e410b78630a09cfc7",
 PrevHash: (string) "",
 Validator: (string) ""
}
2018/05/08 16:45:27 HTTP Server Listening on port : 9000 
```

打开新的终端，运行 `nc localhost 9000`，
输入 `tokens` , 然后输入 `BPM`

![](img/8afca470c62edeef47b262dd1831fe69.jpg)

可以打开多个终端，输入不同的 `tokens` ,来检验 PoS 算法

![](img/3554b79995ce260af32ddec2ab386b82.jpg)