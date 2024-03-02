# 第四章 PoW-挖矿及共识流程

本章节主要是熟悉比特币 pow 挖矿及共识流程，和简单实现一个 pow 挖矿 helloworld 的小 demo。

## 比特币 pow 挖矿及共识流程

比特币挖矿的过程如下：

1.  构建一个空区块，称为候选区块
2.  从内存池中打包交易至候选区块
3.  构造区块头，填写区块头的下述字段
    　　1）填写版本号 version 字段
    　　2）填写父区块哈希 prevhash 字段
    　　3）用 merkle 树汇总全部的交易，将 merkle root 的哈希值填写至 merkle root 字段
    　　4）填写时间戳 timestamp 字段
    　　5）填写目标值 Bits 字段
4.  开始挖矿。挖矿就是不断重复计算区块头的哈希值，修改 nonce 参数，直到找到一个满足条件的 nonce 值，也就是该 nonce 值下，hash 函数运算出来的 hash 值 < Bits。当挖矿节点成功求出一个解后把解填入区块头的 nonce 字段。
5.  这时一个新区块就成功挖出了，然后挖矿节点会做下面这些事：
    　　1) 按照标准清单检验新区块，检验通过后进行下面的 2)和 3)步骤
    　　2）立刻将这个新区块发给它的所有相邻节点，相邻节点收到这个新区块后进行验证，验证有效后会继续传播给所有相邻节点。
    　　3）将这个新区块连接到现有的区块链中，按照如下规则：
    　　　　根据新区块的 prevhash 字段在现有区块链中寻找这个父区块，
    　　　　(Ⅰ) 如果父区块是主区块链的“末梢”，则将新区块添加上去即可；
    　　　　(Ⅱ) 如果父区块所在的链是备用链，则节点将新区块添加到备用链，同时比较备用链与主链的工作量。如果备用链比主链积累了更多的工作量，节点将选择备用链作为其新的主链，而之前的主链则成为了备用链；
    　　　　(Ⅲ) 如果在现有的区块链中找不到它的父区块，那么这个区块被认为是“孤块”。孤块会被保存在孤块池中，直到它们的父区块被节点接收到。一旦收到了父区块并且将其连接到现有的区块链上，节点就会将孤块从孤块池中取出，并且连接到它的父区块，让它作为区块链的一部分。

pow 工作量证明共识机制流程图如下：

![](img/8da82d13aab135ceac0ad5882a6c7a4f.jpg)

演示挖坑过程，可链接下面网址：

[`anders.com/blockchain/blockchain.html`](https://anders.com/blockchain/blockchain.html)

## pow 简单的例子

下边这个代码很糙，只是简化版的 pow 挖矿过程，里面的 N 就是挖矿 Bits，我设置了一个常量。

```go
package main

import (
    "crypto/sha256"
    "fmt"
    "log"
    "bytes"
    "encoding/binary"
)
//声明了一个挖矿难度 Bits
const N  = 0x00FFffffffff

//定义一个 block 结构体
type block struct {
   //区块数据
    Data  []byte
    //随机数
    Nonce int
    //当前块的 hash
    hash []byte
    //挖矿目标难度值
    Bits int
}

func main() {
    //初始化一个空块
    var b = block{[]byte("helloworld"), 0,nil,N}

    nonce := 0
    for {

    //拼接 block 字段内容
        dataBytes := b.PreParae(nonce)
        //矿工计算函数，我只用了一次 256hash
        hash := sha256.Sum256(dataBytes)
        //将 hash 转换成 Uint64 类型，将与 N 进行比较大小
        hash1 := BytesToUint64(hash[:])
        //不断显现 hash 函数后的值
        fmt.Printf("\r%x", hash)
        //hash 值与目标值进行大小比较
        if hash1 < uint64(N) {
        //挖矿成功后，给 b 重新赋值，并跳出循环
            fmt.Println()
            b.Nonce=nonce
            b.hash=hash[:]
            break
        } else {
            nonce++
        }
    }
    fmt.Println(b)
}
//block 字段拼接
func (b block) PreParae(nonce int) []byte {
    data := bytes.Join(
        [][]byte{
            b.Data,
            IntToHex(int64(nonce)),
            b.hash,
        },
        []byte{},
    )
    return data
}
//字节转换成 64 进制
func BytesToUint64(array []byte) uint64 {
    var data uint64 = 0
    for i := 0; i < len(array); i++ {
        data = data + uint64(uint(array[i])<<uint(8*i))
    }

    return data
}
//将 64 进制数字转换成字节数组
func IntToHex(num int64) []byte {
    buff := new(bytes.Buffer)
    err := binary.Write(buff, binary.BigEndian, num)
    if err != nil {
        log.Panic(err)
    }

    return buff.Bytes()
} 
```