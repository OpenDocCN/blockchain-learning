# 第二章 密码学 hash 函数

# 第二章 密码学 hash 函数

所有货币都需要某种方式来控制供应，并执行各种安全属性以防止欺诈。在法定货币中，像中央银行这样的组织控制货币供应，并在实物货币中加入防伪功能。这些安全功能提高了攻击者的门槛，但它们也可以让钱出现伪造。最后，还是需要执行法律来制止人们违反系统规则。

加密数字货币也必须采取安全措施，以防止人们篡改系统状态，同时加密数字货币还需要防止“混淆”，也就是说，对不同的人做出相互不一致的声明。例如，如果 Alice 说服 Bob 她给了他一枚数字硬币，那么她就不能说服 Carol，让她相信她也给了她同一个硬币。加密数字货币与法定货币不同的是，它的安全规则需要完全依靠技术来执行，而不需要依赖中央机构。

顾名思义，加密数字货币着力采用密码技术。密码学提供一个将加密货币体系 规则编码到系统本身的机制，我们不但可以利用密码学防止对系统的干扰，并且能够避免混淆，也能用其将新货币单位创造规则编码到数学协议中。为了能够深刻理解加密数字货币系统，我们需要首先探究该系统所依赖的密码学基础。

密码学是一个高深的学术领域，用到了很多不被大众所知的数学理论，并且其理论也比较复杂。幸运的是，比特币只运用到了密码学中少量相对较为浅显的一些理论。在本章中，我们会特别讨论一下密码学中的哈希算法(Hash) 和数字签名(digital signature)技术，这两个基本概念对构建一个加密数字货币系统非常关键。

## 1 什么是 Hash

在本节，我们会讨论哈希计算。如果你已经熟悉了这个概念，可以直接跳过。

获得指定数据的一个哈希值的过程，就叫做哈希计算。一个哈希，就是对所计算数据的一个唯一表示。对于一个哈希函数，输入任意大小的数据，它会输出一个固定大小的哈希值。

1.  无法从一个哈希值恢复原始数据。也就是说，哈希并不是加密。

2.  对于特定的数据，只能有一个哈希，并且这个哈希是唯一的。

3.  即使是仅仅改变输入数据中的一个字节，也会导致输出一个完全不同的哈希。

![`img.kongyixueyuan.com/15_hash2.png`](img/1daeb949433acddcc49d8bf4ebd66470.jpg)

## 2 Hash 的特性

我们需要理解的第一个密码学的基础知识是密码学哈希函数，哈希函数是 一个数学函数，具有以下三个特性：

*   其输入可为任意大小的字符串。
*   它产生固定大小的输出。为使本章讨论更具体，我们假设输出值大小为 256 位，但是，我们的讨论适用于任意规模的输岀，只要其足够大。
*   它能进行有效计算，简单来说就是对于特定的输入字符串，在合理时间内，我们可以算出哈希函数的输出。更准确地说，对应 n 位的字符串， 其哈希值计算的复杂度为 0 (n)。

这些属性定义了一个通用哈希函数，可以用于构建数据结构，例如哈希表。 我们将只专注于加密哈希函数。 对于加密安全的哈希函数，我们将要求它具有以下三个附加属性：

(1)碰撞阻力(collision-resistance);

(2)隐秘性(hiding)；

(3)谜题友好(puzzle-friendliness)。

我们将更仔细地研究每一个属性，并会逐步阐释我们为什么需要这样的函数。 学习过密码的读者可能会注意到，我们这里对于哈希函数的论述与一般的密码学课程会有所不同，特别是关于谜题友好。在一般密码学中，谜题友好并非加密的哈希函数的一般要求，却对加密数字货币这一特性非常有用。

### 2.1 特性 1：碰撞阻力。

加密哈希函数的第一个属性是它的碰撞阻力(也理解为抗冲突性)。这里的碰撞指对于两个不同的输人，产生相同的输出。如果对于哈希函数 H (•)，没有人能够找到碰撞，我们则称该函数具有碰撞阻力(见图 1.1)。即：

```java
碰撞阻力：如果无法找到两个值，X 和 y，其中 x≠y，并且 H(x)= H(y)，则称哈希函数 H 具有碰撞阻力。 
```

![`img.kongyixueyuan.com/1001_hash%E5%87%BD%E6%95%B0.png`](img/4257f60fcdc633d5d3bf65c5caa36015.jpg)

X 和 y 分別是不同输人，当作为哈希函数的输入时，会产生相同的输出。这时我们就说这个函数是哈希碰撞的。

