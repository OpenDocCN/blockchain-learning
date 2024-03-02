# 第六章 POS

通过本章节，大家一起学习单纯的 pos 是什么机制，以及如何工作的。它较 pow 有什么优缺点。

## POS 概述

POS 即权益证明，英文全称 Proof of Stake。它是由匿名极客`Sunny King`发明的。`Sunny King`很神秘，类似中本聪，他的具体信息很少有人知道。同时他也是点点币（PPCoin）和质数币（PrimeCoin）的创始人。2012 年点点币问世，成为全球首次使用 POS 共识机制的数字货币。

基于 POW 的公链挖矿纯靠算力，挖矿难度越高，算力也会越来越中心化。目前比特币挖矿的算力超过 70%在中国，据`Blockchain.info`2018 年的统计数据显示，目前世界前十大矿池中，中国独占 8 家，而中国 70%的算力在四川。这样完全背离了去中心化的思想，只有算力高的机构或矿场才能挖到区块，而算力低的个人几乎没有可能挖到区块。算力低的矿工要是参与挖矿，那得赔死，投入电费、场地、矿机等等，结果几乎甚至没有收益。然后他们大部分会放弃挖矿或加入其他大型矿池。而且挖矿代币价格的较大波动也会影响挖矿收益。所以 POW 类型的挖矿具有很大的投资风险。POW 挖矿比拼的就是算力，随着挖矿难度的提高，算力会要求越来越高，导致参与挖矿的矿机数量越来越多，配置也越来越高，如此会造成多大的资源浪费啊。

针对 POW 的问题，POS 指出了一个全新的概念-币龄，币龄 = 持有的币数 * 持有币的天数，例如钱包里有 90 个点点币，持有了 10 天，则币龄=900。决定下一个区块是由谁出，是看当前时刻谁的币龄大。出块后币龄归零，重新计时持币时间。币龄是对应账户恒定数量持币数的值，如果账户的持币数量发生变化，那么币龄也会归零，重新计时。通过币龄来决定出块，不再需要比拼算力，多么环保啊，节省了多少资源啊。

而且为了让持币人把币握住，不乱抛售，POS 还使用了“利息”机制。用户的持币钱包或客户端 24 小时处于工作状态或后台运行状态，其实就是 pos 的挖矿状态，系统会在固定时间参照币龄给予持币人一定量的利息（一定量的币）。如此币的抛售将不会像传统的币那么频繁，币价相应的也会稳定。

POS 出块时间可以设为恒定值，如 10 分钟一个块，那么每 10 分钟，会触发记账功能。

为了让 POS 出块人带有随机性，需要借助一些随机函数，币龄只是增加成为出块者可能性的筹码。如，A 的币龄 100，B 的币龄 10，那么 A 能够出块的概率要比 B 大 10 倍。

使用 POS 较 POW 更不容易遭受 51%攻击，相比起掌握系统一半以上的算力，拥有整个系统 51%的财力会更加困难。

POS 解决或者缓解了 POW 的一些问题，但是它也有它的很大缺点：

1.更容易被垄断：因为持币越多，持有的越久，币龄就越高，越容易挖到区块并得到激励，持币少的人基本上没有机会，这样整个系统的安全性实际上会被持币数量较大的一部分人（大股东们）掌握；而 POW 理论上则不存在这个问题，因为理论上任何人都可以购买矿机获得提高自己的算力（甚至可以联合起来），提升自己挖矿成功的概率；

2.很难应对分叉的情况：当出现分叉时，PoS 可以在两条链上同时挖矿并获得收益。而 PoW 则不存在这个问题，因为当出现分叉以后，PoW 总是选择工作量大的链做为主链。

但是在实际应用中，纯 POS 的共识机制是不可行的，通常会和 POW 混合一起用或者通过 POS 升级改进 POW。这样会更好的发挥各自的优点，减小双方缺点带来的影响。

