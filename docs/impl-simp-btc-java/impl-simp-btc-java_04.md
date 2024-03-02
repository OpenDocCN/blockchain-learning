# 第四章 工作量证明(proof-of-work)

# 第四章 工作量证明(proof-of-work)

在上一节，我们构造了一个非常简单的数据结构 -- 区块，它也是整个区块链数据库的核心。目前所完成的区块链原型，已经可以通过链式关系把区块相互关联起来：每个块都与前一个块相关联。

但是，当前实现的区块链有一个巨大的缺陷：向链中加入区块太容易，也太廉价了。而区块链和比特币的其中一个核心就是，要想加入新的区块，必须先完成一些非常困难的工作。在本文，我们将会弥补这个缺陷。

## 1\. 课程目标

1.  了解什么是 PoW
2.  学会生成目标 hash
3.  学会在程序中通过代码实现 PoW

## 2\. 项目代码及效果展示

### 2.1 项目代码结构

![`img.kongyixueyuan.com/004_001_%E9%A1%B9%E7%9B%AE%E7%BB%93%E6%9E%84%E5%9B%BE.png`](img/800c7e43fe1bd0b6df347d66b2bed605.jpg)

### 2.2 项目运行结果

![`img.kongyixueyuan.com/004_002_%E8%BF%90%E8%A1%8C%E6%95%88%E6%9E%9C.gif`](img/568291e744781f7c8d9c9132f596bf15.jpg)

## 3\. 创建项目

### 3.1 创建工程

打开 IntelliJ IDEA 的工作空间，将上一个项目代码目录`part1_Basic_Prototype`，复制为`part2_Proof_Of_Work`。

然后打开 IntelliJ IDEA 开发工具。

打开工程：`part2_Proof_Of_Work`，并删除 target 目录。然后进行以下修改：

```java
step1：先将项目重新命名为：part2_Proof_Of_Work。
step2：修改 pom.xml 配置文件。
    改为：<artifactId>part2_Proof_Of_Work</artifactId>标签
    改为：<name>part2_Proof_Of_Work Maven Webapp</name> 
```

> 说明：我们每一章节的项目代码，都是在上一个章节上进行添加。所以拷贝上一次的项目代码，然后进行新内容的添加或修改。

![`img.kongyixueyuan.com/004_003_%E5%88%9B%E5%BB%BA%E9%A1%B9%E7%9B%AE.gif`](img/a9ad1429ae91864f38ebef5e6baa24ec.jpg)

### 3.2 代码实现

#### 3.2.1 创建`ProofOfWork.java`

打开`part2_Proof_Of_Work`目录里的`cldy.hanru.blockchain`包。新建一个包命名为：pow。并新建 java 文件，命名为`ProofOfWork.java`。

在`ProofOfWork.java`文件中编写代码如下：

