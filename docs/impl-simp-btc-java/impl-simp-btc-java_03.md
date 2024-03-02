# 第三章 牛刀小试-创建最简单的区块链

# 第三章 牛刀小试-创建最简单的区块链

# 基本原型(basic-prototype)

## 1\. 课程目标

1.  了解区块链的结构
2.  学会创建一个区块(Block)
3.  学会创建区块链(BlockChain)
4.  学会向一个区块链上添加新的区块

## 2\. 项目代码及效果展示

### 2.1 项目代码结构

![`img.kongyixueyuan.com/0301_%E9%A1%B9%E7%9B%AE%E7%BB%93%E6%9E%84%E5%9B%BE.png`](img/ff96c1106d17b217e14ab1ea26608926.jpg)

### 2.2 项目运行结果

#### 2.2.1 创建区块

![`img.kongyixueyuan.com/0302_%E8%BF%90%E8%A1%8C%E7%BB%93%E6%9E%9C1.png`](img/85aa31197e8203ca64aee3ca227b73b8.jpg)

#### 2.2.2 创建区块链效果

![`img.kongyixueyuan.com/0304_%E8%BF%90%E8%A1%8C%E7%BB%93%E6%9E%9C2.png`](img/36227b2c1280b50ffe16e9d7b0d5b08e.jpg)

#### 2.2.3 添加区块效果

![`img.kongyixueyuan.com/0304_%E8%BF%90%E8%A1%8C%E7%BB%93%E6%9E%9C3.png`](img/1823ba33898bee1e995276610823488a.jpg)

## 3\. 创建项目

### 3.1 创建工程

首先打开 IntelliJ IDEA 开发工具

新建工程：`part1_Basic_Prototype`

在`pom.xml`配置文件中注释掉暂时不用的配置信息：

![`img.kongyixueyuan.com/0306_%E5%88%9B%E5%BB%BA%E9%A1%B9%E7%9B%AE.gif`](img/826bd40b3e9c888118dc3f3d2bee90aa.jpg)

### 3.2 代码实现

#### 3.2.1 创建`Block.java`

在`cldy.hanru.blockchain.block`包下，创建一个文件`Block.java`。

在`Block.java`文件中编写代码如下：

```java
package cldy.hanru.blockchain.block;

import java.math.BigInteger;
import java.time.Instant;

import org.apache.commons.codec.binary.Hex;
import org.apache.commons.codec.digest.DigestUtils;
import org.apache.commons.lang3.StringUtils;

import lombok.AllArgsConstructor;
import lombok.Data;

import cldy.hanru.blockchain.util.ByteUtils;
/**
 * 
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
     * 创建新的区块
     * 
     * @param previousHash
     * @param data
     * @return
     */
    public static Block newBlock(String previousHash, String data,long height) {
        Block block = new Block("", previousHash, data, Instant.now().getEpochSecond(),height);
        block.setHash();
        return block;
    }

    /**
     * 设置 Hash
     * 注意：在准备区块数据时，一定要从原始数据类型转化为 byte[]，不能直接从字符串进行转换
     */
    private void setHash() {
        byte[] prevBlockHashBytes = {};
        if (StringUtils.isNoneBlank(this.getPrevBlockHash())) {
            prevBlockHashBytes = new BigInteger(this.getPrevBlockHash(), 16).toByteArray();
        }

        byte[] headers = ByteUtils.merge(prevBlockHashBytes, this.getData().getBytes(),
                ByteUtils.toBytes(this.getTimeStamp()));

        this.setHash(DigestUtils.sha256Hex(headers));
    }

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

#### 3.2.2 创建`ByteUtils.java`

在`src/main/java`目录下，先新建一个`cldy.hanru.blockchain.util`包，在该包下创建一个 java 文件，命名为：`ByteUtils.java`。里面编写所需要的工具方法。

编写代码如下：

```java
package cldy.hanru.blockchain.util;

import java.nio.ByteBuffer;
import java.util.Arrays;
import java.util.stream.Stream;

