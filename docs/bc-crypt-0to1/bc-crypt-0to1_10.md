# 第十章 RSA 算法

RSA 是目前使用最广泛的公钥密码体制之一。它是 1977 年由罗纳德·李维斯特（Ron Rivest）、阿迪·萨莫尔（Adi Shamir）和伦纳德·阿德曼（Leonard Adleman）一起提出的。当时他们三人都在麻省理工学院工作。RSA 就是他们三人姓氏开头字母拼在一起组成的。
RSA 算法的安全性基于 RSA 问题的困难性，也就是基于大整数因子分解的困难性上。但是 RSA 问题不会比因子分解问题更加困难，也就是说，在没有解决因子分解问题的情况下可能解决 RSA 问题，因此 RSA 算法并不是完全基于大整数因子分解的困难性上的。

## 1\. RSA 算法描述

### 1.1 RSA 产生公私钥对

具体实例讲解如何生成密钥对
1.随机选择两个不相等的质数 p 和 q。
alice 选择了 61 和 53。（实际应用中，这两个质数越大，就越难破解。）

2.计算 p 和 q 的乘积 n。
n = 61×53 = 3233
n 的长度就是密钥长度。3233 写成二进制是 110010100001，一共有 12 位，所以这个密钥就是 12 位。实际应用中，RSA 密钥一般是 1024 位，重要场合则为 2048 位。

3.计算 n 的欧拉函数φ(n)。称作 L
根据公式φ(n) = (p-1)(q-1)
alice 算出φ(3233)等于 60×52，即 3120。

4.随机选择一个整数 e，也就是公钥当中用来加密的那个数字
条件是 1< e < φ(n)，且 e 与φ(n) 互质。
alice 就在 1 到 3120 之间，随机选择了 17。（实际应用中，常常选择 65537。）

5.计算 e 对于φ(n)的模反元素 d。也就是密钥当中用来解密的那个数字
所谓"模反元素"就是指有一个整数 d，可以使得 ed 被φ(n)除的余数为 1。ed ≡ 1 (mod φ(n))
alice 找到了 2753，即 17*2753 mode 3120 = 1

6.将 n 和 e 封装成公钥，n 和 d 封装成私钥。
在 alice 的例子中，n=3233，e=17，d=2753，所以公钥就是 (3233,17)，私钥就是（3233, 2753）。

### 1.2 RSA 加密

首先对明文进行比特串分组，使得每个分组对应的十进制数小于 n，然后依次对每个分组 m 做一次加密，所有分组的密文构成的序列就是原始消息的加密结果，即 m 满足 0<=m<n，则加密算法为：
c≡ m^e mod n; c 为密文，且 0<=c<n。

### 1.3 RSA 解密

对于密文 0<=c<n，解密算法为：
m≡ c^d mod n;

### 1.4 RSA 签名验证

RSA 密码体制既可以用于加密又可以用于数字签名。下面介绍 RSA 数字签名的功能。
已知公钥（e，n），私钥 d
1.对于消息 m 签名为：sign ≡ m ^d mod n
2.验证：对于消息签名对（m，sign），如果 m ≡ sign ^e mod n，则 sign 是 m 的有效签名

## 2\. 已知公私钥，golang 实现 RSA 加解密