> H(x)代表的是将 x 带入 hash 函数中计算出来的 hash 值，也就是上面说的固定大小的字符串。
> 
> 也就是说如果 H(x)和 H(y)相等，但 x 和 y 不相等的话，就代表不同的输入字符会产生同样的结果，冲突了。

请注意，我们说没有人能找到碰撞，并不表示不存在碰撞，而是在有限的时间和计算能力来看，找到冲突的可能性几乎为零。 实际上，我们知道碰撞确实存在，我们可以通过一个简单的计数论证(counting argumenl)证明这一点。 哈希函数的输入空间包含所有长度的任意字符串，但输出空间则只包含特定固定长度的字符串。 因为输入空间大于输出空间（实际上，输入空间是无限的，而输出空间是有限的），所以一定会有输入字符串映射到相同的输出字符串。 实际上，根据鸽巢原理(Pi­geonhole Principle)，我们可以得岀，必然会有大量可能的输入映射到任何特定输出。

![`img.kongyixueyuan.com/1002_%E7%A2%B0%E6%92%9E.png`](img/216a301d80088109236845c0e00c6b9e.jpg)

因为输入的数量超过输出的数量，我们可以确定某一个输出，一定对应了哈希函数的多个输入。

现在更糟糕的是，对于加密的哈希函数，我们虽然说应该找不到碰撞，但有些方法是能保证找到碰撞的。下面这个方法，对应于一个 256 位输出大小的哈希函数，选择 $ 2^{256}+1 $个不同数值，计算每个数的哈希值，并检查是否有两个相等的输出。因为我们这里选择的输入多于输出，因此在应用哈希函数时，那铁定就能找到至少一对字符串是冲突的，因为我们尝试了所有的可能性。

使用上述方法一定能找到碰撞。但如果我们随机性地选择输入，并计算哈希值，我们在检验第$ 2^{256}+1 $个输入之前便很可能找到碰撞，实际上，如果我们随机选择$ 2^{130}+1 $个输入值，找到至少两个等同哈希值的概率为 99\. 8%。仅仅通过检验可能输出数量的平方根次数，便大体能找到碰撞，这一事实在概率学中 被称为是生日悖论(birlhday paradox)。

这个碰撞检测算法对每个哈希函数都有效，但是它的问题是其计算需要花 很长很长时间才能完成。对于一个 256-bit 输出的哈希函数来说，最坏的情况是你需要进行 $ 2^{256}+1 $次哈希函数计算，平均次数为 $ 2^{128} $次，这简直是一个天文数 字——如果一台电脑每秒计算 10000 个哈希值，计算 $ 2^{128} $个哈希值，需要花 $ 2^{27} $多年时间！换个角度，我们可以说，如果人类制造的每台电脑在整个宇宙起源时便开始计算，到目前为止，它们找到碰撞的概率仍然无穷小，比下两秒钟地球将被大陨石摧毁的概率还要小得多。

因此，为了寻找一个任意的哈希函数的碰撞，我们只是有了一个一般，但并不实用的算法。一个更艰难的问题是：有没有其他的方法，可以用于对于某一特定哈希函数找到碰撞？也就是说，虽然一般的碰撞测试算法不适用，但仍 可能有其他的算法，可以有效地找到某个哈希函数的碰撞。

以下面的哈希函数为例：

$ H(x) = x\quad mod\quad2^{256} $​

这个函数接受任何长度的输人，产生一个固定大小输出(256 位)，且能进行有效计算，因此符合我们对哈希函数的要求。但是对于这个函数，我们确实具备一个有效的能够寻找碰撞的方法。注意，这个函数仅返回输入的最后 256 位，因 此，数值$3$和 $3+ 2^{256} $就构成了碰撞。这个简单的例子表明，虽然我们的一般碰撞测试方法在实践中不可行，但至少对于某些哈希函数，存在有效的测试碰撞的方法。

然而对于某些哈希函数，我们无法确认识别碰撞的有效方法是否存在，我们只是怀疑这些函数具有防碰撞特性，但是我们已经证明，世界上没有哈希函数具有防碰撞特性。我们实践中依赖的加密的哈希函数仅仅是人们经过不懈努力之后暂未成功找到碰撞的函数。因此，我们选择相信那些加密的哈希函数具有哈希阻力。在某些情况下，如之前的 MD5 哈希函数，在多年的努力之后最终找到了碰撞，导致该函数在实践中被逐渐淘汰，最终被弃用。