```java
package cldy.hanru.blockchain.pow;

import java.math.BigInteger;

import org.apache.commons.codec.digest.DigestUtils;
import org.apache.commons.lang3.StringUtils;

import cldy.hanru.blockchain.block.Block;
import cldy.hanru.blockchain.util.ByteUtils;
import lombok.AllArgsConstructor;
import lombok.Data;

/**
 * 工作量证明
 * @author hanru
 *
 */
@Data
@AllArgsConstructor
public class ProofOfWork {

    /**
     * 难度目标位
     * 0000 0000 0000 0000 1001 0001 0000  .... 0001
     * 256 位 Hash 里面前面至少有 16 个零
     */
    public static final int TARGET_BITS = 16;

    /**
     * 要验证的区块
     */
    private Block block;

    /**
     * 难度目标值
     */
    private BigInteger target;

    /**
     * 创建新的工作量证明对象
     * 
     * 对 1 进行移位运算，将 1 向左移动 (256 - TARGET_BITS) 位，得到我们的难度目标值
     * @param block
     * @return
     */
    public static ProofOfWork newProofOfWork(Block block) {
        /*
        1.创建一个 BigInteger 的数值 1.
        0000000.....00001
        2.左移 256-bits 位

        以 8 bit 为例
        0000 0001
        0010 0000

        8-6 
         */

        BigInteger targetValue = BigInteger.ONE.shiftLeft((256 - TARGET_BITS));
        return new ProofOfWork(block, targetValue);
    }

    /**
     * 运行工作量证明，开始挖矿，找到小于难度目标值的 Hash
     * @return
     */
    public PowResult run() {
        long nonce = 0;
        String shaHex = "";
        System.out.printf("开始进行挖矿：%s \n", this.getBlock().getData());
        long startTime = System.currentTimeMillis();
        while (nonce < Long.MAX_VALUE) {
            byte[] data = this.prepareData(nonce);
            shaHex = DigestUtils.sha256Hex(data);
            System.out.printf("\r%d: %s",nonce,shaHex);
            if (new BigInteger(shaHex, 16).compareTo(this.target) == -1) {
                System.out.println();
                System.out.printf("耗时 Time: %s seconds \n", (float) (System.currentTimeMillis() - startTime) / 1000);
                System.out.printf("当前区块 Hash: %s \n\n", shaHex);
                break;
            } else {
                nonce++;
            }
        }
        return new PowResult(nonce, shaHex);
    }

    /**
     * 根据 block 的数据，以及 nonce，生成一个 byte 数组
     *
     * 注意：在准备区块数据时，一定要从原始数据类型转化为 byte[]，不能直接从字符串进行转换
     * @param nonce
     * @return
     */
    private byte[] prepareData(long nonce) {
        byte[] prevBlockHashBytes = {};
        if (StringUtils.isNoneBlank(this.getBlock().getPrevBlockHash())) {
            prevBlockHashBytes = new BigInteger(this.getBlock().getPrevBlockHash(), 16).toByteArray();
        }

        return ByteUtils.merge(
                prevBlockHashBytes,
                this.getBlock().getData().getBytes(),
                ByteUtils.toBytes(this.getBlock().getTimeStamp()),
                ByteUtils.toBytes(TARGET_BITS),
                ByteUtils.toBytes(nonce)
        );

    }

    /**
     * 验证区块是否有效
     *
     * @return
     */
    public boolean validate() {
        byte[] data = this.prepareData(this.getBlock().getNonce());
        return new BigInteger(DigestUtils.sha256Hex(data), 16).compareTo(this.target) == -1;
    }

} 
```

#### 3.2.2 创建`PowResult.java`文件

在`cldy.hanru.blockchain.pow`包下新建 java 文件，命名为`PowResult.java`。

在`PowResult.java`文件中编写代码如下：

```java
package cldy.hanru.blockchain.pow;

import lombok.AllArgsConstructor;
import lombok.Data;

/**
 * 工作量计算结果
 * @author ruby
 *
 */
@Data
@AllArgsConstructor
public class PowResult {

    /**
     * 计数器
     */
    private long nonce;
    /**
     * hash 值
     */
    private String hash;

} 
```

#### 3.2.3 修改`Block.java`

```java
修改步骤：
step1：修改 Block 类的属性
    添加 nonce 字段，用于表示计算产生目标 hash 的随机数
step2：修改 newBlock()
    删除 setHash()
    通过 ProofOfWork.java 中的 newProofOfWork()函数创建 pow 对象
    调用 run()方法，获取 powResult 对象，并进行设置 nonce 和 hash。 
```

修改完后代码如下：

```java
 package cldy.hanru.blockchain.block;

import java.time.Instant;

import org.apache.commons.codec.binary.Hex;

import cldy.hanru.blockchain.pow.PowResult;
import cldy.hanru.blockchain.pow.ProofOfWork;
import lombok.AllArgsConstructor;
import lombok.Data;
/**
 * 区块
 * @author hanru
 *
 */
@AllArgsConstructor
@Data
public class Block {

    /**
     * 区块 hash 值
     */
    private String hash;

    /**
     * 前一个区块的 hash 值
     */
    private String prevBlockHash;

    /**
     * 区块数据，未来会替换为交易
     */
    private String data;

    /**
     * 时间戳，单位秒
     */
    private long timeStamp;

    /**
     * 区块的高度
     */
    private long height;

    /**
     * 工作量证明计数器
     */
    private long nonce;

    /**
     * 创建新的区块
     * 
     * @param previousHash
     * @param data
     * @return
     */
    public static Block newBlock(String previousHash, String data,long height) {
        Block block = new Block("", previousHash, data, Instant.now().getEpochSecond(),height,0);
        ProofOfWork pow = ProofOfWork.newProofOfWork(block);
        PowResult powResult = pow.run();
        block.setHash(powResult.getHash());
        block.setNonce(powResult.getNonce());
//        block.setHash();
        return block;
    }

    /**
     * 设置 Hash
     * 注意：在准备区块数据时，一定要从原始数据类型转化为 byte[]，不能直接从字符串进行转换
     */
//    private void setHash() {
//        byte[] prevBlockHashBytes = {};
//        if (StringUtils.isNoneBlank(this.getPrevBlockHash())) {
//            prevBlockHashBytes = new BigInteger(this.getPrevBlockHash(), 16).toByteArray();
//        }
//
//        byte[] headers = ByteUtils.merge(prevBlockHashBytes, this.getData().getBytes(),
//                ByteUtils.toBytes(this.getTimeStamp()));
//
//        this.setHash(DigestUtils.sha256Hex(headers));
//    }

    private static final String ZERO_HASH = Hex.encodeHexString(new byte[32]);

    /**
     * 创建创世区块
     * @return
     */
    public static Block newGenesisBlock() {
        return Block.newBlock(ZERO_HASH, "Genesis Block",0);
    }

} 
```