根据 masternodes.online 的数据显示，目前基于 POS 算法的币种数量已有 330，但是市值在 2000 万美金以上的只有 9 种，分别是 DASH、PIVX、SYS、BLOCK、XZC、SMART、XSN、PAC、DEV。其中，市值最高的为 DASH，目前为 21.57 亿美元，POS 年化收益约为 7.19%；XZC 市值 9805 万美元，POS 年化收益约为 27.28%。

## Go 实现一个纯 POS 机制的简单挖矿

设计思路：
1.有两个挖矿节点，分别持有的币为 10 个和 5 个，持币时间为 1 。所以他们的币龄也就是 10 和 5 。
2.创建一个长度为 15 的数组，分别将两个节点的账户地址按照比例布满这个数组。
3.选择一个从 0 到 14 的随机数，将该随机数作为 2 步数组的下表，找到该下表下的挖矿地址。
4.挖矿地址确认后，通过方法产生区块。
5.将区块上链，这里的区块链我们定义了一个切片。上链过程就是 append 过程。

```go
package main

import (
    //"sync"
    "time"
    "crypto/sha256"
    "encoding/hex"
    "fmt"
    "math/rand"
)

// 声明一个块结构体
type Block struct {
    Index     int  //区块高度
    Timestamp string //时间戳，也就是当前时间转换成字符串
    BPM       int   //要保存的上链数据
    Hash      string  //当前区块 hash 值
    PrevHash  string  //父区块 hash 值
    Validator string  //当前区块出块人的账户地址
}

//声明一个区块链，类型是切片。也就是将区块放在这个切片里面
var Blockchain []Block

//出块函数，也就是当随机抽中哪个账户后，该账户就会作为该区块的参数，生成一个最新区块
func generateBlock(oldBlock Block, BPM int, address string) (Block, error) {

    var newBlock Block

    t := time.Now()

    //生成区块过程就是给区块的字段赋值
    newBlock.Index = oldBlock.Index + 1
    newBlock.Timestamp = t.String()
    newBlock.BPM = BPM
    newBlock.PrevHash = oldBlock.Hash
    newBlock.Hash = calculateBlockHash(newBlock)
    newBlock.Validator = address

    //返回最新区块
    return newBlock, nil
}

// SHA256 算法计算当前区块的 hash 值
func calculateHash(s string) string {
    h := sha256.New()
    h.Write([]byte(s))
    hashed := h.Sum(nil)
    return hex.EncodeToString(hashed)
}

//将区块字段拼接，为生成 hash 函数做准备
func calculateBlockHash(block Block) string {
    record := string(block.Index) + block.Timestamp + string(block.BPM) + block.PrevHash
    return calculateHash(record)
}

//持币人信息
type Node struct {
    tokens int
    address string

}

//现在参与挖矿的币总数为 15 个，我们认为现在是平等机会的 15 个持币 1 个的矿工。
//声明一个数组保存 15 个账户地址
//两个节点参与挖矿
var N[15] string
var p [2] Node

//进入主函数
func main() {

    //给两个挖矿节点赋值
    p[0]=Node{10,"abc"}
    p[1]=Node{5,"bcd"}

    //通过下边的 for 循环，按照两个节点他们的持币数，分别将他们的地址赋值到之前定义好的账户地址数组中。
    //可以看出，前 10 个账户都是 p[0]的；后边 5 个是 p[1]的
    var cnt = 0
    for i:=0;i<2;i++ {
        for j:=0;j<p[i].tokens;j++{

            N[cnt] = p[i].address
            cnt++
        }
    }

    //设置一个随机数种子
    rand.Seed(time.Now().Unix())
    var firstBlock Block

    //根据随机数种子产生的随机数，找到矿工的地址，然后出块
    var b,_ = generateBlock(firstBlock,10,N[rand.Intn(cnt)])

    //将新的区块放到区块链上。
    Blockchain = append(Blockchain,b)

    fmt.Println(Blockchain)
} 
```