> 生日悖论(birlhday paradox)是指，加果一个房间里有 23 个或 23 个以上的人，那么至少有两个人的生日相同的概率要大于 50%。这就意味着在一个典型的标准小学班级(30 人)中，存在两人生日相同的可能性更高。对于 60 或者更多的人，这种概率要大于 99%。这个概率的计算假定了一年是 365 天，不算闰年。
> 
> $3$和$ 3+2^{256} $对$2^{256}$求余数之后，结果都是$3$。
> 
> 通俗的再次解释一下：不管怎么算，$ 2^{128}+1 $这个数字的意味着，不可能用暴力破解。因为简单估算一下，如果一秒钟算$ 2^{30} $ 个（十亿）hash 函数，十亿台电脑一起算，大概需要$ 2^{70} $秒。一年估摸着有$ 2^{25} $ 秒，所以需要$ 2^{45} $ 年。然后宇宙至今为止大概$ 2^{35} $岁。所以即使是这么多台电脑从大爆炸开始就狂算，到现在有可能一个冲突都还没找到。

**应用：信息摘要**

现在我们知道什么是碰撞阻力了，我们自然会问：碰撞阻力有什么用途？ 以下就是一个应用：哈希函数 H 具有碰撞阻力，x 和 y 是两个不同的输入，那 么可以假设它们的哈希函数 H (x)和 H (y)也不同——如果已知 x 和 y 不同， 但哈希值相同，那么 H 具有碰撞阻力的假设就不成立了。

这个论证使我们可以将哈希输出作为信息摘要(message digest)。以 Secure- Box 为例，SecureBox 是一个允许用户上传文件，并保证文件被完整下载的线上 文件存储系统。假设爱丽丝上传了很大的文件，并希望能够在之后下载时确认她下载的文件与她上传的文件相同。一种方法是将整个文件进行本地存储，并直接将其与下载文件对比。如果这样可行，那么将文件上传便显得毫无意义，倘若爱丽丝需要使用本地文件副本以保证其完整性，她可以直接使用本地副本。

无碰撞哈希函数为这个问题提供了简单有效的解决方法，爱丽丝只需要记住原文件的哈希值，从 SecureBox 下载文件后，她可以计算下载文件的哈希值，并与原文件哈希值进行对比。如果哈希值相同，那么爱丽丝可以说该文件就是她上传的那一个，但是如果不同，她则可以确定文件被破坏了。记录哈希值可以帮助爱丽丝检测文件在传输过程中，或在 SecureBox 服务器上是否产生了意外损坏，或者检测文件是否受到服务器的蓄意修改。保证主体不受其他实体的恶意行为侵害，这正是密码学的核心。

这里的哈希函数对于一个信息生成固定长度的摘要，或生成了简明总结， 这为我们提供了一种记住之前所见事物，并在今后认出这些事物的有效方法。

虽然整个文件可能非常大，存储规模达数 G，但其哈希值的长度固定。例如， 哈希函数为 256 位。这样做，极大地降低了存储要求。

### 2.2 特性 2：隐秘性

我们希望哈希函数拥有的第二个特性是其隐秘性。隐秘性保证，如果我们仅仅知道哈希函数的输出 y = H (x)，我们没有可行的办法算出输入值 x。问题是，上述的表示形式不一定是正确的。考虑以下简单的例子：我们做一个拋硬 币的实验，如果拋硬币结果为正面，我们会宣布字符串哈希为“正面”；如果结果为反面，我们会宣布字符串哈希为“反面”。

然后我们问我们的对手，在他没有见到拋硬币，而只见到哈希函数的输出的前提下说出哈希函数的输入字符串(很快我们就知道为什么要玩这个游戏了)。为了回答问题，对手会简单计算“正面”字符串的哈希值及“反面”字符串的哈希值，然 后对手便可以知道他得到的是哪一个。这样，只需要几步，对手就能反解出输入值。

对手能够猜岀字符串，这是因为 x 只有两个可能，他可以很轻易地将两个可能对应的哈希值计算出来。为了能够实现隐秘性，我们需要 x 的取值来自一个非常广泛的集合，也就是说，仅仅通过尝试几个特定的 x，就能找到输出值的方式将不会发生了。

现在的问题是：在类似拋硬币的“正面”、“反面”实验中，如果我们想要的反解的输入值并非来自分散的集合，我们是否还能得到隐秘性？幸运的是， 对于这个问题答案是肯定的！我们甚至能够通过与另一个较为分散的输入进行结合，而将一个并不分散的输人进行隐秘。现在我们可以更精确地表示隐秘的含义了(双竖线||为连接符号，代表把一系列事件、事情等联系起来)。

隐秘性哈希函数 H 具有隐秘性，如果：当其输入 r 选自一个高阶最小熵(high min-entroy)的概率分布，在给定 H (r||x)条件下来确定 x 是不可行的。