#### 3.3.4 修改 main()主函数

在 main.java 中修改 main()方法，修改测试代码如下：

```java
package cldy.hanru.blockchain;

import java.math.BigInteger;
import java.text.SimpleDateFormat;
import java.util.Date;

import cldy.hanru.blockchain.block.Block;
import cldy.hanru.blockchain.block.Blockchain;
import cldy.hanru.blockchain.pow.ProofOfWork;
import org.apache.commons.codec.digest.DigestUtils;

/**
 * 测试
 *
 * @author hanru
 */
public class Main {

    public static void main(String[] args) {

//        // 1.创建创世区块
//        Block genesisBlock = Block.newGenesisBlock();
//        System.out.println("创世区块的信息：");
//        System.out.println("\thash:" + genesisBlock.getHash());
//        System.out.println("\tprevBlockHash:" + genesisBlock.getPrevBlockHash());
//        System.out.println("\tdata:" + genesisBlock.getData());
//        SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
//        String date = sdf.format(new Date(genesisBlock.getTimeStamp()*1000L));
//
//        System.out.println("\ttimeStamp:" + date);
//
//        //2.创建第二个区块
//        Block block2 = Block.newBlock(genesisBlock.getHash(), "I am hanru",1);
//        System.out.println("第二个区块的信息：");
//        System.out.println("\thash:" + block2.getHash());
//        System.out.println("\tprevBlockHash:" + block2.getPrevBlockHash());
//        System.out.println("\tdata:" + block2.getData());
//        String date2 = sdf.format(new Date(block2.getTimeStamp()*1000L));
//        System.out.println("\ttimeStamp:" + date2);

        //3.测试 Blockchain

        Blockchain blockchain = Blockchain.newBlockchain();

        System.out.println("创世链的信息：");
        System.out.println("区块的长度：" + blockchain.getBlockList().size());

        //4.添加区块
        blockchain.addBlock("Send 1 BTC to 韩茹");
        blockchain.addBlock("Send 2 more BTC to ruby");
        blockchain.addBlock("Send 4 more BTC to 王二狗");

        for (int i = 0; i < blockchain.getBlockList().size(); i++) {
            Block block = blockchain.getBlockList().get(i);
            System.out.println("第" + block.getHeight() + "个区块信息：");
            System.out.println("\tprevBlockHash: " + block.getPrevBlockHash());
            System.out.println("\tData: " + block.getData());
            System.out.println("\tHash: " + block.getHash());
            SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
            String date2 = sdf.format(new Date(block.getTimeStamp() * 1000L));
            System.out.println("\ttimeStamp:" + date2);

            ProofOfWork pow = ProofOfWork.newProofOfWork(block);
            System.out.println("是否有效: " + pow.validate() + "\n");
            System.out.println();
        }

/*
        // 5.检测 pow
        //1.创建一个 big 对象 0000000.....00001
        BigInteger target = BigInteger.ONE;

        System.out.printf("0x%x\n",target); //0x1

        //2.左移 256-bits 位
        target = target.shiftLeft((256 - ProofOfWork.TARGET_BITS));

        System.out.printf("0x%x\n",target); //61
        //61 位：0x1000000000000000000000000000000000000000000000000000000000000
        //64 位：0x0001000000000000000000000000000000000000000000000000000000000000

        //检测 hash
        System.out.println();
        String s1="HelloWorld";
        String hash = DigestUtils.sha256Hex(s1);
        System.out.printf("0x%s\n",hash);
*/
    }

} 
```

