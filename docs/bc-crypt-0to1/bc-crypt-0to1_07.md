# 第七章 3DES 算法

3DES，或叫 3 重 DES，英文全称是 triple-DES，是普通 DES 的升级改进版。在 AES 未出现之前，DES 加密慢慢被发现存有较大的安全性，为此 3DES 作为过渡期的重要对称加密诞生了。1999 年，NIST 将 3-DES 指定为过渡的加密标准。
3DES 并不是一个全新的加密算法，它可以被认为是 DES 系列的加密范畴。DES 的密钥长度是 8 个字节，由于长度较短，较容易被暴力破解。增加密钥的长度成为提高 DES 安全性的重大突破口。密钥长度增加至 2 倍，也就是 2DES（双重 DES），但这个算法存在一种中间相遇攻击隐患，对其安全性构成了威胁，所以实际应用中，很少或不推荐双重 DES。密码长度增加至 3 倍，也就是 3DES。该算法不仅很大提高了 DES 的安全性，而且还可以抵抗中间相遇攻击。到目前为止，还没有相关它被暴力破解或其它安全性受到威胁的信息。尽管已经公布了高级加密标准 AES，但是目前 3DES 还被当作一个安全有效的加密算法在使用。

## 1\. 3DES 算法的原理及加解密过程

密钥长度为 192bit（也就是 24 字节），加密过程是进行 3 次 DES 加密或解密的密码算法叫 3DES。
由于当时 DES 算法的应用较多，所以设计 3DES 不得不考虑与 DES 的兼容问题，也就是 2 者之间可以混用，3DES 加密，DES 能够解密，DES 加密，3DES 能够解密。最终 IBM 公司设计出来了合理方案，将第 2 重加密过程改为解密过程，整体的加密过程是加密-->解密-->加密，当 3DES 的密钥是 DES 密钥的 3 次重复时，两者完全兼容，此时的 3DES 实际只有最后一重加密是有效的。如果 3DES 的密钥不是 DES 密钥的 3 次重复，此时两者不存在兼容，3DES 的第二重解密实际上也是加密过程，只不过用的 DES 的解密算法而已。
3DES 加密解密过程首先对输入的私钥平均分成 3 组，每组密钥对应一重 DES 算法，其具体实现如下：
设 Ek()和 Dk()代表 DES 算法的加密和解密过程，k 代表 DES 算法使用的密钥，M 代表明文，C 代表密文，这样：
3DES 加密过程为：C=Ek3(Dk2(Ek1(M)))
3DES 解密过程为：M=Dk1(EK2(Dk3(C)))

简单加密示意图如下：
![](img/9a514009c1f5e06638ec9522edb4ac10.jpg)

简单解密示意图如下：
![](img/20cc87d759bb1596b122e55d520e9640.jpg)

## 2\. Golang 实现 3DES_CBC 模式加解密

```go
package main

import (
    "bytes"
    "crypto/des"
    "crypto/cipher"
    "fmt"
)

//补码
func PKCS5Padding(ciphertext []byte, blocksize int)[]byte  {
  //求得补码的长度 x
    padding := blocksize-len(ciphertext)%blocksize
    //将 x 转换成字节，并创建一个长度为 x，元素都为 x 的切片
    padtext := bytes.Repeat([]byte{byte(padding)},padding)
    //返回补码后要加密的明文
    return append(ciphertext,padtext...)
}
//去码
func PKCS5UnPadding(origData []byte)[]byte  {
   //求得加密时补码的长度，长度就等于密文最后元素的 10 进制数字
    length:=len(origData)
    unpadding:=int(origData[length-1])
    //返回去码之后要进行解密的密文
    return origData[:(length-unpadding)]
}

//3DES 加密
////3DES 的密钥长度必须为 24 位
func TripleEncrypt(origData []byte,key[]byte)[]byte  {
    //通过调用 3des 库里方法产生分组密钥块
    block,_:=des.NewTripleDESCipher(key)
    //补码
    origData = PKCS5Padding(origData,block.BlockSize())
    //设置加密模式，此处用 CBC 模式
    blockMode:=cipher.NewCBCEncrypter(block,key[:8])
    //创建密文数组，加密
    crypted:=make([]byte,len(origData))
    //加密
    blockMode.CryptBlocks(crypted,origData)
    return crypted
}
//解密
func TrileDesDecrypt(crypted, key []byte)[]byte  {
  //设置分组的密钥块
    block,_:=des.NewTripleDESCipher(key)
    //设置解密模式
    blockMode:=cipher.NewCBCDecrypter(block,key[:8])
    //创建切片
    origData :=make([]byte,len(crypted))
    //解密
    blockMode.CryptBlocks(origData,crypted)
    //去码得到原文
    origData=PKCS5UnPadding(origData)
    return origData
}
func main()  {
    fmt.Println("hello world")
    var key=[]byte("123456789012345678901239")
    var encirtcode =TripleEncrypt([]byte("hello world"),key)
    var decryptcode=TrileDesDecrypt(encirtcode,key)
    fmt.Printf("%x\n",encirtcode)
    fmt.Println(string(decryptcode))

} 
```