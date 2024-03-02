# 第十二章 椭圆曲线密码学

## 1\. 椭圆曲线密码学简介

椭圆曲线密码学（英语：Elliptic curve cryptography，缩写为 ECC），一种建立公开密钥加密的算法，基于椭圆曲线数学。椭圆曲线在密码学中的使用是在 1985 年由 Neal Koblitz 和 Victor Miller 分别独立提出的。
ECC 的主要优势是在某些情况下它比其他的方法使用更小的密钥——比如 RSA 加密算法——提供相当的或更高等级的安全。
椭圆曲线密码学的许多形式有稍微的不同，所有的都依赖于被广泛承认的解决椭圆曲线离散对数问题的困难性上.
不管是 RSA 还是 ECC 或者其它，公钥加密算法都是依赖于某个正向计算很简单（多项式时间复杂度），而逆向计算很难（指数级时间复杂度）的数学问题。
椭圆曲线依赖的数学难题是:
k 为正整数，G 是椭圆曲线上的点（称为基点）, k*G=Q , 已知 G 和 Q，很难计算出 k

## 2\. 椭圆曲线

一般，椭圆曲线可以用如下二元三阶方程表示：
　　y² = x³ + ax + b，其中 a、b 为系数。
参数 a=0;b=7,得到 y² = x³ +7，这个方程式产生的曲线就是 secp256k1 曲线。
曲线形状：![](img/ff11826ebe923c769f13aea6162f377b.jpg)

## 3\. 椭圆曲线在密码学的应用

椭圆曲线是连续的，并不适合用于加密；所以，我们必须把椭圆曲线变成离散的点，我们要把椭圆曲线定义在有限域上。
我们给出一个有限域 Fp，Fp 中有 p（p 为质数）个元素 0,1,2,…, p-2,p-1
Fp 的加法是 a+b≡c(mod p)
Fp 的乘法是 a×b≡c(mod p)
Fp 的除法是 a÷b≡c(mod p)，即 a×b^(-1)≡c (mod p)，b-1 也是一个 0 到 p-1 之间的整数，但满足 b×b-1≡1 (mod p)

考虑 K=kG ，其中 K、G 为椭圆曲线 Ep(a,b)上的点，n 为 G 的阶（nG=O∞ ），k 为小于 n 的整数。则给定 k 和 G，根据加法法则，计算 K 很容易但反过来，给定 K 和 G，求 k 就非常困难。因为实际使用中的 ECC 原则上把 p 取得相当大，n 也相当大，要把 n 个解点逐一算出来列成上表是不可能的。这就是椭圆曲线加密算法的数学依据。 点 G 称为基点（base point）k（k 小于 n）为私有密钥（privte key），K 为公开密钥（public key)

## 4\. 椭圆曲线运算

• 加法
• 过曲线上的两点 A、B 画一条直线，找到直线与椭圆曲线的交点，交点关于 x 轴对称位置的点，定义为 A+B，即为加法。如下图所示：
![](img/361db4e3bb469efdf5f6b6f967591327.jpg)
• 二倍运算
• 上述方法无法解释 A + A，即两点重合的情况。因此在这种情况下，将椭圆曲线在 A 点的切线，与椭圆曲线的交点，交点关于 x 轴对称位置的点，定义为 A + A，即 2A，即为二倍运算。
![](img/be8200fd14f6ad41478821af0f9c0b56.jpg)
• 正负取反
• 将 A 关于 x 轴对称位置的点定义为-A，即椭圆曲线的正负取反运算。如下图所示：
![](img/e2cc5b14bffe3d5435c27b0004463a2f.jpg)
• 无穷远点
• 如果将 A 与-A 相加，过 A 与-A 的直线平行于 y 轴，可以认为直线与椭圆曲线相交于无穷远点。
综上，定义了 A+B、2A 运算，因此给定椭圆曲线的某一点 G，可以求出 2G、3G（即 G + 2G）、4G......。即：当给定 G 点时，已知 x，求 xG 点并不困难。反之，已知 xG 点，求 x 则非常困难。此即为椭圆曲线加密算法背后的数学原理。

## 5\. 有限域上的椭圆曲线运算

• 椭圆曲线要形成一条光滑的曲线，要求 x,y 取值均为实数，即实数域上的椭圆曲线。但椭圆曲线加密算法，并非使用实数域，而是使用有限域。按数论定义，有限域 GF(p)指给定某个质数 p，由 0、1、2......p-1 共 p 个元素组成的整数集合中定义的加减乘除运算。
• 假设椭圆曲线为 y² = x³ + x + 1，其在有限域 GF(23)上时，写作：y² ≡ x³ + x + 1 (mod 23)
• 此时，椭圆曲线不再是一条光滑曲线，而是一些不连续的点。以点(1,7)为例，7² ≡ 1³ + 1 + 1 ≡ 3 (mod 23)。如此还有如下点：
• (0,1) (0,22)(1,7) (1,16)(3,10) (3,13)(4,0)(5,4) (5,19)(6,4) (6,19)(7,11) (7,12)(9,7) (9,16)(11,3) (11,20)等等。
• 另外，如果 P(x,y)为椭圆曲线上的点，则-P 即(x,-y)也为椭圆曲线上的点。如点 P(0,1)，-P=(0,-1)=(0,22)也为椭圆曲线上的点。

