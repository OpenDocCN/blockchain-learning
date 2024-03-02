# 第十一章 DSA 算法

DSA（Digital Signature Algorithm）是 Schnorr 和 ElGamal 签名算法的变种，被美国 NIST 作为 DSS(DigitalSignature Standard)。
DSA 加密算法主要依赖于整数有限域离散对数难题，素数 P 必须足够大，且 p-1 至少包含一个大素数因子以抵抗 Pohlig &Hellman 算法的攻击。M 一般都应采用信息的 HASH 值。DSA 加密算法的安全性主要依赖于 p 和 g，若选取不当则签名容易伪造，应保证 g 对于 p-1 的大素数因子不可约。其安全性与 RSA 相比差不多。

DSA 一般用于数字签名和认证。在 DSA 数字签名和认证中，发送者使用自己的私钥对文件或消息进行签名，接受者收到消息后使用发送者的公钥来验证签名的真实性。DSA 只是一种算法，和 RSA 不同之处在于它不能用作加密和解密，也不能进行密钥交换，只用于签名,它比 RSA 要快很多.

## 1\. DSA 签名及验证

DSA 算法中应用了下述参数：

p：L bits 长的素数。L 是 64 的倍数，范围是 512 到 1024；

q：p – 1 的 160bits 的素因子；

g：g = h^((p-1)/q) mod p，h 满足 h < p – 1, h^((p-1)/q) mod p > 1；

x：x < q，x 为私钥 ；

y：y = g^x mod p ，( p, q, g, y )为公钥；

H( x )：One-Way Hash 函数。DSS 中选用 SHA( Secure Hash Algorithm )。

p, q, g 可由一组用户共享，但在实际应用中，使用公共模数可能会带来一定的威胁。

签名及验证协议：
1.P 产生随机数 k，k < q；
2.P 计算 r = ( g^k mod p ) mod q
s = ( k^(-1) (H(m) xr)) mod q
签名结果是( m, r, s )。
3.验证时计算 w = s^(-1)mod q
u1 = ( H( m ) *w ) mod q
u2 = ( r* w ) mod q
v = (( g^u1 * y^u2 ) mod p ) mod q
若 v = r，则认为签名有效。

## 2\. golang 实现 DSA 签名及验证

```go
package main

import (
    "crypto/dsa"
    "crypto/rand"
    "fmt"

)
//作用 1 确保传递数据的完整性 2 确保数据的来源
func main()  {
    //DSA 专业做签名和验签
    var param dsa.Parameters//结构体里有三个很大很大的数 bigInt
    //结构体实例化
    dsa.GenerateParameters(&param,rand.Reader,dsa.L1024N160)//L 是 1024，N 是 160，这里的 L 是私钥，N 是公钥初始参数
    //通过上边参数生成 param 结构体，里面有三个很大很大的数

    //生成私钥
    var priv dsa.PrivateKey//privatekey 是个结构体，里面有 publickey 结构体，该结构体里有 Parameters 字段
    priv.Parameters=param
    //通过随机读数与 param 一些关系生成私钥
    dsa.GenerateKey(&priv,rand.Reader)

    //通过私钥生成公钥
    pub:=priv.PublicKey
    message:=[]byte("hello world")
    //r,s 是两个整数,通过私钥给 message 签名,得到两个随机整数 r，s
    r,s,_:=dsa.Sign(rand.Reader,&priv,message)

    //利用公钥验签，验证 r，s
    b:= dsa.Verify(&pub,message,r,s)
    if b==true{
        fmt.Println("验签成功")
    }else {
        fmt.Println("验证失败")
    }
} 
```