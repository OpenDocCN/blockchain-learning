# 第十六章 HMAC 算法及 golang 实现代码详解

## 1 HMAC 算法概述

HMAC 算法中文名称叫哈希消息认证码，英文全称是 Hash-based Message Authentication Code。它的算法是基于某个哈希散列函数（主要是 SHA 系列和 MD 系列），以一个密钥和一个消息为输入，生成一个消息摘要作为输出。HMAC 算法与其他哈希散列算法最大区别就是需要有密钥。它的算法函数是利用分组密码来建立的一个单向 Hash 函数。
下表显示具体的算法对应输出摘要的长度。

| 算法 | 摘要长度（位） | 备注 |
| --- | --- | --- |
| HmacMD5 | 128 | BouncyCastle 实现 |
| HmacSHA1 | 160（20 个字节） | BouncyCastle 实现 |
| HmacSHA256 | 256 | BouncyCastle 实现 |
| HmacSHA384 | 384 | BouncyCastle 实现 |
| HmacSHA512 | 512 | JAVA6 实现 |
| HmacMD2 | 128 | BouncyCastle 实现 |
| HmacMD4 | 128 | BouncyCastle 实现 |
| HmacSHA224 | 224 | BouncyCastle 实现 |

HMAC 的密钥可以是任何长度，如果密钥的长度超过了摘要算法信息分组的长度，则首先使用摘要算法计算密钥的摘要作为新的密钥。一般不建议使用太短的密钥，因为密钥的长度与安全强度是相关的。通常选取密钥长度不小于所选用摘要算法输出的信息摘要的长度。

## 2 HMAC 算法分析

HMAC 的算法原理比较简单，算法公式是: HMAC(K,M)=H(K ^ opad | [H(K ^ ipad∣M)])
其中
K，M 为输入参数密钥和消息；
H 代表所采用的 HASH 算法(如 SHA-256);
opad 用 16 进制字 0x5c 重复 B 次;
ipad 用 016 进制字 x36 重复 B 次;
按位异或运算符"^"；按位或运算符"|"；

分析公式运算过程：
1.将密钥通过 K ^ opad 进行处理得到 Kopad；
2.处理消息：K ^ ipad 得到的 Kipad 与消息 M 组合，得到 MK，对 MK 进行 Hash 运算得到 MH
3.拼接处理后的 Kopad 和 MH，得到 MM
4.对 MM 进行 Hash 运算，得到的值就是 HMAC 的散列值。

## 3 HMAC 算法运算过程

首先先看一下 HMAC 运算过程的流程图，然后再一步步分析它。
![](img/893bef1c7f0bae7a2bec5c1b756060a9.jpg)

（1）检查密钥 K 的长度。如果 K 的长度大于 B，则先使用该 HMAC 算法内的散列算法，算出一个长度为 L 的新密钥。如果后 K 的长度小于 B，则在其后面追加 0 来使其长度达到 B。

（2）将上一步生成的 B 字长的密钥字符串与 ipad 做异或运算，得到 Kipad。

（3）将需要处理的数据流 M 填充至第二步的结果字符串中。也就是把 M append 到 Kipad 里。

（4）使用该 HMAC 算法内的散列算法 H 计算上一步中生成的数据流的信息摘要值。

（5）将第一步生成的 B 字长密钥字符串与 opad 做异或运算。

（6）再将第四步得到的结果填充到第五步的结果之后。

（7）使用哈希函数 H 计算上一步中生成的数据流的信息摘要值，输出结果就是最终的 HMAC 值。

上边提到的 B 代表 H 中所处理的块大小，这个大小是处理块大小，而不是输出 hash 的大小
如，SHA-1 和 SHA-256 B = 64
SHA-384 和 SHA-512 B = 128

由上述描述过程，我们知道 HMAC 算法的计算过程实际是对原文做了两次类似于添加剂处理的哈希过程。

## 4 HMAC 算法 golang 封装的代码详细解析

以下对封装好的 HMAC 的代码进行了详细解析：

