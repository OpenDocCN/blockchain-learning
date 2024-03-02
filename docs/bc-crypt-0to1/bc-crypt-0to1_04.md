# 第四章 RC4 算法

RC4 加密算法是 RSA 三人组中的头号人物 Ron Rivest 在 1987 年设计的密钥长度可变的流加密算法簇。该算法的速度可以达到 DES 加密的 10 倍左右，且具有很高级别的非线性。1994 年 9 月，它的算法被发布在互联网上。由于 RC4 算法加密是采用的 xor，所以，一旦子密钥序列出现了重复，密文就有可能被破解。RC4 作为一种老旧的验证和加密算法易于受到黑客攻击，现在逐渐不推荐使用了。

## 1\. RC4 实现过程

RC4 算法的实现非常简单，使用从 1 到 256 个字节（8 到 2048 位）可变长度密钥初始化一个 256 个字节的状态向量 S，S 的元素记为 S[0]，S[1]，S[2]，...，S[255]，S 先初始化为 S[i]=i。以后自始至终都包含从 0 到 255 的所有 8 比特数，只是对它进行置换操作。每次生成的密钥字节 k[i]由 S 中 256 个元素按一定方法选出一个元素而生成。每生成一个密钥字节，S 向量中元素会进行一次置换操作。则 RC4 算法分为两部分：初始化 S 和密钥流的生成，其中密钥流的生成过程中每次产生的密钥字与对应明文的元素进行异或运算得到密文字。

### 1.1 初始化 S

生成 S 的步骤如下：
1）声明一个长度为 256 的字节数组，并给 S 中的元素从 0 到 255 以升序的方式填充，即 S[0]=0，S[1]=1，S[2]=2，...，S[255]=255。
2）j:=0
3）对于 0<=i<=255，循环下边两个方法：
j = (j + S[i] + int(K[i%keylen])) % 256
S[i], S[j]=S[j], S[i]

### 1.2 密钥流的生成

步骤如下：
1）i=0；j=0
2）i = (i + 1) % 256
3）j = (j + S[i]) % 256
4）S[i], S[j]=S[j], S[i]
5）输出密钥字 key = S[(S[i]+S[j])%256]

### 1.3 RC4 的安全性

由于 RC4 算法加密采用的是异或方式，所以，一旦子密钥序列出现了重复，密文就有可能被破解，但是目前还没有发现密钥长度达到 128 位的 RC4 有重复的可能性，所以，RC4 也是目前最安全的加密算法之一。

### 1.4 RC4 加密过程

简单介绍下 RC4 的加密过程：
1）利用自己的密钥，产生密钥流发生器
2）密钥流发生器根据明文的长度产生伪随机序列
3）伪随机序列每个位元素与明文对应的位元素进行异或运算，生成密文
示意图：
![](img/c0c547a615f7ac4bebbe1638d5d9a983.jpg)

## 2\. golang 实现 RC4 加密

### 2.1 golang 实现 RC4 加密：

```go
package main

import "fmt"

func main() {

    data := []byte("helloworld");
    output:=make([]byte,len(data))
    fmt.Printf("明文:%s\n", data);

    K := []byte("qwuoaknfabbalafbj");
    keylen := len(K);
    SetKey(K, keylen);

    output=Transform(output, data, len(data));
    fmt.Printf("密文: %x\n", output);

    SetKey(K, keylen);
    output1:=make([]byte,len(data))
    output1=Transform(output1, output, len(data));
    fmt.Printf("解密后明文:%s", output1);
}

var S = [256]int{}
//初始化 S 盒
func SetKey(K []byte, keylen int) {
    for i := 0; i < 256; i++ {
        S[i] = i;
    }

    j := 0;
    for i := 0; i < 256; i++ {
        j = (j + S[i] + int(K[i%keylen])) % 256;

        S[i], S[j]=S[j], S[i];

    }

}

//生成密钥流
func Transform(output []byte, data []byte, lenth int)[]byte {
    i := 0;
    j := 0
    output=make([]byte,lenth)

    for k := 0; k < lenth; k++ {
        i = (i + 1) % 256;
        j = (j + S[i]) % 256;
        S[i], S[j]=S[j], S[i];
        key := S[(S[i]+S[j])%256];
        //按位异或操作

        output[k] = uint8(key)^data[k];

    }
    return output
} 
```

### 2.2 利用 goland 封装好的方法实现 RC4 加密，但是没有解密

```go
package main
import (
    "fmt"
    "crypto/rc4"
)
func main()  {
var key []byte = []byte("fd6cde7c2f4913f22297c948dd530c84") //初始化用于加密的 KEY
rc4obj, _ := rc4.NewCipher(key) //返回 Cipher

str := []byte("helloworld")  //需要加密的字符串
plaintext := make([]byte, len(str)) //
rc4obj.XORKeyStream(plaintext, str)
//XORKeyStream 方法将 src 的数据与秘钥生成的伪随机位流取 XOR 并写入 dst。
//plaintext 就是你加密的返回过来的结果了，注意：plaintext 为 base-16 编码的字符串，每个字节使用 2 个字符表示 必须格式化成字符串

stringinf := fmt.Sprintf("%x\n", plaintext) //转换字符串
fmt.Println(stringinf)
} 
```