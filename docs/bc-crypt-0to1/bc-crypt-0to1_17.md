# 第十七章 RipeMD160 算法

## 1 RipeMD 算法简述

RIPEMD（RACE Integrity Primitives Evaluation Message Digest），中文译为“RACE 原始完整性校验消息摘要”，是比利时鲁汶大学 COSIC 研究小组开发的散列函数算法。RIPEMD 使用 MD4 的设计原理，并针对 MD4 的算法缺陷进行改进，1996 年首次发布 RIPEMD-128 版本，在性能上与较受欢迎的 SHA-1 相似。

RipeMD 算法是针对 MD4 和 MD5 算法缺陷分析提出的一种升级版本算法。这些算法主要是针对摘要值的长度进行了区分，如下表：

| 算法 | 摘要长度 | 备注 |
| --- | --- | --- |
| RipeMD128 | 128 | BouncyCastle 实现 |
| RipeMD160 | 160 | BouncyCastle 实现 |
| RipeMD256 | 256 | BouncyCastle 实现 |
| RipeMD320 | 320 | BouncyCastle 实现 |
| HmacRipeMD128 | 128 | BouncyCastle 实现 |
| HmacRipeMD160 | 160 | BouncyCastle 实现 |

所以构成 RIPEMD 家族的四个成员分别是：RIPEMD-128、RIPEMD-160、RIPEMD-256、RIPEMD-320。但安全性最高，使用最广泛的是 RIPEMD160 算法。

## 2 RipeMD160 算法简介

RIPEMD-160 是基于 Merkle-Damgard 构造的加密散列函数。哈希值的输出值一般是 16 进制的字符串。而 16 进制字符串，每两个字符占一个字节。我们知道，一个字节=8bit.所以使用 ripemd160 加密函数所得到的是一个 160bit 的值。

Merkle-Damgard 结构属于一个函数迭代运算的结构，抗 hash 碰撞能力强。

算法的原理几乎和 MD 算法的原理一样，但具体函数内容和过程不一样。RIPEMD-160 的核心是一个有 10 个循环的压缩函数模块，其中每个循环由 16 个处理步骤组成。在每个循环中使用不同的原始逻辑函数，算法的处理分为两种不同的情况，在这两种情况下，分别以相反的顺序使用 5 个原始逻辑函数。每一个循环都以当前分组的消息字和 160 位的缓存值 A、B、C、D、E 为输入得到新的值。每个循环使用一个额外的常数，在最后一个循环结束后，两种情况的计算结果 A、B、C、D、E 和 A′、B′、C′、D′、E′及链接变量的初始值经过一次相加运算产生最终的输出。对所有的 512 位的分组处理完成之后，最终产生的 160 位输出即为消息摘要。

在比特币系列的公链中经常用到该算法，尤其是账户地址的生成。

## 3 RipeMD160 算法过程

RipeMD160 算法过程的流程与 MD5 算法几乎一样，下面一步步进行分析。

### 3.1 填充

规则与 MD5 一模一样，使消息填充后的长度与 448 模 512 同与（即长度≡448 mod 512）。

### 3.2 填充消息长度

规则与 MD5 一模一样，将消息的长度填充到经过第一步填充之后的消息。最后得到的消息就是 512 的整数倍。

### 3.3 设置初始向量

比 MD5 多一个向量元素，但存储方式一样，采用小端法。
定义了 5 个 16 进制的大整数
A=67452301;
B=EFCDAB89;
C=98BADCFE;
D=10325476;
E=c3d2e1f0;

### 3.4 以 512 比特的分组（16 个字）为单位处理消息

和 MD5 处理分组消息的原理类似，实现过程不同。
先看一下流程图：
![](img/099bf765b8e3cb92292a99b23b7873ac.jpg)

图中 Y[q]为分组后的 512 比特的消息；f 是每个加密块用到的运算函数；Ki 是常量表 K 中的元素,与 MD5 的 T 表一样；Xi 是消息的第 q 个分组中第 i 个字（i=0，...，15）；+为模 2³² 加法。
具体 f 压缩函数为：
f1=x ^ y ^ z
f2=(x&y) | (~x&z)
f3=(x | ~y) ^ z
f4=(x&y) | (y&~z)
f5=x ^ (y | ~z)
按位异或运算符"^"；按位或运算符"|"；按位与运算符"&"；按位取反运算符"~"。

