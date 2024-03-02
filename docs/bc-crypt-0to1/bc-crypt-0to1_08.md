# 第八章 AES 算法

AES 英文全称是 Advanced Encryption Standard,中文是高级加密标准。它的出现就是为了替代之前的加密标准 DES。1997 年 1 月 2 号，美国国家标准技术研究所（National Institute of Standards and Technology: NIST）发起征集高级加密标准算法的活动，目的是重新确立一种新的分组密码代替 DES，成为新的美联邦信息处理的标准。该活动得到了全世界很多密码工作者的响应，先后有很多人提交了自己设计的算法。最终获胜的是由两位比利时的著名密码学家 Joan Daemen 和 Vincent Rijmen 设计的 Rijndael 算法。2001 年 11 月，NIST 正式公布该算法，命名为 AES 算法。

## 1\. AES 算法原理

AES 算法采用分组密码体制，即 AES 加密会首先把明文切成一段一段的，而且每段数据的长度要求必须是 128 位 16 个字节，如果最后一段不够 16 个字节，需要用 Padding 来把这段数据填满 16 个字节，然后分组对每段数据进行加密，最后再把每段加密数据拼起来形成最终的密文。AES 算法的密钥长度可以有三种，分别是 128 位，256 位，512 位。
AES 算法的加密过程使用了四个变换：字节替换变换（SubBytes）、行移位变换（SiftRows）、列混淆变换（MixColumns）和轮密钥加变换（AddRoundKey）。解密过程用了这四个变换的逆操作，分别是逆字节替换变换（InvSubBytes）、逆行移位变换（InvShiftRows）、逆列混淆变换（InvMixColumns）和轮密钥加变换。这里说明一下，轮密钥加变换的逆运算就是它本身，所以名字就通用一个。

### 1.1 字节替换变换

字节替换变换是一个非线形变换。输入的任意字节我们看作是有限域 GF(2⁸ )的元素，也就是这些字节都会在这个有限域内找到。在这个有限域内任何元素通过映射运算都会找到与之对应的元素，而且他们之间映射是可逆的。根据这个映射关系，制作了一个 S 盒对照表。根据这个表我们会很容易的查找对应的映射元素。如果输入的字节为 xy，查找 S 盒中的第 x 行盒第 y 列找到对应的值，将其输出替换 xy 输出。例如 1D，替换之后就是 A4。S 盒的制作方法这里不做讲解。

下表就是 AES 的 S 盒：
![](img/99f65402b7f84ab911340368520c3d30.jpg)
注：S 盒中的元素都是 16 进制,字母大写表示。

### 1.2 逆字节替换变换

逆字节替换变换是字节替换变换的逆变换。字节替换变换的映射运算是可逆的，所以根据映射逆运算也制作了一张逆 S 盒的对照表。查找方法与字节替换变换方法一样。例如 A4，替换之后就是 1D。逆 S 盒的制作方法这里也不做讲解。
下表就是 AES 的逆 S 盒：
![](img/d639c7b1f33572af2ffca33e59bd001a.jpg)
注：逆 S 盒中的元素都是 16 进制,字母大写表示。

### 1.3 行移位变换

行移位的功能是实现一个 4x4 矩阵内部字节之间的置换。AES 算法的明文分组要求是每组的字节长度为 16，就是因为能够刚好转换成 4x4 矩阵。
行移位的过程：第一行保持不变，第二行循环左移 1 个字节，第三行循环左移 2 个字节，第四行循环左移 3 个字节。
示意图如下：
![](img/036ce6fe02b78401758be773433875ee.jpg)

### 1.4 逆行移位变换

逆向行移位即是相反的操作。即第一行保持不变，第二行循环右移 1 个字节，第三行循环右移 2 个字节，第四行循环右移 3 个字节。

### 1.5 列混淆变换

列混淆变换将状态矩阵中的每一列视为系数在 GF(2⁸ )上的次数小于 4 的多项式与同一个固定的多项式 a(x)进行模多项式 m(x)=x⁴ +1 的乘法运算。在 AES 中，a(x)={03}x³ +{01}x² +{01}x+{02}。

### 1.6 逆列混淆变换