```go
package main

import (
    "crypto/rand"
    "crypto/rsa"
    "crypto/x509"
    "encoding/base64"
    "encoding/pem"
    "errors"
    "fmt"
)

// 可通过 openssl 产生
//openssl genrsa -out rsa_private_key.pem 1024
var privateKey = []byte(`  
-----BEGIN RSA PRIVATE KEY-----
MIICXQIBAAKBgQDfw1/P15GQzGGYvNwVmXIGGxea8Pb2wJcF7ZW7tmFdLSjOItn9
kvUsbQgS5yxx+f2sAv1ocxbPTsFdRc6yUTJdeQolDOkEzNP0B8XKm+Lxy4giwwR5
LJQTANkqe4w/d9u129bRhTu/SUzSUIr65zZ/s6TUGQD6QzKY1Y8xS+FoQQIDAQAB
AoGAbSNg7wHomORm0dWDzvEpwTqjl8nh2tZyksyf1I+PC6BEH8613k04UfPYFUg1
0F2rUaOfr7s6q+BwxaqPtz+NPUotMjeVrEmmYM4rrYkrnd0lRiAxmkQUBlLrCBiF
u+bluDkHXF7+TUfJm4AZAvbtR2wO5DUAOZ244FfJueYyZHECQQD+V5/WrgKkBlYy
XhioQBXff7TLCrmMlUziJcQ295kIn8n1GaKzunJkhreoMbiRe0hpIIgPYb9E57tT
/mP/MoYtAkEA4Ti6XiOXgxzV5gcB+fhJyb8PJCVkgP2wg0OQp2DKPp+5xsmRuUXv
720oExv92jv6X65x631VGjDmfJNb99wq5QJBAMSHUKrBqqizfMdOjh7z5fLc6wY5
M0a91rqoFAWlLErNrXAGbwIRf3LN5fvA76z6ZelViczY6sKDjOxKFVqL38ECQG0S
pxdOT2M9BM45GJjxyPJ+qBuOTGU391Mq1pRpCKlZe4QtPHioyTGAAMd4Z/FX2MKb
3in48c0UX5t3VjPsmY0CQQCc1jmEoB83JmTHYByvDpc8kzsD8+GmiPVrausrjj4p
y2DQpGmUic2zqCxl6qXMpBGtFEhrUbKhOiVOJbRNGvWW
-----END RSA PRIVATE KEY-----
`)

//openssl
//openssl rsa -in rsa_private_key.pem -pubout -out rsa_public_key.pem
var publicKey = []byte(`  
-----BEGIN PUBLIC KEY-----
MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQDfw1/P15GQzGGYvNwVmXIGGxea
8Pb2wJcF7ZW7tmFdLSjOItn9kvUsbQgS5yxx+f2sAv1ocxbPTsFdRc6yUTJdeQol
DOkEzNP0B8XKm+Lxy4giwwR5LJQTANkqe4w/d9u129bRhTu/SUzSUIr65zZ/s6TU
GQD6QzKY1Y8xS+FoQQIDAQAB
-----END PUBLIC KEY-----    
`)

// 加密
func RsaEncrypt(origData []byte) ([]byte, error) {
    //解码 pem 格式的公钥，得到公钥的载体 block
    block, _ := pem.Decode(publicKey)
    if block == nil {
        return nil, errors.New("public key error")
    }
    // 解析得到公钥
    pubInterface, err := x509.ParsePKIXPublicKey(block.Bytes)
    if err != nil {
        return nil, err
    }
    // 接口类型断言
    pub := pubInterface.(*rsa.PublicKey)
    //加密
    return rsa.EncryptPKCS1v15(rand.Reader, pub, origData)
}

// 解密
func RsaDecrypt(ciphertext []byte) ([]byte, error) {
    //解码 pem 格式的私钥，得到公钥的载体 block
    block, _ := pem.Decode(privateKey)
    if block == nil {
        return nil, errors.New("private key error!")
    }
    //解析得到 PKCS1 格式的私钥
    priv, err := x509.ParsePKCS1PrivateKey(block.Bytes)
    if err != nil {
        return nil, err
    }
    // 解密
    return rsa.DecryptPKCS1v15(rand.Reader, priv, ciphertext)
}

func main() {
    data, _ := RsaEncrypt([]byte("hello world"))
    fmt.Println(base64.StdEncoding.EncodeToString(data))
    origData, _ := RsaDecrypt(data)
    fmt.Println(string(origData))
} 
```

## 3\. golang 实现 RSA 生成公私钥并加解密

```go
package main

import (
    "crypto/rsa"
    "crypto/rand"
    "fmt"
    "crypto/md5"
    "encoding/base64"
)

//通过 RSA 实现加密和解密
//利用 RSA 的方法生成私钥对

func main()  {
    //RSA 首先生成的是私钥，然后根据私钥生成公钥
    //生成 1024 位私钥
    pri,_:=rsa.GenerateKey(rand.Reader,2048)
    //根据私钥产生公钥
    pub:=&pri.PublicKey
    //fmt.Println("私钥",pri)
    //fmt.Println("公钥",pub)
    //定义明文
    plaintext:=[]byte("hello china")
    //加密成密文,OAEP 补码
    ciphertext,_:=rsa.EncryptOAEP(md5.New(),rand.Reader,pub,plaintext,nil)
    fmt.Println(base64.StdEncoding.EncodeToString(ciphertext))

    //解密
    plaintext,_=rsa.DecryptOAEP(md5.New(),rand.Reader,pri,ciphertext,nil)
    fmt.Println(string(plaintext))
} 
```

## 4\. golang 实现 RSA 签名和验签

```go
package main

import (
    "crypto/rsa"
    "crypto/rand"
    "crypto/md5"
    "crypto"
    "fmt"
)

//RSA 实现签名和验签

func main()  {
    //生成私钥
    priv,_:=rsa.GenerateKey(rand.Reader,1024)
    //产生公钥
    pub:=&priv.PublicKey
    //设置明文
    plaintext:=[]byte("hello world")
    //给明文做哈希散列
    h:=md5.New()
    h.Write(plaintext)
    hashed:=h.Sum(nil)
    //签名
    opts:=&rsa.PSSOptions{SaltLength:rsa.PSSSaltLengthAuto,Hash:crypto.MD5}
    sign,_:=rsa.SignPSS(rand.Reader,priv,crypto.MD5,hashed,opts)

    //认证
    e:=rsa.VerifyPSS(pub,crypto.MD5,hashed,sign,opts,)
    if e==nil{
        fmt.Println("验证成功")
    }else {
        fmt.Println("验证失败")
    }

} 
```