```go
package main

import (
    "hash"
    "crypto/md5"
    "encoding/hex"
    "fmt"
)
// 原理公式 hmac = H([key ^ opad] H([key ^ ipad] text))

//定义一个 hmac 的结构体
type hmac struct {
    size         int //散列函数 MD5 的输出数据字节数
    B    int // 加密块的长度
    opad, ipad   []byte //分别重复 16 进制 0x36 和 0x5c，使其字节长度等于 blocksize
    outer, inner hash.Hash //hmac 外部和内部压缩函数，类型是实现 MD5 的接口
}

//处理消息的组合方法
func (h *hmac) Sum(in []byte) []byte {
    origLen := len(in)

    //实现 Sum 这个接口的方法如下：
    /*func (d0 *digest) Sum(in []byte) []byte {
    d := *d0
    hash := d.checkSum()
    return append(in, hash[:]...)
}*/
    //通过实现的方法，我们可以看出 in 一般传参为 nil；d0 就是要加密的信息；
    //要是 in 不为空，则代表需要将 in 后边填充已经运算过一段信息的 hash 值。
    //in 就是消息第一次被 hash 处理的数据
    in = h.inner.Sum(in)

    //输出压缩函数保证初始 hash 为空
    h.outer.Reset()
    //将 key 与 0x5c 异或后的数据，写入
    h.outer.Write(h.opad)
    //再写入 in
    h.outer.Write(in[origLen:])

    //最后得到组合后的数据，进行一次 MD5 运算就得到 HMAC 的 hash 值。
    return h.outer.Sum(in[:origLen])
}

//写入的方法
func (h *hmac) Write(p []byte) (n int, err error) {
    return h.inner.Write(p)
}

//方法返回散列函数输出数据字节数
func (h *hmac) Size() int { return h.size }

//方法返回散列函数的分割数据块字长
func (h *hmac) BlockSize() int { return h.B }//方法返回 blocksize

//
func (h *hmac) Reset() {
    h.inner.Reset()
    h.inner.Write(h.ipad)
}

//创建一个 hmac 对象去实现 hash.Hash 这个接口
func New(h func() hash.Hash, key []byte) hash.Hash {
    hm := new(hmac)
    hm.outer = h() //输出压缩函数
    hm.inner = h() //输入压缩函数,函数是一个，都是 MD5

    hm.size = hm.inner.Size()//散列函数 MD5 的输出数据字节数=16
    hm.B = hm.inner.BlockSize()//散列函数的分割数据块字长=64

    //创建两个变量，用于去接收 key 变化之后的结果
    hm.ipad = make([]byte, hm.B)
    hm.opad = make([]byte, hm.B)
    if len(key) > hm.B {
        // 如果密码长度太大，对它进行 hash 运算。
        hm.outer.Write(key)
        key = hm.outer.Sum(nil)
    }
    //将 key 覆盖到创建的两个变量中
    copy(hm.ipad, key)
    copy(hm.opad, key)

    //copy key 后的两个变量，分别将他的每一个元素与 0x36 和 0x36 异或运算
    for i := range hm.ipad {
        hm.ipad[i] ^= 0x36

    }
    for i := range hm.opad {
        hm.opad[i] ^= 0x36
    }

    //将异或后的值写入 hm 里。
    hm.inner.Write(hm.ipad)
    hm.outer.Write(hm.opad)

    return hm
}

func main()  {

    //创建运算对象，HMAC 需要两个参数：key 和 hash 函数
    hmac := New(md5.New, []byte("23456uikjdfgh"))

    //将明文写入到 hmac 中
    hmac.Write([]byte("helloworld"))

    //hmac 对象对写入数据的运算
    hashBytes:=hmac.Sum(nil)

    //字节转换成字符串
    hash:= hex.EncodeToString(hashBytes)

    fmt.Println(hash)

} 
```

## 5 用 golang 使用 HMAC 算法

使用非常简单，几行代码，不做解释：

```go
package main

import (
    "crypto/hmac"
    "crypto/md5"
    "crypto/sha1"
    "encoding/hex"
    "fmt"
)

func Md5(data string) string {
    md5 := md5.New()
    md5.Write([]byte(data))
    md5Data := md5.Sum([]byte(""))
    return hex.EncodeToString(md5Data)
}

func Hmac(key, data string) string {
    hmac := hmac.New(md5.New, []byte(key))
    hmac.Write([]byte(data))
    return hex.EncodeToString(hmac.Sum(nil)
}

func Sha1(data string) string {
    sha1 := sha1.New()
    sha1.Write([]byte(data))
    return hex.EncodeToString(sha1.Sum(nil))
}

func main() {
    fmt.Println(Md5("hello"))
    fmt.Println(Hmac("key2", "hello"))
    fmt.Println(Sha1("hello"))
} 
```