import org.apache.commons.lang3.ArrayUtils;

/**
 * 字节数组工具类
 * @author hanru
 *
 */
public class ByteUtils {
    /**
     * 将多个字节数组合并成一个字节数组
     * @param bytes
     * @return
     */
    public static byte[] merge(byte[]... bytes) {
        Stream<Byte> stream = Stream.of();
        for (byte[] b : bytes) {
            stream = Stream.concat(stream, Arrays.stream(ArrayUtils.toObject(b)));
        }
        return ArrayUtils.toPrimitive(stream.toArray(Byte[]::new));
    }

    /**
     * long 转化为 byte[]
     * @param val
     * @return
     */
    public static byte[] toBytes(long val) {
        return ByteBuffer.allocate(Long.BYTES).putLong(val).array();
    }
} 
```

#### 3.2.3 创建`Blockchain.java`

在`cldy.hanru.blockchain.block`包下，创建一个文件`Blockchain.java`。

在`Blockchain.java`文件中编写代码如下：

```java
package cldy.hanru.blockchain.block;

import java.util.LinkedList;
import java.util.List;

import lombok.AllArgsConstructor;
import lombok.Data;

/**
 * 区块链
 * @author hanru
 *
 */
@Data
@AllArgsConstructor
public class Blockchain {

    /**
     * 存储区块的集合
     */
    private List<Block> blockList;

    /**
     * 创建区块链
     * @return
     */
    public static Blockchain newBlockchain() {
        List<Block> blocks = new LinkedList<>();
        blocks.add(Block.newGenesisBlock());
        return new Blockchain(blocks);
    }

    /**
     * 根据 block，添加区块
     * @param block
     */
    public void addBlock(Block block) {
        this.blockList.add(block);
    }

    /**
     * 根据 data 添加区块
     * @param data
     */
    public void addBlock(String data) {
        Block previousBlock = blockList.get(blockList.size() - 1);
        this.addBlock(Block.newBlock(previousBlock.getHash(), data,previousBlock.getHeight()+1));
    }

} 
```

#### 3.2.4 创建`main.java`

在`src/main/java`目录下，先新建一个`cldy.hanru.blockchain`包，在该包下创建一个 java 文件，命名为：`Main.java`。里面编写所需要的工具方法。

编写代码如下：

```java
package cldy.hanru.blockchain;

import java.text.SimpleDateFormat;
import java.util.Date;

import cldy.hanru.blockchain.block.Block;
import cldy.hanru.blockchain.block.Blockchain;

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
    }

} 
```

#### 3.2.5 修改配置文件`pow.xml`

在`src/main/java`目录下，修改 Maven 的配置文件：pow.xml。添加项目所需要的依赖包。

编写代码如下：

```java
<?xml version="1.0" encoding="UTF-8"?>

<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>cldy.hanru.blockchain</groupId>
    <artifactId>part1_Basic_Prototype</artifactId>
    <version>1.0-SNAPSHOT</version>
    <packaging>jar</packaging>

    <name>part1_Basic_Prototype Maven Webapp</name>
    <!-- FIXME change it to the project's website -->
    <url>http://www.example.com</url>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <maven.compiler.source>1.8</maven.compiler.source>
        <maven.compiler.target>1.8</maven.compiler.target>
    </properties>

    <dependencies>
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.11</version>
            <scope>test</scope>
        </dependency>

        <!-- lombok 是一个可以通过简单的注解的形式来帮助我们简化消除一些必须有但显得很臃肿的 Java 代码的工具 -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>1.16.20</version>
        </dependency>

        <!--处理 Java 基本对象方法的工具类包，该类包提供对字符、数组等基本对象的操作，弥补了 java.lang api 基本处理方法上的不足。 -->
        <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-lang3</artifactId>
            <version>3.7</version>
        </dependency>

        <!-- commons-codec 是 Apache 开源组织提供的用于摘要运算、编码的包。在该包中主要分为四类加密：BinaryEncoders、DigestEncoders、LanguageEncoders、NetworkEncoders。 -->
        <dependency>
            <groupId>commons-codec</groupId>
            <artifactId>commons-codec</artifactId>
            <version>1.11</version>
        </dependency>

    </dependencies>

