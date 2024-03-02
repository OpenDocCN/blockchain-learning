# 第五章 用 go 语言实现一个 pow 共识工程

## 1.1 项目代码结构

![](img/a32cbea4f680f83c9d969875d23ad035.jpg)

## 1.2 项目运行结果

![](img/8dcf7a75b1e640d0e03a3aff3d8c8dcf.jpg)

## 1.3 完整代码

Block.go:

```go
package BLC

import (
    "time"
    "fmt"
)

type Block struct {
    //1\. 区块高度
    Height int64
    //2\. 上一个区块 HASH
    PrevBlockHash []byte
    //3\. 交易数据
    Data []byte
    //4\. 时间戳
    Timestamp int64
    //5\. Hash
    Hash []byte
    // 6\. Nonce
    Nonce int64
}

//1\. 创建新的区块
func NewBlock(data string,height int64,prevBlockHash []byte) *Block {

    //创建区块
    block := &Block{height,prevBlockHash,[]byte(data),time.Now().Unix(),nil,0}

    // 调用工作量证明的方法并且返回有效的 Hash 和 Nonce
    pow := NewProofOfWork(block)

    // 挖矿验证
    hash,nonce := pow.Run()

    block.Hash = hash[:]
    block.Nonce = nonce

    fmt.Println("")

    return block

}

//2\. 单独写一个方法，生成创世区块

func CreateGenesisBlock(data string) *Block {

    return NewBlock(data,1, []byte{0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0})
} 
```

BlockChain.go

```go
package BLC

type Blockchain struct {
    Blocks []*Block  // 存储有序的区块
}

// 增加区块到区块链里面
func (blc *Blockchain) AddBlockToBlockchain(data string,height int64,preHash []byte)  {
    // 创建新区块
    newBlock := NewBlock(data,height,preHash)
    // 往链里面添加区块
    blc.Blocks = append(blc.Blocks,newBlock)
}

//1\. 创建带有创世区块的区块链
func CreateBlockchainWithGenesisBlock() *Blockchain {
    // 创建创世区块
    genesisBlock := CreateGenesisBlock("Genesis Data.......")
    // 返回区块链对象
    return &Blockchain{[]*Block{genesisBlock}}
} 
```

ProofOfWork.go

```go
package BLC

import (
    "math/big"
    "bytes"
    "crypto/sha256"
    "fmt"
)

//0000 0000 0000 0000 1001 0001 0000 .... 0001

// 256 位 Hash 里面前面至少要有 16 个零
const targetBit  = 16

// hash nil
//256 位
// 32
// 8
// 二进制表示 0000 0000 0000 0000 0000 0000 0000 0000
// 2 的 32 - 8 次方

type ProofOfWork struct {
    Block *Block // 当前要验证的区块
    // 0000 0001 0000 0000 0000 0000 0000 0000
    target *big.Int // 大数据存储 2²⁴
}

// 前面必须有 8 个零
// 0000 0000 1111 1111 1111 1111 1111 1111

// 0000 0001
// 0001 0000
// 0000 1111

// 0001 0000
// 0000 1111

// 0000 0000 0000 0000 1111 111

// 数据拼接，返回字节数组
func (pow *ProofOfWork) prepareData(nonce int) []byte {
    data := bytes.Join(
        [][]byte{
            pow.Block.PrevBlockHash,
            pow.Block.Data,
            IntToHex(pow.Block.Timestamp),
            IntToHex(int64(targetBit)),
            IntToHex(int64(nonce)),
            IntToHex(int64(pow.Block.Height)),
        },
        []byte{},
    )

    return data
}

func (proofOfWork *ProofOfWork) IsValid() bool {

    //1.proofOfWork.Block.Hash
    //2.proofOfWork.Target

    var hashInt big.Int
    // []byte 转 Int
    hashInt.SetBytes(proofOfWork.Block.Hash)

    // Cmp compares x and y and returns:
    //
    //   -1 if x <  y
    //    0 if x == y
    //   +1 if x >  y
    if proofOfWork.target.Cmp(&hashInt) == 1 {
        return true
    }

    return false
}

func (proofOfWork *ProofOfWork) Run() ([]byte,int64) {

    //1\. 将 Block 的属性拼接成字节数组

    //2\. 生成 hash

    //3\. 判断 hash 有效性，如果满足条件，跳出循环

    nonce := 0

    var hashInt big.Int // 存储我们新生成的 hash
    var hash [32]byte

    for {
        //准备数据
        dataBytes := proofOfWork.prepareData(nonce)

        // 生成 hash
        hash = sha256.Sum256(dataBytes)
        fmt.Printf("\r%x",hash)

        // 将 hash 存储到 hashInt
        hashInt.SetBytes(hash[:])

        //判断 hashInt 是否小于 Block 里面的 target
        // Cmp compares x and y and returns:
        //
        //   -1 if x <  y
        //    0 if x == y
        //   +1 if x >  y
        if proofOfWork.target.Cmp(&hashInt) == 1 {
            break
        }

        nonce = nonce + 1
    }

    return hash[:],int64(nonce)
}

// 创建新的工作量证明对象
func NewProofOfWork(block *Block) *ProofOfWork  {

    //1.big.Int 对象 1
    // 2
    //0000 0001
    // 8 - 2 = 6
    // 0100 0000  64
    // 0010 0000
    // 0000 0000 0000 0001 0000 0000 0000 0000 0000 0000 .... 0000

    //1\. 创建一个初始值为 1 的 target

    // int Int
    target := big.NewInt(1) // 1

    //2\. 左移 256 - targetBit

    target = target.Lsh(target,256 - targetBit)

    return &ProofOfWork{block,target}
} 
```

utils.go:

```go
package BLC

import (
    "bytes"
    "encoding/binary"
    "log"
)

// 将 int64 转换为字节数组
func IntToHex(num int64) []byte {
    buff := new(bytes.Buffer)
    err := binary.Write(buff, binary.BigEndian, num)
    if err != nil {
        log.Panic(err)
    }

    return buff.Bytes()
} 
```

主函数 main.go

```go
package main

import (
    "kongyixueyuan.com/blockchain_go_videos-master/part8-proof-of-work/BLC"
    "fmt"
)

func main()  {

    // 创世区块
    blockchain := BLC.CreateBlockchainWithGenesisBlock()

    // 新区块
    blockchain.AddBlockToBlockchain("Send 100RMB To tom",blockchain.Blocks[len(blockchain.Blocks) - 1].Height + 1,blockchain.Blocks[len(blockchain.Blocks) - 1].Hash)

    blockchain.AddBlockToBlockchain("Send 200RMB To lily",blockchain.Blocks[len(blockchain.Blocks) - 1].Height + 1,blockchain.Blocks[len(blockchain.Blocks) - 1].Hash)

    blockchain.AddBlockToBlockchain("Send 300RMB To hanmeimei",blockchain.Blocks[len(blockchain.Blocks) - 1].Height + 1,blockchain.Blocks[len(blockchain.Blocks) - 1].Hash)

    blockchain.AddBlockToBlockchain("Send 50RMB To lucy",blockchain.Blocks[len(blockchain.Blocks) - 1].Height + 1,blockchain.Blocks[len(blockchain.Blocks) - 1].Hash)

    fmt.Println(blockchain)
    fmt.Println(blockchain.Blocks)
} 
```