## 4\. 工作量证明讲解

### 4.1 什么是 PoW

区块链的一个关键点就是，一个人必须经过一系列困难的工作，才能将数据放入到区块链中。正是由于这种困难的工作，才保证了区块链的安全和一致。此外，完成这个工作的人，也会获得相应奖励（这也就是通过挖矿获得币）。

这个机制与生活现象非常类似：一个人必须通过努力工作，才能够获得回报或者奖励，用以支撑他们的生活。在区块链中，是通过网络中的参与者（矿工）不断的工作来支撑起了整个网络。矿工不断地向区块链中加入新块，然后获得相应的奖励。在这种机制的作用下，新生成的区块能够被安全地加入到区块链中，它维护了整个区块链数据库的稳定性。值得注意的是，完成了这个工作的人必须要证明这一点，即他必须要证明他的确完成了这些工作。

整个 “努力工作并进行证明” 的机制，就叫做工作量证明（proof-of-work）。要想完成工作非常地不容易，因为这需要大量的计算能力：即便是高性能计算机，也无法在短时间内快速完成。另外，这个工作的困难度会随着时间不断增长，以保持每 10 分钟出 1 个新块的速度。**在比特币中，这个工作就是找到一个块的哈希**，同时这个哈希满足了一些必要条件。这个哈希，也就充当了证明的角色。因此，寻求证明（寻找有效哈希），就是矿工实际要做的事情。

> 在区块链中，我们使用哈希算法保证一个块的一致性。哈希算法的输入数据包含了前一个块的哈希，因此使得不太可能（或者，至少很困难）去修改链中的一个块：因为如果一个人想要修改前面一个块的哈希，那么他必须要重新计算这个块以及后面所有块的哈希。

### 4.2 对比比特币

**Hashcash**

比特币使用 Hashcash ，一个最初用来防止垃圾邮件的工作量证明算法。它可以被分解为以下步骤：

1.  取一些公开的数据（比如，如果是 email 的话，它可以是接收者的邮件地址；在比特币中，它是区块头）
2.  给这个公开数据添加一个计数器。计数器默认从 0 开始
3.  将 **data(数据)** 和 **counter(计数器)** 组合到一起，获得一个哈希
4.  检查哈希是否符合一定的条件：

    A. 如果符合条件，结束

    B. 如果不符合，增加计数器，重复步骤 3-4

因此，这是一个暴力算法：改变计数器，计算新的哈希，检查，增加计数器，计算哈希，检查，如此往复。这也是为什么说它的计算成本很高，因为这一步需要如此反复不断地计算和检查。

现在，让我们来仔细看一下一个哈希要满足的必要条件。在原始的 Hashcash 实现中，它的要求是 “一个哈希的前 20 位必须是 0”。在比特币中，这个要求会随着时间而不断变化。因为按照设计，必须保证每 10 分钟生成一个块，而不论计算能力会随着时间增长，或者是会有越来越多的矿工进入网络，所以需要动态调整这个必要条件。

**比特币中目标 Hash 的计算**

1.  难度 Difficulty：

    整个网络会通过调整“难度”这个变量来控制生成工作量证明所需要的计算力。

    随着难度增加，矿工通常在循环遍历 4 亿次随机数值后仍未找到区块，则会启用超额随机数。

2.  难度目标 Bits：

    使整个网络的计算力大致每 10 分钟产生一个区块所需要的难度数值即为难度目标。

    Bits 是用来存储难度目标的 16 进制数值。

    举例计算：例如 516532 块：

    ```java
     A：Bits = "0x17502ab7"

     B：exponent 指数，exponent = 0x17

     C：coefficient 系数，coefficient = 0x502ab7

     D：target = coefficient * Math.Pow(2, 8 * (exponent - 3))

     E：目标 hash：000000000000000000502ab700000000d6420b16625d309c4561290000000000

     F：实际 hash：00000000000000000041ff1cfc5f15f929c1a45d262f88e4db83680d90658c0c 
    ```

