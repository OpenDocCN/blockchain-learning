# 第三章 序列密码

序列密码，也叫流密码，是利用种子密钥通过密钥流生成器产生与明文长度一致的伪随机序列，该随机序列与明文进行某种算法相结合产生的密文的一种密码算法。

使用序列密码对某一消息 m 执行加密操作时，都是按字进行加密的，一般是先将 m 分成连续的字符，m=m1m2m3…;然后使用密钥流 k=k1k2k3...中的第 i 个字符 ki 对明文消息的第 i 字符 mi 执行加密变换，i=1,2,3...；所有的加密输出连接在一起就构成了对 m 执行加密后的密文。解密需要和加密同步进行，所以使用序列密码，发送方和接收方需要对明文或密文在信息的同一位置进行操作。
加解密过程示意图如下：
![](img/aa95c3956b42fab9acc48455301a7900.jpg)

序列密码具有实现简单、便于硬件实施、加解密处理速度快、没有或只有有限的错误传播等特点，但是因为这类密码主要运用在军事，政治机密机构上，因此它的研究成果较少有公开。目前可以公开在其他领域应用的算法有 RC4，SEAL，A5 等。

## 1 . 一次一密密码

一次一密加密是 1917 年 Mauborgne 和 Vernam 联合提出的一种理想加密方案，它要求对明文消息逐字符加密，每个字符加密都是独立的，加密的密钥是与明文长度一致的毫无规则随机的密钥序列，这个密钥序列就是一次一密乱码本。在使用时，密文发送方和接收方手里头都拥有一个一模一样的乱码本，该乱码本是由双方协商确定的一个足够长的随机密钥序列。发送方对明文进行加密时，用到的密钥序列是来自乱码本发送消息长度的最前面的一段，加密完后，立马对用过的密钥序列进行销毁。接收方收到密文后，用乱码本的密钥序列依次对密文进行解密，得到明，同时也要把用过的密钥序列进行销毁，不再使用。

一次一密的乱码本被认为是无条件安全的密码体制，即是一种不可攻破的密码体制。窃听者得到密文信息，根本没有可能对其进行解密，因为加密的密钥是完全随机的，毫无规律，攻击者没有任何信息对它进行密文分析。但前提是一次一密乱码本不能泄漏。
一次一密概念的提出对序列密码的产生提供了方向。正是基于这个概念，越来越多的序列密码算法不断产生。

## 2\. 序列密码的结构

序列密码的结构可细分为同步流密码和自同步流密码。同步流密码指它的密钥流的产生与明文无关，而是通过某种独立的随机方法产生的伪随机数字流。自同步流密码也叫异步流密码，它与同步流密码相反，密钥流的产生与明文有关，具体是后一个密钥字的产生与前一个明文加密后的字有关。自同步密码这个特性，使得它非常难以研究，所以大部分序列密码的研究都集中在同步密码上。

### 2.1 同步流密码

同步流密码产生密码流的过程分为两部分，一个是密钥流产生器，另一个是加密变换器。
加密过程表达式是：c[i]=E(k[i],m[i])，参数都是字节数组的单个元素。解密过程和加密过程必须同步，表达式是一个。因为密钥流的产生每次都是不一样的。所以加密时，每次产生的密钥流元素先缓存到寄存器中，等解密用完这个元素以后再继续进行加密。整个过程有点类似 tcp 协议。目前最为常用的流密码体制是有限域 GF(2)上的二元加法流密码，其加密变换可表示为 c[i]=k[i]⊕m[i]。
特点：
1）同步要求
在同步流密码中，发送方和接收方必须是同步的，即双方使用同样的密钥，对同一位置进行操作。一旦密文字符在传输中出现丢失，损坏或者删除，那么解密将失败
2）无错误传播
密文字符在传输过程中被修改，只是对该字符产生影响，并不影响其他密文字符的解密。
3）主动攻击性破坏同步性
作为同步要求的结果，主动攻击者对传输中的密文字符进行重放，插入，删除等破坏操作，直接会造成加解密过程的同步性。所以在使用时，需要借助其他密码学技术对传输的密文进行认证和完整性的验证操作。

### 2.2 自同步流密码

自同步密码的密钥流的产生不独立于明文流和密文流，通常第 i 个密钥字的产生不仅与主密钥有关，而且与前面已经产生的若干个密文字有关。
特点：
1）自同步
发送方在传输密文流过程中，某些密文字符被攻击，接收方的解密只是在这些被攻击过的密文与发送方不同步，而其他密文流解密同步不会有问题。
2）有限的错误传播
接收方的解密只是对攻击过的 i 个密文字符有影响，而对其他密文流不会有问题。所以产生的明文至多有 i 个错误。
3）主动攻击破坏当前的同步性
4）明文统计扩算
每个明文字符都会影响其后的整个密文，即明文的统计学特性扩散到了密文中。因此，自同步流密码在抵抗利用明文冗余而发起的攻击方面要强于同步流密码。

### 2.3 密钥流生成器

流密钥的重要部分是密钥流生成器。理想的密钥流生成器是生成完全随机的密钥流，但实际中因为它是根据用户的私钥通过一定的算法产生的，不可能做到真正的随机，所以产生的密钥流是伪随机序列。
一个密钥流生成器通常由一个线形反馈移位寄存器（LFSR）和一个非线形组合部分构成。线形反馈移位寄存器可以称为驱动部分。其工作原理是将驱动部分在 j 时刻的状态变量 x 作为输入，输入到非线形组合部分 f，将 f（x）作为当前时刻的 k[j]。驱动部分负责提供非线形组合部分使用的序列，而非线形部分以各时刻移位寄存器的状态组合出密钥序列 j 时刻的值 k[j]。通俗讲就是驱动部分内部的变量是不断变化的，在每个不同时刻的值都是不一样的，它不断向非线形组合部分输入变量 x 的不同时刻的值，非线形组合部分接收到此时刻的 x，通过函数 f 产生当前的密钥流字节。

### 2.4 反馈移位寄存器

反馈移位寄存器是流密码产生密钥流的一个重要组成部分，GF(2)上一个 n 级反馈移位寄存器由 n 个二元存储器和一个反馈函数 f(a[1],a[2],a[3],a[4],...,a[n])组成，n 级反馈移位寄存器如下图所示。
![](img/336df027715668a91c006694e0c368cc.jpg)
每一存储器称为移位寄存器的一级，在任一时刻，这些级的内容构成反馈位移寄存器的状态，在每一状态对应 GF(2)上一个维向量，总共有 2^n 种可能的状态。每一时刻的状态可用长为 n 的序列 a[1]，a[2]，a[3]，...，a[n]或者 n 维向量（a[1]，a[2]，a[3]，...，a[n]）表示，其中 a[i]是当前时刻第 i 级存储器的内容。初始状态由用户确定，当第 i 个移位时钟脉冲到来时，每一级存储器 a[i]都将其内容向下一级 a[i-1]传递，反馈函数 f(a[1],a[2],a[3],a[4],...,a[n])根据寄存器当前的状态计算出下一时刻的 a[n]。反馈函数是一个 n 元布尔函数，即 n 个变量 a[1]，a[2]，a[3]，...，a[n]可以分别独立地取 0 和 1 两个可能的值，函数中的运算有逻辑与、逻辑或、逻辑补等运算，最后的函数值为 0 或 1.