## 6\. 椭圆曲线加密算法原理

设私钥、公钥分别为 k、K，即 K = kG，其中 G 为 G 点。

公钥加密：

选择随机数 r，将消息 M 生成密文 C，该密文是一个点对，即：

C = {rG, M+rK}，其中 K 为公钥

私钥解密：

M + rK - k(rG) = M + r(kG) - k(rG) = M

其中 k、K 分别为私钥、公钥。

## 7\. 椭圆曲线签名算法原理

• 椭圆曲线签名算法，即 ECDSA。设私钥、公钥分别为 k、K，即 K = kG，其中 G 为 G 点。
• 私钥签名：
• 1、选择随机数 R，计算点 RG(x, y)。
• 2、根据随机数 R、消息 M 的哈希 h、私钥 k，计算出两个*big.int 类型的数 r，s。
• 3、将 r，s 转换成字节切片，并拼接一起，形成签名
• 4、将消息 M、和签名发给接收方。
公钥验证签名：
1、接收方收到消息 M、以及签名。
2、将签名提取出 r，s
3、根据消息求哈希 h。
4、通过 r，s 产生的一个点，如果这个点在椭圆曲线上，即验签成功。

## 8\. 椭圆曲线 Ecc 加密解密代码实现

```go
•    package main

import (
   "/crypto/ecies"
   "crypto/elliptic"
   "crypto/rand"
   "fmt"
   "encoding/hex"
)

func main() {
   msg:="hello world"
//调用以太坊的曲线加密包下的方法产生私钥 prv,_:=ecies.GenerateKey(rand.Reader,elliptic.P256(),nil )
//私钥产生公钥
   pub:=prv.PublicKey

//调用加密方法对明文进行加密
ct,_:=ecies.Encrypt(rand.Reader,&pub,[]byte(msg),nil,nil)
   scrt:=hex.EncodeToString(ct)
   fmt.Println(scrt)
//对密文进行解密
   ms,_:=prv.Decrypt(ct,nil,nil)
   fmt.Println(string(ms))

} 
```

## 9\. 代码实现(ECDSA)椭圆曲线的签名和验签

```go
package main

import (
   "fmt"
   "crypto/ecdsa"
   "crypto/elliptic"
   "crypto/rand"
   "crypto/sha256"
   "math/big"
)

func main() {
   //明文
   message := []byte("Hello world")

   //获取私钥
   key, err := NewSigningKey()
   if err != nil {
      return
   }

   //用私钥对明文进行签名
   signature, err := Sign(message, key)

   fmt.Printf("签名后：%x\n", signature)
   if err != nil {
      return
   }

   //用公钥对签名进行验证，确认签名是否是对当前明文的有效
   if !Verify(message, signature, &key.PublicKey) {
      fmt.Println("验证失败！")
      return
   }else{
      fmt.Println("验证成功！")
   }
}

func NewSigningKey() (*ecdsa.PrivateKey, error) {
   key, err := ecdsa.GenerateKey(elliptic.P256(), rand.Reader)
   return key, err
}

// 用私钥对明文进行签名
func Sign(data []byte, privkey *ecdsa.PrivateKey) ([]byte, error) {
   // 对明文进行 sha256 散列，生成一个长度为 32 的字节数组
   digest := sha256.Sum256(data)

   // 通过椭圆曲线方法对散列后的明文进行签名，返回两个 big.int 类型的大数
   r, s, err := ecdsa.Sign(rand.Reader, privkey, digest[:])
   if err != nil {
      return nil, err
   }
   //将大数转换成字节数组，并拼接起来，形成签名
   signature := append(r.Bytes(), s.Bytes()...)
   return signature, nil
}

// 通过公钥验证签名
func Verify(data, signature []byte, pubkey *ecdsa.PublicKey) bool {
   // 将明文转换成字节数组
   digest := sha256.Sum256(data)

   //声明两个大数 r，s
   r := big.Int{}
   s := big.Int{}
   //将签名平均分割成两部分切片，并将切片转换成*big.int 类型
   sigLen := len(signature)
   r.SetBytes(signature[:(sigLen / 2)])
   s.SetBytes(signature[(sigLen / 2):])

   //通过公钥对得到的 r，s 进行验证
   return ecdsa.Verify(pubkey, digest[:], &r, &s)
} 
```