3.  难度重定：

    全网中每新增 2016 个区块，全网难度将重新计算，该新难度值将依据前 2016 个区块的哈希算力而定。

    按照每 10 分钟产生一个区块的速度计算，每产生 2016 个区块大约 14 天，也就是两周。

    ![`img.kongyixueyuan.com/0304_block2.jpeg`](img/22af507c91e4f72012032b83866d2818.jpg)

### 4.3 代码实现

好了，了解完了比特币的理论层面，现在来设计我们的实现思路。因为比特币中有比特币社区维护 Bits 值，而我们的程序中为了方便实现，所以简化了实现过程。

#### 4.3.1 生成目标 hash

首先，定义挖矿的难度值。

在`ProofOfWork`类中，添加静态常量，表示难度系数(256 位 hash 值前有多少个 0)：

```java
public static final int TARGET_BITS = 16; 
```

当一个块被挖出来以后，“TARGET_BITS” 代表了区块头里存储的难度，也就是开头有多少个 0。这里的 16 指的是算出来的哈希前 16 位必须是 0，如果用 16 进制表示，就是前 4 位必须是 0，这一点从最后的输出可以看出来。目前我们并不会实现一个动态调整目标的算法，所以将难度定义为一个全局的常量即可。

16 其实是一个可以任意取的数字，其目的只是为了有一个目标（target）而已，这个目标占据不到 256 位的内存空间。同时，我们想要有足够的差异性，但是又不至于大的过分，因为差异性越大，就越难找到一个合适的哈希。

在`ProofOfWork.java`中，定义`ProofOfWork`类：

```java
public class ProofOfWork {

    /**
     * 难度目标位
     * 0000 0000 0000 0000 1001 0001 0000  .... 0001
     * 256 位 Hash 里面前面至少有 16 个零
     */
    public static final int TARGET_BITS = 16;

    /**
     * 要验证的区块
     */
    private Block block;

    /**
     * 难度目标值
     */
    private BigInteger target;
} 
```

这里，我们构造了 **ProofOfWork** 类，里面存储了指向一个块(`block`)和一个目标(`target`)的指针。这里的 “目标” ，也就是前一节中所描述的必要条件。这里使用了一个大整数 ，我们会将哈希与目标进行比较：先把哈希转换成一个大整数，然后检测它是否小于目标。

```java
 /**
     * 创建新的工作量证明对象
     * 
     * 对 1 进行移位运算，将 1 向左移动 (256 - TARGET_BITS) 位，得到我们的难度目标值
     * @param block
     * @return
     */
    public static ProofOfWork newProofOfWork(Block block) {
        /*
        1.创建一个 BigInteger 的数值 1.
        0000000.....00001
        2.左移 256-bits 位

        以 8 bit 为例
        0000 0001
        0010 0000

        8-6 
         */

        BigInteger targetValue = BigInteger.ONE.shiftLeft((256 - TARGET_BITS));
        return new ProofOfWork(block, targetValue);
    } 
```

在 `newProofOfWork()` 函数中，我们将 `BigInteger` 初始化为 1，然后左移 `256 - TARGET_BITS` 位。**256** 是一个 SHA-256 哈希的位数，我们将要使用的是 SHA-256 哈希算法。**target（目标）** 的 16 进制形式为：

```java
目标 hash：256bit。转为 16 进制是 64 个长度，因为 1 前面都是 0，所以只打印出 61 个长度
0x1000000000000000000000000000000000000000000000000000000000000
但实际上是：前面有 3 个 0
0x0001000000000000000000000000000000000000000000000000000000000000 
```

接下来，我们构建`PowResult`类，里面声明两个属性，用于表示随机数和生成的目标 hash，代码如下：

```java
package cldy.hanru.blockchain.pow;

import lombok.AllArgsConstructor;
import lombok.Data;

/**
 * 工作量计算结果
 * @author ruby
 *
 */
@Data
@AllArgsConstructor
public class PowResult {

    /**
     * 计数器
     */
    private long nonce;
    /**
     * hash 值
     */
    private String hash;

} 
```

现在，我们在 main 中编写一段测试代码。

在`main.java`中：