<!--
    <build>
        <finalName>part1_Basic_Prototype</finalName>
        <pluginManagement>
            <plugins>
                <plugin>
                    <artifactId>maven-clean-plugin</artifactId>
                    <version>3.0.0</version>
                </plugin>
                <plugin>
                    <artifactId>maven-resources-plugin</artifactId>
                    <version>3.0.2</version>
                </plugin>
                <plugin>
                    <artifactId>maven-compiler-plugin</artifactId>
                    <version>3.7.0</version>
                </plugin>
                <plugin>
                    <artifactId>maven-surefire-plugin</artifactId>
                    <version>2.20.1</version>
                </plugin>
                <plugin>
                    <artifactId>maven-war-plugin</artifactId>
                    <version>3.2.0</version>
                </plugin>
                <plugin>
                    <artifactId>maven-install-plugin</artifactId>
                    <version>2.5.2</version>
                </plugin>
                <plugin>
                    <artifactId>maven-deploy-plugin</artifactId>
                    <version>2.8.2</version>
                </plugin>

            </plugins>
        </pluginManagement>
    </build>

    -->

</project> 
```

## 4\. 项目基本原型讲解

### 4.1\. 区块 Block

首先从 “区块” 谈起。在区块链中，真正存储有效信息的是区块（block）。而在比特币中，真正有价值的信息就是交易（transaction）。实际上，交易信息是所有加密货币的价值所在。除此以外，区块还包含了一些技术实现的相关信息，比如版本，当前时间戳和前一个区块的哈希。

#### 4.1.1 区块结构

不过，我们要实现的是一个简化版的区块链，而不是一个像比特币技术规范所描述那样成熟完备的区块链。所以在我们目前的实现中，区块仅包含了部分关键信息，它的数据结构如下(在`Block.java`文件中)：

在 cldy.hanru.blockchain.block 包下，创建`Block.java`文件，并添加`Block`类，字段如下：

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
} 
```

说明一下：

| 字段 | 解释 |
| --- | --- |
| `Height` | 当前区块的高度，就是区块的编号 |
| `PrevBlockHash` | 前一个块的哈希，即父哈希 |
| `Data` | 区块存储的实际有效信息，也就是交易。随着项目的逐步完善后期会将 Data 修改为 Transaction。 |
| `Timestamp` | 当前时间戳，也就是区块创建的时间 |
| `Hash` | 当前块的哈希 |

> 什么是创世区块？
> 
> ```java
> 区块链中的第一个区块，也就是高度 Heigth 为 0 的区块，往往叫做创世区块。 
> ```

#### 4.1.2 对比比特币

我们这里的 `Timestamp`，`PrevBlockHash`, `Hash`，在比特币技术规范中属于区块头（`block header`），区块头是一个单独的数据结构。

![`img.kongyixueyuan.com/08_block.png`](img/f977d41d475dddeaf022091c209169c0.jpg)

完整的比特币的区块头（`block header`）结构如下：

| Field | Purpose | Updated when... | Size (Bytes) |
| --- | --- | --- | --- |
| Version | Block version number | You upgrade the software and it specifies a new version | 4 |
| hashPrevBlock | 256-bit hash of the previous block header | A new block comes in | 32 |
| hashMerkleRoot | 256-bit hash based on all of the transactions in the block | A transaction is accepted | 32 |
| Time | Current timestamp as seconds since 1970-01-01T00:00 UTC | Every few seconds | 4 |
| Bits | Current target in compact format | The difficulty is adjusted | 4 |
| Nonce | 32-bit number (starts at 0) | A hash is tried (increments) | 4 |