处理分组消息的过程分为 5 轮，每轮分为两组，分别是 Left 加密块和 Right 加密块运算，每个加密块运算又循环 16 步迭代运算。最后左右两侧经过 5 轮运算后得到的数据，和本组运算的初始向量按照指定组合顺序，将它们三组数据进行组合。组合之后的值就是本组运算结果，该结果就是下一组消息运算的初始向量。
Left 和 Right 运算的函数顺序是相反的，同时它们分别总共运行 80 次。

伪代码如下：

```go
h0:=67452301;
h1:=EFCDAB89;
h2:=98BADCFE;
h3:=10325476;
h4:=c3d2e1f0;

for(i := 0 to blocks - 1) {
    aLeft := h0
    bLeft := h1
    cLeft := h2
    dLeft := h3
    eLeft := h4

    aRight := h0
    bRight := h1
    cRight := h2
    dRight := h3
    eRight := h4
//每运行 16 步左右加密块要更换函数 f
for(int j := 0 to 79) {
        t0 := rotleft(s[j]) (aLeft + f(bLeft, cLeft, dLeft) + X[r[i]]) + eLeft
        aLeft := eLeft;
        eLeft := dLeft
        dLeft := rotleft(10) (c)
        cLeft := bLeft
        bLeft := t0

        t1 := rotRight(s[j]) (aRight + f(bRight, cRight, dRight) + X[r[i]]) + eRight
        aRight := eRight;
        eRight := dRight
        dRight := rotRight(10) (c)
        cRight := bRight
        bRight := t1    
    }

    t := h1 + cLeft + dRight
    h1 := h2 + dLeft + eRight
    h2 := h3 + eLeft + aRight
    h3 := h4 + aLeft + bRight
    h4 := h0 + bLeft + cRight
    h0 := t
} 
```

### 3.5 16 步迭代运算

与 MD5 几乎一样，只是 D 是 C 的循环移位得到的，而 MD5 它们的值一样。
单步操作示意图：
![](img/6213569e3c5bf6c7aae8f05d3dda15e0.jpg)
具体运算过程请查看 MD5 的运算。

## 4 go 语言封装的 RipeMD160 方法

封装路径 "golang.org/x/crypto/ripemd160"

### 4.1 pakage ripeMD160.go 代码

```go
import (
    "crypto"
    "hash"
)

func init() {
    crypto.RegisterHash(crypto.RIPEMD160, New)
}

// The size of the checksum in bytes.
const Size = 20

// The block size of the hash algorithm in bytes.
const BlockSize = 64

const (
    _s0 = 0x67452301
    _s1 = 0xefcdab89
    _s2 = 0x98badcfe
    _s3 = 0x10325476
    _s4 = 0xc3d2e1f0
)

// digest represents the partial evaluation of a checksum.
type digest struct {
    s  [5]uint32       // running context
    x  [BlockSize]byte // temporary buffer
    nx int             // index into x
    tc uint64          // total count of bytes processed
}

func (d *digest) Reset() {
    d.s[0], d.s[1], d.s[2], d.s[3], d.s[4] = _s0, _s1, _s2, _s3, _s4
    d.nx = 0
    d.tc = 0
}

// New returns a new hash.Hash computing the checksum.
func New() hash.Hash {
    result := new(digest)
    result.Reset()
    return result
}

func (d *digest) Size() int { return Size }

func (d *digest) BlockSize() int { return BlockSize }

func (d *digest) Write(p []byte) (nn int, err error) {
    nn = len(p)
    d.tc += uint64(nn)
    if d.nx > 0 {
        n := len(p)
        if n > BlockSize-d.nx {
            n = BlockSize - d.nx
        }
        for i := 0; i < n; i++ {
            d.x[d.nx+i] = p[i]
        }
        d.nx += n
        if d.nx == BlockSize {
            _Block(d, d.x[0:])
            d.nx = 0
        }
        p = p[n:]
    }
    n := _Block(d, p)
    p = p[n:]
    if len(p) > 0 {
        d.nx = copy(d.x[:], p)
    }
    return
}

func (d0 *digest) Sum(in []byte) []byte {
    // Make a copy of d0 so that caller can keep writing and summing.
    d := *d0

    // Padding.  Add a 1 bit and 0 bits until 56 bytes mod 64.
    tc := d.tc
    var tmp [64]byte
    tmp[0] = 0x80
    if tc%64 < 56 {
        d.Write(tmp[0 : 56-tc%64])
    } else {
        d.Write(tmp[0 : 64+56-tc%64])
    }

    // Length in bits.
    tc <<= 3
    for i := uint(0); i < 8; i++ {
        tmp[i] = byte(tc >> (8 * i))
    }
    d.Write(tmp[0:8])

    if d.nx != 0 {
        panic("d.nx != 0")
    }

    var digest [Size]byte
    for i, s := range d.s {
        digest[i*4] = byte(s)
        digest[i*4+1] = byte(s >> 8)
        digest[i*4+2] = byte(s >> 16)
        digest[i*4+3] = byte(s >> 24)
    }

    return append(in, digest[:]...)
} 
```