```java
package cldy.hanru.blockchain;

import java.math.BigInteger;
import java.text.SimpleDateFormat;
import java.util.Date;

import cldy.hanru.blockchain.block.Block;
import cldy.hanru.blockchain.block.Blockchain;
import cldy.hanru.blockchain.pow.ProofOfWork;
import org.apache.commons.codec.digest.DigestUtils;

/**
 * 测试
 * 
 * @author hanru
 *
 */
public class Main {

     // 5.检测 pow
        //1.创建一个 big 对象 0000000.....00001
        BigInteger target = BigInteger.ONE;

        System.out.printf("0x%x\n",target); //0x1

        //2.左移 256-bits 位
        target = target.shiftLeft((256 - ProofOfWork.TARGET_BITS));

        System.out.printf("0x%x\n",target); //61
        //61 位：0x1000000000000000000000000000000000000000000000000000000000000
        //64 位：0x0001000000000000000000000000000000000000000000000000000000000000

        //检测 hash
        System.out.println();
        String s1="HelloWorld";
        String hash = DigestUtils.sha256Hex(s1);
        System.out.printf("0x%s\n",hash);
} 
```

运行结果：

![`img.kongyixueyuan.com/004_004_%E6%B5%8B%E8%AF%95pow.png`](img/a78de5eb5bc31b18077f65837ec3c82d.jpg)

通过运行结果可以看出，基于"HelloWorld"算出来的 hash 值(0x872e4e50ce9990d8b041330c47c9ddd11bec6b503ae9386a99da8584e9bb12c4),要比目标 hash 大，因此它并不是一个有效的工作量证明。只有比目标 hash 小，才是一个有效的证明。

hash 比较：

```java
目标 hash：
0x0001000000000000000000000000000000000000000000000000000000000000

基于"HelloWorld"生成的 hash 值：
0x872e4e50ce9990d8b041330c47c9ddd11bec6b503ae9386a99da8584e9bb12c4

比目标 hash 大，无效。
形如下面的 hash 值，比目标 hash 小，所以有效 hash：
0x00008b0f41ec78bab747864db66bcb9fb89920ee75f43fdaaeb5544f7f76ca23 
```

> 因为目标 hash 为 0x0001 0000 0000 ... 0000(前面有 3 个 0)，要想比目标 hash 小，那么我们生成的 hash 就要前面至少 4 个 0。

#### 4.3.2 计算实际 hash

你可以把目标想象为一个范围的上界：如果一个数（由哈希转换而来）比上界要小，那么是有效的，反之无效。因为要求比上界要小，所以会导致有效数字并不会很多。因此，也就需要通过一些困难的工作（一系列反复地计算），才能找到一个有效的数字。

现在，我们需要有数据来进行哈希，准备数据。

在`ProofOfWork.java`中：

```java
/**
     * 根据 block 的数据，以及 nonce，生成一个 byte 数组
     *
     * 注意：在准备区块数据时，一定要从原始数据类型转化为 byte[]，不能直接从字符串进行转换
     * @param nonce
     * @return
     */
    private byte[] prepareData(long nonce) {
        byte[] prevBlockHashBytes = {};
        if (StringUtils.isNoneBlank(this.getBlock().getPrevBlockHash())) {
            prevBlockHashBytes = new BigInteger(this.getBlock().getPrevBlockHash(), 16).toByteArray();
        }

        return ByteUtils.merge(
                prevBlockHashBytes,
                this.getBlock().getData().getBytes(),
                ByteUtils.toBytes(this.getBlock().getTimeStamp()),
                ByteUtils.toBytes(TARGET_BITS),
                ByteUtils.toBytes(nonce)
        );

    } 
```

这个部分比较直观：只需要将 `target` ，`nonce` 与 `block` 进行合并。这里的 `nonce`，就是上面 Hashcash 所提到的计数器，它是一个密码学术语。

很好，到这里，所有的准备工作就完成了，下面来实现 PoW 算法的核心。

在`ProofOfWork.java`中，定义一个`run()`方法，用于产生有效 hash，以及 nonce 值：

