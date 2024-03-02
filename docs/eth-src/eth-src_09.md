# 第九章 交易签名

在前面章节介绍交易数据结构时，曾提到过 R、S、V 三个字段是根据用户私钥对交易数据签名所得，那么具体是签名流程是怎么样的呢？

整个交易数据签名过程分为五步:

1.  创建交易
    创建一条 Transcation 数据，主要包含九个字段: AccountNonce, Price, GasLimit, Recipient, Amount, Payload, chainId, 0, 0
2.  序列化数据
    根据 RLP 协议将上述创建的交易进行序列化处理
3.  哈希计算
    使用 Keccak-256 算法计算序列化数据哈希值
4.  私钥签名
    使用帐户私钥对上一步哈希数据进行数字签名
5.  字节拆分
    对签名结果进行字节拆分，得出 R、S 、 V 并同步交易实例参数

Signer 作为一个接口，有三种扩展实现: FrontierSigner、HomesteadSigner、EIP155Signer. 除了 FrontierSigner 外，其他几个实现都是在对前一版本进行扩展开发。

> 注: 在后续的代码分析，经常可以看到 Frontier、Homestead、EIP、Constantinople 等关键词。其中， Frontier、Homestead、Constantinople 是以太坊软件的大版本代号，而 EIP 是对 Homestead 版本上的一些 issue 问题更新，EIP 后面跟的数字便是对应 issue 的具体编号。

## 补充知识点

**1\. 交易数据结构中是不存储发送人地址的，那么如何获取地址呢？**
可以通过交易数据中的 R、S、V 签名数据中推算得出。

```go
func (fs FrontierSigner) Sender(tx *Transaction) (common.Address, error) {
    return recoverPlain(fs.Hash(tx), tx.data.R, tx.data.S, tx.data.V, false)
}
//sourcepath: core/types/transaction_signing.go 
```

**2\. 通过交易签名可以查出 chain id 吗**
可以。可从 V 签名数据中推导而出。

```go
func deriveChainId(v *big.Int) *big.Int {
    if v.BitLen() <= 64 {
        v := v.Uint64()
        if v == 27 || v == 28 {
            return new(big.Int)
        }
        return new(big.Int).SetUint64((v - 35) / 2)
    }
    v = new(big.Int).Sub(v, big.NewInt(35))
    return v.Div(v, big.NewInt(2))
}
//sourcepath: core/types/transaction_signing.go 
```