而我们的 `Data`, 在比特币中对应的是交易，是另一个单独的数据结构。为了简便起见，目前将这两个数据结构放在了一起。在真正的比特币中，区块的数据结构如下：

| Field | Description | Size |
| --- | --- | --- |
| Magic no | value always 0xD9B4BEF9 | 4 bytes |
| Blocksize | number of bytes following up to end of block | 4 bytes |
| Blockheader | consists of 6 items | 80 bytes |
| Transaction counter | positive integer VI = VarInt | 1 - 9 bytes |
| transactions | the (non empty) list of transactions | <transaction counter="">-many transactions</transaction> |

![`om1c35wrq.bkt.clouddn.com/09_blockchain.png`](img/75c556c1b2bf96b3c20d5b55809e529c.jpg)

```java
 区块链的结构 
```

### 4.2 计算 Hash

在我们的简化版区块中，还有一个 `Hash` 字段，那么，要如何计算哈希呢？哈希计算，是区块链一个非常重要的部分。正是由于它，才保证了区块链的安全。计算一个哈希，是在计算上非常困难的一个操作。即使在高速电脑上，也要耗费很多时间 (这就是为什么人们会购买 GPU，FPGA，ASIC 来挖比特币) 。这是一个架构上有意为之的设计，它故意使得加入新的区块十分困难，继而保证区块一旦被加入以后，就很难再进行修改。在接下来的内容中，我们将会讨论和实现这个机制。

**拼接字段**

目前，我们仅取了 `Block` 类的部分字段（`timeStamp`, `data` 和 `prevBlockHash`），并将它们相互拼接起来，然后在拼接后的结果上计算一个 SHA-256，然后就得到了哈希.

因为`TimeStamp`是一个整型数据，我们需要将它转为`[]byte`

`ByteUtils.java`文件中：

```java
/**
 * long 转化为 byte[]
 * @param val
 * @return
 */
public static byte[] toBytes(long val) {
    return ByteBuffer.allocate(Long.BYTES).putLong(val).array();
} 
```

拼接多个字节数组：

```java
/**
     * 将多个字节数组合并成一个字节数组
     * @param bytes
     * @return
     */
    public static byte[] merge(byte[]... bytes) {
        Stream<Byte> stream = Stream.of();
        for (byte[] b : bytes) {
            stream = Stream.concat(stream, Arrays.stream(ArrayUtils.toObject(b)));
        }
        return ArrayUtils.toPrimitive(stream.toArray(Byte[]::new));
    } 
```

**设置 Hash**

在 `SetHash` 方法中完成这些操作：

`Block.java`文件中：

```java
/**
     * 设置 Hash
     * 注意：在准备区块数据时，一定要从原始数据类型转化为 byte[]，不能直接从字符串进行转换
     */
    private void setHash() {
        byte[] prevBlockHashBytes = {};
        if (StringUtils.isNoneBlank(this.getPrevBlockHash())) {
            prevBlockHashBytes = new BigInteger(this.getPrevBlockHash(), 16).toByteArray();
        }

        byte[] headers = ByteUtils.merge(prevBlockHashBytes, this.getData().getBytes(),
                ByteUtils.toBytes(this.getTimeStamp()));

        this.setHash(DigestUtils.sha256Hex(headers));
    } 
```