```java
/**
     * 运行工作量证明，开始挖矿，找到小于难度目标值的 Hash
     * @return
     */
    public PowResult run() {
        long nonce = 0;
        String shaHex = "";
        System.out.printf("开始进行挖矿：%s \n", this.getBlock().getData());
        long startTime = System.currentTimeMillis();
        while (nonce < Long.MAX_VALUE) {
            byte[] data = this.prepareData(nonce);
            shaHex = DigestUtils.sha256Hex(data);
            System.out.printf("\r%d: %s",nonce,shaHex);
            if (new BigInteger(shaHex, 16).compareTo(this.target) == -1) {
                System.out.println();
                System.out.printf("耗时 Time: %s seconds \n", (float) (System.currentTimeMillis() - startTime) / 1000);
                System.out.printf("当前区块 Hash: %s \n\n", shaHex);
                break;
            } else {
                nonce++;
            }
        }
        return new PowResult(nonce, shaHex);
    } 
```

首先我们对变量进行初始化：

*   `nonce` 是计数器。

然后开始一个 “无限” 循环，在这个循环中，我们做的事情有：

1.  准备数据
2.  用 SHA-256 对数据进行哈希
3.  将哈希转换成一个大整数
4.  将这个大整数与目标进行比较

跟之前所讲的一样简单。现在我们可以移除 `block` 的 `setHash` 方法，然后修改 `newBlock()` 函数：

在`Block.java`中，修改`newBlock()`函数：

```java
 /**
     * 创建新的区块
     * 
     * @param previousHash
     * @param data
     * @return
     */
    public static Block newBlock(String previousHash, String data,long height) {
        Block block = new Block("", previousHash, data, Instant.now().getEpochSecond(),height,0);
        ProofOfWork pow = ProofOfWork.newProofOfWork(block);
        PowResult powResult = pow.run();
        block.setHash(powResult.getHash());
        block.setNonce(powResult.getNonce());
//        block.setHash();
        return block;
    } 
```

在这里，你可以看到 `nonce` 被保存为 `Block` 的一个属性。这是十分有必要的，因为待会儿我们对这个工作量进行验证时会用到 `nonce` 。`Block` 类现在看起来像是这样：

```java
public class Block {

    /**
     * 区块 hash 值
     */
    private String hash;

    /**
     * 前一个区块的 hash 值
     */
    private String prevBlockHash;

    /**
     * 区块数据，未来会替换为交易
     */
    private String data;

    /**
     * 时间戳，单位秒
     */
    private long timeStamp;

    /**
     * 区块的高度
     */
    private long height;

    /**
     * 工作量证明计数器
     */
    private long nonce;

} 
```

好了！现在让我们来运行一下是否正常工作：

在`main.java`中，进行测试：

```java
package cldy.hanru.blockchain;

import java.math.BigInteger;
import java.text.SimpleDateFormat;
import java.util.Date;

import cldy.hanru.blockchain.block.Block;
import cldy.hanru.blockchain.block.Blockchain;
import cldy.hanru.blockchain.pow.ProofOfWork;
import org.apache.commons.codec.digest.DigestUtils;

/**
 * 测试
 * 
 * @author hanru
 *
 */
public class Main {

    public static void main(String[] args) {

//        // 1.创建创世区块
//        Block genesisBlock = Block.newGenesisBlock();
//        System.out.println("创世区块的信息：");
//        System.out.println("\thash:" + genesisBlock.getHash());
//        System.out.println("\tprevBlockHash:" + genesisBlock.getPrevBlockHash());
//        System.out.println("\tdata:" + genesisBlock.getData());
//        SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
//        String date = sdf.format(new Date(genesisBlock.getTimeStamp()*1000L));
//
//        System.out.println("\ttimeStamp:" + date);
//
//        //2.创建第二个区块
//        Block block2 = Block.newBlock(genesisBlock.getHash(), "I am hanru",1);
//        System.out.println("第二个区块的信息：");
//        System.out.println("\thash:" + block2.getHash());
//        System.out.println("\tprevBlockHash:" + block2.getPrevBlockHash());
//        System.out.println("\tdata:" + block2.getData());
//        String date2 = sdf.format(new Date(block2.getTimeStamp()*1000L));
//        System.out.println("\ttimeStamp:" + date2);

        //3.测试 Blockchain

        Blockchain blockchain = Blockchain.newBlockchain();

        System.out.println("创世链的信息：");
        System.out.println("区块的长度："+blockchain.getBlockList().size());

        //4.添加区块
        blockchain.addBlock("Send 1 BTC to 韩茹");
        blockchain.addBlock("Send 2 more BTC to ruby");
        blockchain.addBlock("Send 4 more BTC to 王二狗");

        for(int i=0;i<blockchain.getBlockList().size();i++) {
            Block block = blockchain.getBlockList().get(i);
            System.out.println("第"+block.getHeight()+"个区块信息：");
            System.out.println("\tprevBlockHash: " + block.getPrevBlockHash());
            System.out.println("\tData: " + block.getData());
            System.out.println("\tHash: " + block.getHash());
            SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
            String date2 = sdf.format(new Date(block.getTimeStamp()*1000L));
            System.out.println("\ttimeStamp:" + date2);
            System.out.println();
        }

/*
        // 5.检测 pow
        //1.创建一个 big 对象 0000000.....00001
        BigInteger target = BigInteger.ONE;

        System.out.printf("0x%x\n",target); //0x1

        //2.左移 256-bits 位
        target = target.shiftLeft((256 - ProofOfWork.TARGET_BITS));

        System.out.printf("0x%x\n",target); //61
        //61 位：0x1000000000000000000000000000000000000000000000000000000000000
        //64 位：0x0001000000000000000000000000000000000000000000000000000000000000

        //检测 hash
        System.out.println();
        String s1="HelloWorld";
        String hash = DigestUtils.sha256Hex(s1);
        System.out.printf("0x%s\n",hash);
*/
    }

} 
```