> 在信息论中，最小熵是用于测试结果可预测性的手段，而高阶最小熵这个概念比较直观描述了分布(如随机变量)的分散程度。具体来说，在从这样分布中取样时，我们将无法判定取样的倾向。举个具体的例子，如果 r 是从长度为 256 位的字符串中随意选出的，那么任意特定字符串被选中的概率为$ 1/2^{256} $ ,这 是一个小到几乎可以忽略的取值。

**应用：承诺**

现在来看一下隐秘性的应用。具体来说，我们把想做的事情称为承诺（com­mitment）。 这里承诺是一个数字化过程，可以类比为以下动作：首先选定一个数字，将数字装进信封，然后将该信封放到一个人人都看得到的桌子上。这样做以后，可以说你就信封里的数字做出了承诺，在打开信封前，虽然你已经做出了承诺，对其他人来说它还是秘密。在之后，你可以打开信封，来展示承诺的数值。

**承诺协议**

一个承诺协议方案由两个算法构成：

*   com = commit （ msg, nonce），承诺函数将信息（msg）和一个临时随机数（nonce）作为输入，输出就是一个“承诺”。
*   verify （com，msg, nonce），验证函数将某个承诺输出（com）、临时随机 数（nonce）及信息（msg）作为输入，如果 com == commit （ msg, nonce）,则返回“真”（true）;反之则返回“假”（false）。

我们要求以下两个安全特性要成立：

*   隐秘性：已知 com，没有可行的方法找到 msg。
*   约束性：没有可行的办法找到两组（msg，nonce）和（msg’，nonce’）， msg≠msg' ,而 commit （msg, nonce） == commit （msg’，nonce’）。

为了使用承诺方案，我们首先需要产生一个临时随机数。然后将这个临时随机数与承诺信息 msg—起代入承诺函数，计算承诺函数输岀值 com，然后公布该输出。这个过程就如同将封好的信封放到一个人人能看到的桌上那样。之后， 如果我们希望展示之前的承诺值，我们首先公布用于产生承诺的临时随机数， 并公布信息 msg。此时任何人都可以验证这时公布的 msg 是否为之前承诺，这个阶段就如同打开信封。

> 对于每次的承诺值，你都需要选择新的随机值，这一点很重要。在密码学中，术语 nonce 是指，该取值只能使用一次。

以上两个安全特性决定了这一算法就如同密封及打开信封。第一，如果仅仅知道 com，即承诺函数的输出，就如同只看信封并不能得到信息内容；第二就是约束性，这就保证了你一旦承诺信封内的内容，就不能再改变主意。也就是 说，我们无法找到两个不同的信息，当你在承诺一个信息后，而又声称你承诺了另一个信息。

我们如何在承诺协议中保证隐秘性和约束性这两个性质成立呢？在讨论这 一点之前，我们需要讨论如何执行承诺方案。我们可以通过使用加密的哈希函 数来达到目的，考虑如下承诺协议实施方案：

commit ( msg, nonce) = H ( nonce || msg)

其中，nonce 为长度为 256 位的临时随机数。

为承诺一段消息，我们首先生成一个 256 位的临时随机数，然后将这个临时随机数与信息链接，并返回这个链接值的哈希值，来作为承诺输出。为了便于验证，我们还要设定其他人来计算一下临时随机数与信息链接之后的哈希值， 比对一下计算结果是否与承诺输出相同。

再来看一下我们的承诺方案要求的两个特性，如果我们将承诺和验证换成 H (nonce || msg)，那么这些特性就变成:

*   隐秘性：已知 H (nonce || msg)，没有可行方法找到 msg。

*   约束性：没有可行方法找到两对(msg， nonce )和(msg ‘ , nonce ’)， msg≠msg’ ,而 H (nonce || msg) == H (nonce’ || msg’)。

承诺的隐秘特性正是我们要求哈希函数要具备的隐秘性，如果将一个解密钥匙选定为 256 位的随机值，那么由隐秘性得出，如果解密钥匙与信息链接， 那么仅仅从哈希函数的输出中恢复信息就是不可行的。约束性隐含在哈希函数的碰撞阻力特性中，如果哈希函数具有碰撞阻力，那么我们将不能找到不同的 msg 及 msg’值,而 H ( nonce || msg) = H ( nonce’ || msg’)，如果这种情况发生， 将构成碰撞。

因此，如果哈希函数 H 具有碰撞阻力及隐秘性，从安全特性上来讲，这个承诺方案将有效。