逆列混淆变换是列混淆变换的逆，它将状态矩阵中的每一列视为系数在 GF(2^ 8)上的次数小于 4 的多项式与同一个固定的多项式 a^-1 (x)进行模多项式 m(x)=x⁴ +1 的乘法运算。a^-1 (x)={0B}x³ +{0D}x² +{09}x+{0E}。

### 1.7 轮密钥加变换

任何数和自身的异或结果为 0。加密过程中，每轮的输入与轮密钥异或一次；因此，解密时再异或上该轮的密钥即可恢复输入。

### 1.8 密钥扩展算法

AES 加密的每一轮用到的密钥都是不一样的。AES 密钥扩展算法的输入值是 4 个字（16 字节），输出值是一个由 44 个字组成（176 字节）的一维线性数组。

## 2\. AES 加密过程

AES 的解密就是加密的逆运算，所以这里我只讲解 AES 加密的过程。一般 AES 的每组明文加密需要重复 10 轮加密才能完成，所有分组的明文经过 10 轮加密后，拼接一起就是最后的密文。下面我只讲解一轮的加密过程。
每一轮的加密过程：

1.  将明文分成 N 组，每组长度为 128 比特，也就是 16 个字节；
2.  将分组明文字节输出转换成 16 进制（字母大写输出）；
3.  该轮分组
4.  将分组明文进行字节替换；
5.  字节替换后，转换成 4X4 矩阵，然后进行行移位变换；
6.  将列混淆变换后的 4X4 矩阵，再进行列混淆变换，输出处理后的字节数组；
7.  轮密钥加：经过第 6 步后的字节数组与轮密钥进行 XOR 异或运算得到新的字节数组；
8.  第 7 部回到第 4 步

    整体的加解密过程如下图：
    ![](img/cd1f4a383ce3ac78170aca0245cba491.jpg)

## 3\. AES 的安全性

AES 加密算法中，每轮使用不同的常数消除了密钥的对称性；使用了非对称性的密钥扩展算法消除了相同密钥的可能性；加密和解密使用不同的变换，从而消除了弱密钥和半弱密钥存在的肯能性。经过验证，AES 加密算法能有效地抵抗现有的攻击，如差分攻击、相关密钥攻击、插值攻击等。

## 4\. golang 实现 AES_CFB 加解密

AES_CFB 加解密代码实现：

```go
package main

import (
    "fmt"
    "crypto/aes"
    "io"
    "crypto/rand"
    "crypto/cipher"
    "encoding/base64"
)

//通过 CFB 模式，进行 AES 加密
//加密
func AESEncrypt(plaintext []byte, key []byte) []byte {
    //分组密钥,key 字节的长度必须是 16 或 24，或 32；密钥的长度可以使用 128 位、192 位或 256 位；位只有两种形式 0 和 1，而字节是有 8 个位组成的。可以表示 256 个状态。1 字节（byte）=8 位（bit）
    block,_:=aes.NewCipher(key)
    //block 是个*aes.aesCipherGCMiv 类型 包含 encode 和 decode  这两个 code 类型是[]uint32, n。 n=BlockSize+28，n 是新创建 enc 和 dec 的长度

    //创建数组，目的是存储你接下来加密的密文
    ciphertext:=make([]byte,aes.BlockSize+len(plaintext))
    //设置内存空间可读，类似在明文前面加入一个长度 16 切片，用于被读取内存流
    iv :=ciphertext[:aes.BlockSize]
    //[]uint8
    //[0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0]
    //读内存流,把 rand.reader 随机读取的内存流数据复制到 iv 里
    io.ReadFull(rand.Reader,iv)//n 是 iv 的长度
    //iv=[127 78 10 244 97 152 178 224 62 49 156 74 239 99 211 94]每次不一样，包含时间戳应该
    //设置加密模式，返回一个流，也就是把 iv 放到 block 里，返回 stream 流
    //下边方法将 iv 的数 copy 到 block 里的 next 字段的字节数组里
    stream :=cipher.NewCFBEncrypter(block,iv)

//&{0xc42007e2a0 [127 78 10 244 97 152 178 224 62 49 156 74 239 99 211 94]（输出和读出的 iv 一样） [0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0] 16 false}
    //拿着你的密文进行异或运算，加密利用 ciphertext[:aes.BlockSize]与明文进行异或,overlap 使部分重叠

    stream.XORKeyStream(ciphertext[aes.BlockSize:],plaintext)
    return ciphertext
}
//解密
func AesDecrypt(ciphertext []byte,key []byte)[]byte  {
    block,_:=aes.NewCipher(key)
    iv:=ciphertext[:aes.BlockSize]
    ciphertext=ciphertext[aes.BlockSize:]
    //设置解密方式
    stream :=cipher.NewCFBDecrypter(block,iv)
    //解密
    stream.XORKeyStream(ciphertext,ciphertext)
    return ciphertext
}
func main()  {
    fmt.Println("hello world")
    var encryptcode = AESEncrypt([]byte("abc"),[]byte("123456789abcdejg"))
    var decryptcode =AesDecrypt( encryptcode,[]byte("123456789abcdejg"))
    fmt.Println(string(decryptcode))
    fmt.Println(base64.StdEncoding.EncodeToString(encryptcode))
} 
```