### 4.1 package ripemd160block

```go
import (
    "math/bits"
)

// work buffer indices and roll amounts for one line
var _n = [80]uint{
    0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15,
    7, 4, 13, 1, 10, 6, 15, 3, 12, 0, 9, 5, 2, 14, 11, 8,
    3, 10, 14, 4, 9, 15, 8, 1, 2, 7, 0, 6, 13, 11, 5, 12,
    1, 9, 11, 10, 0, 8, 12, 4, 13, 3, 7, 15, 14, 5, 6, 2,
    4, 0, 5, 9, 7, 12, 2, 10, 14, 1, 3, 8, 11, 6, 15, 13,
}

var _r = [80]uint{
    11, 14, 15, 12, 5, 8, 7, 9, 11, 13, 14, 15, 6, 7, 9, 8,
    7, 6, 8, 13, 11, 9, 7, 15, 7, 12, 15, 9, 11, 7, 13, 12,
    11, 13, 6, 7, 14, 9, 13, 15, 14, 8, 13, 6, 5, 12, 7, 5,
    11, 12, 14, 15, 14, 15, 9, 8, 9, 14, 5, 6, 8, 6, 5, 12,
    9, 15, 5, 11, 6, 8, 13, 12, 5, 12, 13, 14, 11, 8, 5, 6,
}

// same for the other parallel one
var n_ = [80]uint{
    5, 14, 7, 0, 9, 2, 11, 4, 13, 6, 15, 8, 1, 10, 3, 12,
    6, 11, 3, 7, 0, 13, 5, 10, 14, 15, 8, 12, 4, 9, 1, 2,
    15, 5, 1, 3, 7, 14, 6, 9, 11, 8, 12, 2, 10, 0, 4, 13,
    8, 6, 4, 1, 3, 11, 15, 0, 5, 12, 2, 13, 9, 7, 10, 14,
    12, 15, 10, 4, 1, 5, 8, 7, 6, 2, 13, 14, 0, 3, 9, 11,
}

var r_ = [80]uint{
    8, 9, 9, 11, 13, 15, 15, 5, 7, 7, 8, 11, 14, 14, 12, 6,
    9, 13, 15, 7, 12, 8, 9, 11, 7, 7, 12, 7, 6, 15, 13, 11,
    9, 7, 15, 11, 8, 6, 6, 14, 12, 13, 5, 14, 13, 13, 7, 5,
    15, 5, 8, 11, 14, 14, 6, 14, 6, 9, 12, 9, 12, 5, 15, 8,
    8, 5, 12, 9, 12, 5, 14, 6, 8, 13, 6, 5, 15, 13, 11, 11,
}

func _Block(md *digest, p []byte) int {
    n := 0
    var x [16]uint32
    var alpha, beta uint32
    for len(p) >= BlockSize {
        a, b, c, d, e := md.s[0], md.s[1], md.s[2], md.s[3], md.s[4]
        aa, bb, cc, dd, ee := a, b, c, d, e
        j := 0
        for i := 0; i < 16; i++ {
            x[i] = uint32(p[j]) | uint32(p[j+1])<<8 | uint32(p[j+2])<<16 | uint32(p[j+3])<<24
            j += 4
        }

        // round 1
        i := 0
        for i < 16 {
            alpha = a + (b ^ c ^ d) + x[_n[i]]
            s := int(_r[i])
            alpha = bits.RotateLeft32(alpha, s) + e
            beta = bits.RotateLeft32(c, 10)
            a, b, c, d, e = e, alpha, b, beta, d

            // parallel line
            alpha = aa + (bb ^ (cc | ^dd)) + x[n_[i]] + 0x50a28be6
            s = int(r_[i])
            alpha = bits.RotateLeft32(alpha, s) + ee
            beta = bits.RotateLeft32(cc, 10)
            aa, bb, cc, dd, ee = ee, alpha, bb, beta, dd

            i++
        }

        // round 2
        for i < 32 {
            alpha = a + (b&c | ^b&d) + x[_n[i]] + 0x5a827999
            s := int(_r[i])
            alpha = bits.RotateLeft32(alpha, s) + e
            beta = bits.RotateLeft32(c, 10)
            a, b, c, d, e = e, alpha, b, beta, d

            // parallel line
            alpha = aa + (bb&dd | cc&^dd) + x[n_[i]] + 0x5c4dd124
            s = int(r_[i])
            alpha = bits.RotateLeft32(alpha, s) + ee
            beta = bits.RotateLeft32(cc, 10)
            aa, bb, cc, dd, ee = ee, alpha, bb, beta, dd

            i++
        }

        // round 3
        for i < 48 {
            alpha = a + (b | ^c ^ d) + x[_n[i]] + 0x6ed9eba1
            s := int(_r[i])
            alpha = bits.RotateLeft32(alpha, s) + e
            beta = bits.RotateLeft32(c, 10)
            a, b, c, d, e = e, alpha, b, beta, d

            // parallel line
            alpha = aa + (bb | ^cc ^ dd) + x[n_[i]] + 0x6d703ef3
            s = int(r_[i])
            alpha = bits.RotateLeft32(alpha, s) + ee
            beta = bits.RotateLeft32(cc, 10)
            aa, bb, cc, dd, ee = ee, alpha, bb, beta, dd

            i++
        }

        // round 4
        for i < 64 {
            alpha = a + (b&d | c&^d) + x[_n[i]] + 0x8f1bbcdc
            s := int(_r[i])
            alpha = bits.RotateLeft32(alpha, s) + e
            beta = bits.RotateLeft32(c, 10)
            a, b, c, d, e = e, alpha, b, beta, d

            // parallel line
            alpha = aa + (bb&cc | ^bb&dd) + x[n_[i]] + 0x7a6d76e9
            s = int(r_[i])
            alpha = bits.RotateLeft32(alpha, s) + ee
            beta = bits.RotateLeft32(cc, 10)
            aa, bb, cc, dd, ee = ee, alpha, bb, beta, dd

            i++
        }

        // round 5
        for i < 80 {
            alpha = a + (b ^ (c | ^d)) + x[_n[i]] + 0xa953fd4e
            s := int(_r[i])
            alpha = bits.RotateLeft32(alpha, s) + e
            beta = bits.RotateLeft32(c, 10)
            a, b, c, d, e = e, alpha, b, beta, d

            // parallel line
            alpha = aa + (bb ^ cc ^ dd) + x[n_[i]]
            s = int(r_[i])
            alpha = bits.RotateLeft32(alpha, s) + ee
            beta = bits.RotateLeft32(cc, 10)
            aa, bb, cc, dd, ee = ee, alpha, bb, beta, dd

            i++
        }

        // combine results
        dd += c + md.s[1]
        md.s[1] = md.s[2] + d + ee
        md.s[2] = md.s[3] + e + aa
        md.s[3] = md.s[4] + a + bb
        md.s[4] = md.s[0] + b + cc
        md.s[0] = dd

        p = p[BlockSize:]
        n += BlockSize
    }
    return n
} 
```

## 5 go 实现 RipeMD160 算法应用

使用非常简单：

```go
package main

import (
    "fmt"
    "golang.org/x/crypto/ripemd160"
)

func main() {

    //创建 ripemd160 算法加密块
    hasher := ripemd160.New()

    //将明文写入加密块
    hasher.Write([]byte("helloword"))

    //通过加密块的方法对写入的明文进行加密，传参 nil，表示没有新的信息和加密后的 hash 进行组合
    hashBytes := hasher.Sum(nil)

    //字节转成字符串
    hashString := fmt.Sprintf("%x", hashBytes)
    fmt.Println(hashString)
}

//输出
486202e6b75a3b80034e2699b42ed7f4ceaf9a45 
```