运行结果：

![`img.kongyixueyuan.com/004_005_%E8%BF%90%E8%A1%8C%E7%BB%93%E6%9E%9C.png`](img/a79d48f203bec53019fa2a5069a80bce.jpg)

在`main`中，我们创建一个`BlockChain`对象，并添加了 3 个区块，每个区块都需要不同的计算`hash`值，这需要一定的时间，直到小于目标`hash`为止，才算挖矿成功。

#### 4.3.3 验证

还剩下一件事情需要做，对工作量证明进行验证：

在`ProofOfWork.java`中，添加验证方法。

```java
 /**
     * 验证区块是否有效
     *
     * @return
     */
    public boolean validate() {
        byte[] data = this.prepareData(this.getBlock().getNonce());
        return new BigInteger(DigestUtils.sha256Hex(data), 16).compareTo(this.target) == -1;
    } 
```

这里，就是我们就用到了上面保存的 `nonce`。

再来检测一次是否正常工作：

```java
public static void main(String[] args) {
    ...

    for (int i = 0; i < blockchain.getBlockList().size(); i++) {
            Block block = blockchain.getBlockList().get(i);
            System.out.println("第" + block.getHeight() + "个区块信息：");
            System.out.println("\tprevBlockHash: " + block.getPrevBlockHash());
            System.out.println("\tData: " + block.getData());
            System.out.println("\tHash: " + block.getHash());
            SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
            String date2 = sdf.format(new Date(block.getTimeStamp() * 1000L));
            System.out.println("\ttimeStamp:" + date2);

            ProofOfWork pow = ProofOfWork.newProofOfWork(block);
            System.out.println("是否有效: " + pow.validate() + "\n");
            System.out.println();
        }
} 
```

运行结果：

![`img.kongyixueyuan.com/004_006_%E8%BF%90%E8%A1%8C%E7%BB%93%E6%9E%9C2.png`](img/676e930151f7b15c9ccabd770cd6348b.jpg)

从上图可以看出，这次我们产生四个块花费了一分多钟，比没有工作量证明之前慢了很多（也就是成本高了很多）

## 5\. 总结

通过本章节的学习，我们了解了挖矿的原理，挖矿的过程就是重复计算区块头的 hash，不断修改随机数 Nonce，直到小于难度目标 Bits 计算出来的 hash。挖矿是比特币共识机制中的 PoW 算法（Proof of Work，工作量证明机制）。经济学上认为，理性的人都是逐利的，PoW 抑制了节点的恶意动机。

我们离真正的区块链又进了一步：现在需要经过一些困难的工作才能加入新的块，因此挖矿就有可能了。但是，它仍然缺少一些至关重要的特性：区块链数据库并不是持久化的，没有钱包，地址，交易，也没有共识机制。不过，所有的这些，我们都会在接下来的文章中实现，现在，愉快地挖矿吧！

[项目源代码](https://github.com/rubyhan1314/BitcoinForJava)