## 5\. golang 实现 AES_CTR 加解密

AES_CTR 加解密代码实现：

```go
package main

import (
    "bytes"
    "crypto/aes"
    "crypto/cipher"
    "fmt"

)

func main()  {
    str:=[]byte("helloworld")
    key:=[]byte("12345678qwertyui")
    encrypt:=AesCTR_Encrypt(str,key)
    fmt.Printf("%x\n",encrypt)
    decrypt:=AesCTR_Decrypt(encrypt,key)
    fmt.Println(string(decrypt))
}

func AesCTR_Encrypt(plainText, key []byte) []byte {
    //判断用户传过来的 key 是否符合 16 字节，如果不符合 16 字节加以处理
    keylen := len(key)
    if keylen == 0 { //如果用户传入的密钥为空那么就用默认密钥
        key = []byte("Abskjeqiu1234567") //默认密钥
    } else if keylen > 0 && keylen < 16 { //如果密钥长度在 0 到 16 之间，那么用 0 补齐剩余的
        key = append(key, bytes.Repeat([]byte{0}, (16 - keylen))...)
    } else if keylen > 16 {
        key = key[:16]
    }
    //1.指定使用的加密 aes 算法
    block, err := aes.NewCipher(key)
    if err != nil {
        panic(err)
    }

    //2.不需要填充,直接获取 ctr 分组模式的 stream
    // 返回一个计数器模式的、底层采用 block 生成 key 流的 Stream 接口，初始向量 iv 的长度必须等于 block 的块尺寸。
    iv := []byte("wumansgygoaesctf")

    stream := cipher.NewCTR(block, iv)

    //3.加密操作
    cipherText := make([]byte, len(plainText))
    stream.XORKeyStream(cipherText, plainText)

    return cipherText
}

func AesCTR_Decrypt(cipherText, key []byte) []byte {
    //判断用户传过来的 key 是否符合 16 字节，如果不符合 16 字节加以处理
    keylen := len(key)
    if keylen == 0 { //如果用户传入的密钥为空那么就用默认密钥
        key = []byte("Abskjeqiu1234567") //默认密钥
    } else if keylen > 0 && keylen < 16 { //如果密钥长度在 0 到 16 之间，那么用 0 补齐剩余的
        key = append(key, bytes.Repeat([]byte{0}, (16 - keylen))...)
    } else if keylen > 16 {
        key = key[:16]
    }
    //1.指定算法:aes
    block, err := aes.NewCipher(key)
    if err != nil {
        panic(err)
    }
    //2.返回一个计数器模式的、底层采用 block 生成 key 流的 Stream 接口，初始向量 iv 的长度必须等于 block 的块尺寸。
    iv := []byte("wumansgygoaesctf")
    stream := cipher.NewCTR(block, iv)

    //3.解密操作
    plainText := make([]byte, len(cipherText))
    stream.XORKeyStream(plainText, cipherText)

    return plainText
}

//输出
40dbbf4ab999bb36d0d5
helloworld 
```