> 结论反之不成立，就是说，我们"]*以找到碰撺，但都不是满足 H ( nonr：e || msg) = =H ( nonce || msg’)意义下的碰撞。例如，你可以对于同一个信息来产生满足同一承诺的随机数，但这里的哈希函数不具备碰撞阻力特性。

### 2.3 特性 3：谜题友好

哈希函数需要的第三个安全特性为谜题友好特性。这一特性较为复杂，我 们首先解释该特性的技术要求，然后通过举例来阐释该特性的意义。

直觉上，谜题友好可以这样解释，如果有一个人想找到 y 值所对应的输入，假定在输入集合中，有一部分是非常随机的，那么他将非常难以求得 y 值对应的输入。

**谜题友好**，如果对于任意 n 位输出值 y，假定 k 选自高阶最小熵分布，如果无法找到一个可行的方法，在比$ 2^{n} $小很多时间内找到 X，保证 H (k||x)= y 成立，那么我们称哈希函数 H 为谜题友好。

**应用：搜索谜题**

现在，让我们试想一个应用以阐释谜题友好特性的意义。在这个应用中， 我们将建立一个搜索谜题，该谜题是一个需要对庞大空间进行搜索，才能找到解决办法的数学问题。该搜索谜题没有捷径，也就是说除了搜索庞大的空间来 进行求解，别无他法。

搜索谜题构成：

*   一个哈希函数 H
*   从高阶最小熵分布选出的一个取值，id (我们称其为谜题 ID)。
*   目标集合 Y。

该谜题的解决方法为一个解，X，应该满足以下公式：

H (id || x) ∈Y

这个直觉是：如果 H 有一个 n 位输出，那么它的可能取值有$ 2^{n} $个。解决这个谜题要求找到一个位于集合 Y(通常比所有输出值集合小很多)内的输出值， Y 的大小决定了谜题的难度。如果 Y 是所有 n 位字符串的集合，这个谜题就毫无意义。然而，如果 Y 只有一个元素，那么这个谜题难度最大，谜题 ID 取自高阶最小熵分布，这个事实保证了求解无捷径。反过来，如果该 ID 的确定性很高，那么有人可能会作弊，比如通过使用该 ID，事先对谜题进行求解。

如果一个哈希函数具备谜题友好特性，这就意味着对于这个谜题没有一个解决策略，比只是随机地尝试 x 取值会更好。因此，如果我们要把谜题做成很难解决是可以的，只要我们能用适合的随机方式生成谜题 ID。当我们讨论比特币采矿(是一种搜索谜题)时会采用这一思路。

## 3 常用的 Hash 算法

`Hash`算法有很多种，本文中只介绍了项目中使用的 SHA256S 算法。

**安全散列算法 SHA：**

安全散列算法 SHA（Secure Hash Algorithm）是美国国家安全局 （NSA） 设计，美国国家标准与技术研究院（NIST） 发布的一系列密码散列函数，包括 SHA-1、SHA-224、SHA-256、SHA-384 和 SHA-512 等变体。主要适用于数字签名标准（DigitalSignature Standard DSS）里面定义的数字签名算法（Digital Signature Algorithm DSA）。SHA-1 已经不是那么安全了，google 和微软都已经弃用这个加密算法。为此，我们使用热门的比特币使用过的算法 SHA-256 作为实例。其它 SHA 算法，也可以按照这种模式进行使用。

> 著名的 Hash 算法：
> 
> 1.  MD4(RFC 1320)是 MIT 的 Ronald L. Rivest 在 1990 年设计的，MD 是 Message Digest 的缩写。它适用在 32 位字长的处理器上用快速软件实现–它是基于 32 位操作数的位操作来实现的。
> 2.  MD5(RFC 1321)是 Rivest 于 1991 年对 MD4 的改进版本号。它对输入仍以 512 位分组，其输出是 4 个 32 位字的级联，与 MD4 同样。MD5 比 MD4 来得复杂，而且速度较之要慢一点，但更安全，在抗分析和抗差分方面表现更好

## 4 总结

`hash`是一种算法， 是将文本转换成具有相同长度的，不可逆的杂凑字符串散列(很多资料也叫指纹或叫消息摘要`Message Digest`)。本身词义"切碎，剁碎"。也可以说，`hash`就是找到一种数据内容和数据存放地址之间的映射关系。

哈希函数被广泛用于检测数据的一致性。软件提供者常常在除了提供软件包以外，还会发布校验和。当下载完一个文件以后，你可以用哈希函数对下载好的文件计算一个哈希，并与作者提供的哈希进行比较，以此来保证文件下载的完整性。