因为方法中使用了 Block 类中的私有属性的 get 和 set 方法，下面的代码中用到了该类的构造函数，所以我们应该在类中添加类的构造函数以及 get 和 set 方法。也可以使用第三方包：lombok。[安装详情：](https://www.cnblogs.com/parryyang/p/8400636.html)

安装后需要在 Maven 的配置文件 pow.xml 引入依赖包：

```java
<dependencies>
        <!-- lombok 是一个可以通过简单的注解的形式来帮助我们简化消除一些必须有但显得很臃肿的 Java 代码的工具 -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>1.16.20</version>
        </dependency>
</dependencies> 
```

然后在 Block 类的定义上添加注解：

```java
/**
 * 区块
 * @author hanru
 *
 */
@AllArgsConstructor
@Data
public class Block {
    ...
} 
```

另外，在`setHash()`中，需要用到`StringUtils`类，所以需要导入`org.apache.commons.lang3`包。所以继续修改`pow.xml`文件，添加依赖包：

```java
<!--处理 Java 基本对象方法的工具类包，该类包提供对字符、数组等基本对象的操作，弥补了 java.lang api 基本处理方法上的不足。 -->
        <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-lang3</artifactId>
            <version>3.7</version>
        </dependency> 
```

最后，在`setHash()`中，需要导入`commons-codec`包。进行 hash 计算。所以继续修改`pow.xml`文件，添加依赖包：

```java
<!-- commons-codec 是 Apache 开源组织提供的用于摘要运算、编码的包。在该包中主要分为四类加密：BinaryEncoders、DigestEncoders、LanguageEncoders、NetworkEncoders。 -->
        <dependency>
            <groupId>commons-codec</groupId>
            <artifactId>commons-codec</artifactId>
            <version>1.11</version>
        </dependency> 
```

### 4.3 创建区块

#### 4.3.1 创建一个新的区块

接下来，我们会实现一个用于简化创建区块的方法 `newBlock()`：

`Block.java`文件中：

```java
/**
     * 创建新的区块
     * 
     * @param previousHash
     * @param data
     * @return
     */
    public static Block newBlock(String previousHash, String data,long height) {
        Block block = new Block("", previousHash, data, Instant.now().getEpochSecond(),height);
        block.setHash();
        return block;
    } 
```

#### 4.3.2 创建一个创世区块

创世区块因为是区块链中的第一个区块，还是有一些特殊的，比如`prevBlockHash`固定为 0。

```java
/**
     * 创建创世区块
     * @return
     */
    public static Block newGenesisBlock() {
        return Block.newBlock(ZERO_HASH, "Genesis Block",0);
    } 
```

因为创世区块的`prevBlockHash`字段固定为 0，所以我们在 Block 类中添加一个静态常量：

```java
private static final String ZERO_HASH = Hex.encodeHexString(new byte[32]); 
```

在 main 函数中进行测试：

```java
package cldy.hanru.blockchain;

import java.text.SimpleDateFormat;
import java.util.Date;

import cldy.hanru.blockchain.block.Block;
import cldy.hanru.blockchain.block.Blockchain;

/**
 * 测试
 * 
 * @author hanru
 *
 */
public class Main {
    public static void main(String[] args) {
        // 1.创建创世区块
        Block genesisBlock = Block.newGenesisBlock();
        System.out.println("创世区块的信息：");
        System.out.println("\thash:" + genesisBlock.getHash());
        System.out.println("\tprevBlockHash:" + genesisBlock.getPrevBlockHash());
        System.out.println("\tdata:" + genesisBlock.getData());
        SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
        String date = sdf.format(new Date(genesisBlock.getTimeStamp()*1000L));

        System.out.println("\ttimeStamp:" + date);

        //2.创建第二个区块
        Block block2 = Block.newBlock(genesisBlock.getHash(), "I am hanru",1);
        System.out.println("第二个区块的信息：");
        System.out.println("\thash:" + block2.getHash());
        System.out.println("\tprevBlockHash:" + block2.getPrevBlockHash());
        System.out.println("\tdata:" + block2.getData());
        String date2 = sdf.format(new Date(block2.getTimeStamp()*1000L));
        System.out.println("\ttimeStamp:" + date2);
    }
} 
```

运行结果如下：

![`img.kongyixueyuan.com/0302_%E8%BF%90%E8%A1%8C%E7%BB%93%E6%9E%9C1.png`](img/85aa31197e8203ca64aee3ca227b73b8.jpg)

### 4.4\. 区块链 BlockChain

有了区块，下面让我们来实现区块**链**。本质上，区块链就是一个有着特定结构的数据库，是一个有序，每一个块都连接到前一个块的链表。也就是说，区块按照插入的顺序进行存储，每个块都与前一个块相连。这样的结构，能够让我们快速地获取链上的最新块，并且高效地通过哈希来检索一个块。

![`img.kongyixueyuan.com/14_blockchain2.png`](img/26446113c3e416d8b10d2d5b3614c661.jpg)

#### 4.4.1 定义区块链

在 Java 中，可以通过一个 `list` 、 `map` 等来实现这个结构。 但是在基本的原型阶段，我们只用到了 `list`，因为现在还不需要通过哈希来获取块。

`Blockchain.java`文件中：

```java
/**
 * 区块链
 * @author hanru
 *
 */
@Data
@AllArgsConstructor
public class Blockchain {

    /**
     * 存储区块的集合
     */
    private List<Block> blockList;
} 
```

#### 4.4.2 创建一个区块链

为了创建一个区块链，并且能够加入一个新的块，我们必须要有一个已有的块，因为新区块需要引用之前的区块`hash`作为`prevBlockHash`。但是，初始状态下，我们的链是空的，一个块都没有！所以，在任何一个区块链中，都必须至少有一个块。这个块，也就是链中的第一个块，通常叫做创世区块，简称叫创世块（**genesis block**），而创世区块的`prevBlockHash`固定为 0。

接下来，我们提供一个方法，用于创建一个区块链，并且该区块链中包含了创世区块。

`BlockChain`类中：

```java
 /**
     * 创建区块链
     * @return
     */
    public static Blockchain newBlockchain() {
        List<Block> blocks = new LinkedList<>();
        blocks.add(Block.newGenesisBlock());
        return new Blockchain(blocks);
    } 
```

这就是我们的第一个区块链！是不是出乎意料地简单? 就是一个 `Block` 的 list 集合。

在 main 函数中进行测试：

```java
package cldy.hanru.blockchain;

import java.text.SimpleDateFormat;
import java.util.Date;

import cldy.hanru.blockchain.block.Block;
import cldy.hanru.blockchain.block.Blockchain;

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

    }
} 
```

运行结果：

![`img.kongyixueyuan.com/0304_%E8%BF%90%E8%A1%8C%E7%BB%93%E6%9E%9C2.png`](img/36227b2c1280b50ffe16e9d7b0d5b08e.jpg)

#### 4.4.3 添加新区块

现在，让我们能够给它添加一个区块：

`BlockChain`类中：

```java
/**
 * 根据 block，添加区块
 * @param block
 */
public void addBlock(Block block) {
    this.blockList.add(block);
}

/**
 * 根据 data 添加区块
 * @param data
 */
public void addBlock(String data) {
    Block previousBlock = blockList.get(blockList.size() - 1);
    this.addBlock(Block.newBlock(previousBlock.getHash(), data,previousBlock.getHeight()+1));
} 
```

在 main 函数中进行测试：

```java
package cldy.hanru.blockchain;

import java.text.SimpleDateFormat;
import java.util.Date;

import cldy.hanru.blockchain.block.Block;
import cldy.hanru.blockchain.block.Blockchain;

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

    }

} 
```

运行结果：

![`img.kongyixueyuan.com/0304_%E8%BF%90%E8%A1%8C%E7%BB%93%E6%9E%9C3.png`](img/1823ba33898bee1e995276610823488a.jpg)

## 5\. 总结

我们创建了一个非常简单的区块链原型：它仅仅是一个数组构成的一系列区块，每个块都与前一个块相关联。真实的区块链要比这复杂得多。在我们的区块链中，加入新的块非常简单，也很快，但是在真实的区块链中，加入新的块需要很多工作：你必须要经过十分繁重的计算（这个机制叫做工作量证明），来获得添加一个新块的权力。并且，区块链是一个分布式数据库，并且没有单一决策者。因此，要加入一个新块，必须要被网络的其他参与者确认和同意（这个机制叫做共识（consensus））。

[项目源代码](https://github.com/rubyhan1314/BitcoinForJava)