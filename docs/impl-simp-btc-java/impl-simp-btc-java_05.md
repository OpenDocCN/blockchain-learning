# 第五章 区块持久化存储

# 第五章 持久化(Persistence)

到目前为止，我们已经构建了一个有工作量证明机制的区块链。有了工作量证明，挖矿也就有了着落。虽然目前距离一个有着完整功能的区块链越来越近了，但是它仍然缺少了一些重要的特性。在今天的内容中，我们会将区块链持久化到一个数据库中，然后会提供一个简单的命令行接口，用来完成一些与区块链的交互操作。本质上，区块链是一个分布式数据库，不过，我们暂时先忽略 “分布式” 这个部分，仅专注于 “存储” 这一点。

## 1\. 课程目标

1.  了解什么是持久化
2.  学会在使用 RocksDB 数据库
3.  学会 Block 区块对象的序列化和反序列化
4.  项目中的区块能够持久化存储
5.  学会使用迭代器遍历区块

## 2\. 项目代码及效果展示

### 2.1 项目代码结构

![`img.kongyixueyuan.com/05001_%E9%A1%B9%E7%9B%AE%E7%BB%93%E6%9E%84.png`](img/fbd2757501f894eb9176d81c8293c6f3.jpg)

### 2.2 项目运行结果

![`img.kongyixueyuan.com/05002_%E8%BF%90%E8%A1%8C%E7%BB%93%E6%9E%9C.gif`](img/44fa74d0eb3a5563f93e225f97f47b21.jpg)

## 3\. 创建项目

### 3.1 创建工程

打开 IntelliJ IDEA 的工作空间，将上一个项目代码目录`part2_Proof_Of_Work`，复制为`part3_Persistence`。

然后打开 IntelliJ IDEA 开发工具。

打开工程：`part3_Persistence`，并删除 target 目录。然后进行以下修改：

```java
step1：先将项目重新命名为：part3_Persistence。
step2：修改 pom.xml 配置文件。
    改为：<artifactId>part3_Persistence</artifactId>标签
    改为：<name>part3_Persistence Maven Webapp</name> 
```

> 说明：我们每一章节的项目代码，都是在上一个章节上进行添加。所以拷贝上一次的项目代码，然后进行新内容的添加或修改。

### 3.2 代码实现

#### 3.2.1 创建 class 文件：`SerializeUtils.java`

在`SerializeUtils.java`文件中编写代码如下：

```java
package cldy.hanru.blockchain.util;

import com.esotericsoftware.kryo.Kryo;
import com.esotericsoftware.kryo.io.Input;
import com.esotericsoftware.kryo.io.Output;

/**
 * 序列化工具类
 * @author hanru
 */
public class SerializeUtils {

    /**
     * 序列化
     * @param object 需要序列化的对象
     * @return
     */
    public static byte[] serialize(Object object) {
        Output output = new Output(4096, -1);
        new Kryo().writeClassAndObject(output, object);
        byte[] bytes = output.toBytes();
        output.close();
        return bytes;
    }

    /**
     * 反序列化
     * @param bytes 对象对应的字节数组
     * @return
     */
    public static Object deserialize(byte[] bytes) {
        Input input = new Input(bytes);
        Object obj = new Kryo().readClassAndObject(input);
        input.close();
        return obj;
    }
} 
```

#### 3.2.2 创建 class 文件：`RocksDBUtils.java`

新建`cldy.hanru.blockchain.store`包，并创建`RocksDBUtils.java`文件

在`RocksDBUtils.java`文件中编写代码如下：

```java
package cldy.hanru.blockchain.store;

import cldy.hanru.blockchain.block.Block;
import cldy.hanru.blockchain.util.SerializeUtils;
import org.rocksdb.RocksDB;
import org.rocksdb.RocksDBException;

import java.util.Map;

/**
 * 数据库存储的工具类
 * @author hanru
 */
public class RocksDBUtils {
    /**
     * 区块链数据文件
     */
    private static final String DB_FILE = "blockchain.db";

    /**
     * 区块桶前缀
     */
    private static final String BLOCKS_BUCKET_KEY = "blocks";

    /**
     * 最新一个区块的 hash
     */
    private static final String LAST_BLOCK_KEY = "l";

    private volatile static RocksDBUtils instance;

    /**
     * 获取 RocksDBUtils 的单例
     * @return
     */
    public static RocksDBUtils getInstance() {
        if (instance == null) {
            synchronized (RocksDBUtils.class) {
                if (instance == null) {
                    instance = new RocksDBUtils();
                }
            }
        }
        return instance;
    }

    private RocksDBUtils() {
        openDB();
        initBlockBucket();
    }

    private RocksDB db;

    /**
     * block buckets
     */
    private Map<String, byte[]> blocksBucket;

    /**
     * 打开数据库
     */
    private void openDB() {
        try {
            db = RocksDB.open(DB_FILE);
        } catch (RocksDBException e) {
            throw new RuntimeException("打开数据库失败。。 ! ", e);
        }
    }
    /**
     * 初始化 blocks 数据桶
     */
    private void initBlockBucket() {
        try {
            //
            byte[] blockBucketKey = SerializeUtils.serialize(BLOCKS_BUCKET_KEY);
            byte[] blockBucketBytes = db.get(blockBucketKey);
            if (blockBucketBytes != null) {
                blocksBucket = (Map) SerializeUtils.deserialize(blockBucketBytes);
            } else {
                blocksBucket = new HashMap<>();
                db.put(blockBucketKey, SerializeUtils.serialize(blocksBucket));
            }
        } catch (RocksDBException e) {
            throw new RuntimeException("初始化 block 的 bucket 失败。。! ", e);
        }
    }

    /**
     * 保存区块
     *
     * @param block
     */
    public void putBlock(Block block) {
        try {
            blocksBucket.put(block.getHash(), SerializeUtils.serialize(block));
            db.put(SerializeUtils.serialize(BLOCKS_BUCKET_KEY), SerializeUtils.serialize(blocksBucket));
        } catch (RocksDBException e) {
            throw new RuntimeException("存储区块失败。。  ", e);
        }
    }

    /**
     * 查询区块
     *
     * @param blockHash
     * @return
     */
    public Block getBlock(String blockHash) {
        return (Block) SerializeUtils.deserialize(blocksBucket.get(blockHash));
    }

    /**
     * 保存最新一个区块的 Hash 值
     *
     * @param tipBlockHash
     */
    public void putLastBlockHash(String tipBlockHash) {
        try {
            blocksBucket.put(LAST_BLOCK_KEY, SerializeUtils.serialize(tipBlockHash));
            db.put(SerializeUtils.serialize(BLOCKS_BUCKET_KEY), SerializeUtils.serialize(blocksBucket));
        } catch (RocksDBException e) {
            throw new RuntimeException("数据库存储最新区块 hash 失败。。 ", e);
        }
    }

    /**
     * 查询最新一个区块的 Hash 值
     *
     * @return
     */
    public String getLastBlockHash() {
        byte[] lastBlockHashBytes = blocksBucket.get(LAST_BLOCK_KEY);
        if (lastBlockHashBytes != null) {
            return (String) SerializeUtils.deserialize(lastBlockHashBytes);
        }
        return "";
    }

    /**
     * 关闭数据库
     */
    public void closeDB() {
        try {
            db.close();
        } catch (Exception e) {
            throw new RuntimeException("关闭数据库失败。。 ", e);
        }
    }

} 
```

#### 3.2.3 修改`Blockchain.java`文件

打开`cldy.hanru.blockchain.block`包。修改`Blockchain.go`文件。

```java
修改步骤：
step1：删除 list 集合，添加 lastBlockHash 字段
step2：修改 newBlockchain()
step3：修改两个 addBlock()方法
step4：添加内部类 BlockchainIterator
step5：内部类中，添加 hashNext()方法
step6：内部类中，添加 next()方法
step7：Blockchain 类中，添加 getBlockchainIterator()方法 
```

修改完后代码如下:

```java
package cldy.hanru.blockchain.block;

import cldy.hanru.blockchain.store.RocksDBUtils;
import cldy.hanru.blockchain.util.ByteUtils;
import lombok.AllArgsConstructor;
import lombok.Data;
import org.apache.commons.lang3.StringUtils;

/**
 * 区块链
 * @author hanru
 *
 */
@Data
@AllArgsConstructor
public class Blockchain {

    /**
     * 最后一个区块的 hash
     */
    private String lastBlockHash;

    /**
     * 创建区块链
     * @return
     */
    public static Blockchain newBlockchain() {

        String lastBlockHash = RocksDBUtils.getInstance().getLastBlockHash();
        if (StringUtils.isBlank(lastBlockHash)){
            //对应的 bucket 不存在，说明是第一次获取区块链实例
            Block genesisBlock = Block.newGenesisBlock();
            lastBlockHash = genesisBlock.getHash();
            RocksDBUtils.getInstance().putBlock(genesisBlock);
            RocksDBUtils.getInstance().putLastBlockHash(lastBlockHash);

        }
        return new Blockchain(lastBlockHash);
    }

    /**
     * 根据 block，添加区块
     * @param block
     */
    public void addBlock(Block block) {

        RocksDBUtils.getInstance().putLastBlockHash(block.getHash());
        RocksDBUtils.getInstance().putBlock(block);
        this.lastBlockHash = block.getHash();

    }

    /**
     * 根据 data 添加区块
     * @param data
     */
    public void addBlock(String data)  throws Exception{

        String lastBlockHash = RocksDBUtils.getInstance().getLastBlockHash();
        Block lastBlock = RocksDBUtils.getInstance().getBlock(lastBlockHash);
        if (StringUtils.isBlank(lastBlockHash)){
            throw new Exception("还没有数据库，无法直接添加区块。。");
        }
        this.addBlock(Block.newBlock(lastBlockHash,data,lastBlock.getHeight()+1));

    }

    /**
     * 区块链迭代器：内部类
     */
    public class BlockchainIterator{

        /**
         * 当前区块的 hash
         */
        private String currentBlockHash;

        /**
         * 构造函数
         * @param currentBlockHash
         */
        public BlockchainIterator(String currentBlockHash) {
            this.currentBlockHash = currentBlockHash;
        }

        /**
         * 判断是否有下一个区块
         * @return
         */
        public boolean hashNext() {
            if (ByteUtils.ZERO_HASH.equals(currentBlockHash)) {
                return false;
            }
            Block lastBlock = RocksDBUtils.getInstance().getBlock(currentBlockHash);
            if (lastBlock == null) {
                return false;
            }
            // 如果是创世区块
            if (ByteUtils.ZERO_HASH.equals(lastBlock.getPrevBlockHash())) {
                return true;
            }
            return RocksDBUtils.getInstance().getBlock(lastBlock.getPrevBlockHash()) != null;
        }

        /**
         * 迭代获取区块
         * @return
         */
        public Block next() {
            Block currentBlock = RocksDBUtils.getInstance().getBlock(currentBlockHash);
            if (currentBlock != null) {
                this.currentBlockHash = currentBlock.getPrevBlockHash();
                return currentBlock;
            }
            return null;
        }
    }

    /**
     * 添加方法，用于获取迭代器实例
     * @return
     */
    public BlockchainIterator getBlockchainIterator() {
        return new BlockchainIterator(lastBlockHash);
    }

} 
```

#### 3.2.4 修改`pom.xml`配置文件

修改`pom.xml`配置文件，添加依赖包。

修改完后代码如下：

```java
<?xml version="1.0" encoding="UTF-8"?>

<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>cldy.hanru.blockchain</groupId>
    <artifactId>part3_Persistence</artifactId>
    <version>1.0-SNAPSHOT</version>
    <packaging>jar</packaging>

    <name>part3_Persistence Maven Webapp</name>
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

        <!-- 对象序列化/反序列化框架 -->
        <dependency>
            <groupId>com.esotericsoftware</groupId>
            <artifactId>kryo</artifactId>
            <version>4.0.1</version>
        </dependency>

        <!-- rocksdb -->
        <dependency>
            <groupId>org.rocksdb</groupId>
            <artifactId>rocksdbjni</artifactId>
            <version>5.9.2</version>
        </dependency>

    </dependencies>

<!--
    <build>
        <finalName>part3_Persistence</finalName>
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

#### 3.2.5 修改`main.java`

在`main.java`中修改测试代码

```java
package cldy.hanru.blockchain;

import cldy.hanru.blockchain.block.Block;
import cldy.hanru.blockchain.block.Blockchain;
import cldy.hanru.blockchain.pow.ProofOfWork;
import cldy.hanru.blockchain.store.RocksDBUtils;

import java.text.SimpleDateFormat;
import java.util.Date;

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

//        Blockchain blockchain = Blockchain.newBlockchain();
//
//
//        System.out.println("创世链的信息：");
//        System.out.println("区块的长度：" + blockchain.getBlockList().size());
//
//        //4.添加区块
//        blockchain.addBlock("Send 1 BTC to 韩茹");
//        blockchain.addBlock("Send 2 more BTC to ruby");
//        blockchain.addBlock("Send 4 more BTC to 王二狗");
//
//        for (int i = 0; i < blockchain.getBlockList().size(); i++) {
//            Block block = blockchain.getBlockList().get(i);
//            System.out.println("第" + block.getHeight() + "个区块信息：");
//            System.out.println("\tprevBlockHash: " + block.getPrevBlockHash());
//            System.out.println("\tData: " + block.getData());
//            System.out.println("\tHash: " + block.getHash());
//            SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
//            String date2 = sdf.format(new Date(block.getTimeStamp() * 1000L));
//            System.out.println("\ttimeStamp:" + date2);
//
//
//            ProofOfWork pow = ProofOfWork.newProofOfWork(block);
//            System.out.println("是否有效: " + pow.validate() + "\n");
//            System.out.println();
//        }

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

        //5.测试持久化
        Blockchain blockchain = Blockchain.newBlockchain();
        System.out.println(blockchain);
//        RocksDBUtils.getInstance().closeDB();

        //6.添加区块
        try {
            blockchain.addBlock("Send 1 BTC to 韩茹");
            blockchain.addBlock("Send 2 more BTC to ruby");
            blockchain.addBlock("Send 4 more BTC to 王二狗");
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            RocksDBUtils.getInstance().closeDB();
        }

        //7.遍历区块
//        Blockchain blockchain = Blockchain.newBlockchain();
        Blockchain.BlockchainIterator iterator = blockchain.getBlockchainIterator();
        long index = 0;
        while (iterator.hashNext()) {
            Block block = iterator.next();
            System.out.println("第" + block.getHeight() + "个区块信息：");
            System.out.println("\tprevBlockHash: " + block.getPrevBlockHash());
            System.out.println("\tData: " + block.getData());
            System.out.println("\tHash: " + block.getHash());
            SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
            String date = sdf.format(new Date(block.getTimeStamp() * 1000L));
            System.out.println("\ttimeStamp:" + date);
            System.out.println();
        }
        RocksDBUtils.getInstance().closeDB();

    }

} 
```

## 4\. 持久化讲解

### 4.1 选择数据库

目前，我们的区块链实现里面并没有用到数据库，而是在每次运行程序时，简单地将区块链存储在内存中。那么一旦程序退出，所有的内容就都消失了。我们没有办法再次使用这条链，也没有办法与其他人共享，所以我们需要把它存储到磁盘上。

那么，我们要用哪个数据库呢？实际上，任何一个数据库都可以。在比特币原始论文 中，并没有提到要使用哪一个具体的数据库，它完全取决于开发者如何选择。 Bitcoin Core，最初由中本聪发布，现在是比特币的一个参考实现，它使用的是 LevelDB，在我们的另一个版本 golang 版中使用的是 [BoltDB](https://github.com/boltdb/bolt) ，因为对 Go 语言支持比较好。但是我们这里使用的是 Java 来实现，BoltDB 不支持 Java，这里我们选用 [Rocksdb](https://github.com/facebook/rocksdb) 。

> RocksDB 是由 Facebook 数据库工程团队开发和维护的一款 key-value 存储引擎，比 LevelDB 性能更加强大，有关 Rocksdb 的详细介绍，请查看官方文档：[`github.com/facebook/rocksdb`](https://github.com/facebook/rocksdb) ，这里不多做介绍。

### 4.2 对比比特币

在开始实现持久化的逻辑之前，我们首先需要决定到底要如何在数据库中进行存储。为此，我们可以参考 Bitcoin Core 的做法：

简单来说，Bitcoin Core 使用两个 “bucket” 来存储数据：

1.  其中一个 bucket 是 **blocks**，它存储了描述一条链中所有块的元数据
2.  另一个 bucket 是 **chainstate**，存储了一条链的状态，也就是当前所有的未花费的交易输出，和一些元数据

此外，出于性能的考虑，Bitcoin Core 将每个区块（block）存储为磁盘上的不同文件。如此一来，就不需要仅仅为了读取一个单一的块而将所有（或者部分）的块都加载到内存中。但是，为了简单起见，我们并不会实现这一点。

在 **blocks** 中，**key -> value** 为：

| key | value |
| --- | --- |
| `b` + 32 字节的 block hash | block index record |
| `f` + 4 字节的 file number | file information record |
| `l` + 4 字节的 file number | the last block file number used |
| `R` + 1 字节的 boolean | 是否正在 reindex |
| `F` + 1 字节的 flag name length + flag name string | 1 byte boolean: various flags that can be on or off |
| `t` + 32 字节的 transaction hash | transaction index record |

在 **chainstate**，**key -> value** 为：

| key | value |
| --- | --- |
| `c` + 32 字节的 transaction hash | unspent transaction output record for that transaction |
| `B` | 32 字节的 block hash: the block hash up to which the database represents the unspent transaction outputs |

因为目前还没有交易，所以我们只需要 **blocks** bucket。另外，正如上面提到的，我们会将整个数据库存储为单个文件，而不是将区块存储在不同的文件中。所以，我们也不会需要文件编号（file number）相关的东西。最终，我们会用到的键值对有：

1.  32 字节的 block-hash -> block 结构
2.  `l` -> 链中最后一个块的 hash

这就是实现持久化机制所有需要了解的内容了。

### 4.3 RocksDB 的配置

#### 4.3.1 修改 pom.xml 文件

打开 pom.xml 文件：

```java
<!-- rocksdb -->
<dependency>
    <groupId>org.rocksdb</groupId>
    <artifactId>rocksdbjni</artifactId>
    <version>5.9.2</version>
</dependency> 
```

### 4.4 序列化

#### 4.4.1 序列化和反序列化

所谓**序列化**就是将对象状态转换为可保持或传输的格式(比如[]byte，或者二进制数据等)的过程。与**序列化**相对的是**反序列化**,再将这些数据转换为对象。这两个过程结合起来,可以轻松地存储和传输数据。

我们若要将区块链持久化存储到数据库中，其实就是将每个区块对象存入到数据库中。

上面提到，在 RocksDB 中，只能存储 `[]byte` 类型的数据，但是我们想要存储 `Block` 实例。所以，我们需要将对象进行序列化。

首先修改 pom.xml 文件，添加序列化和反序列化所需要的依赖包 [Kryo](https://github.com/EsotericSoftware/kryo)。

```java
<!-- 对象序列化/反序列化框架 -->
<dependency>
     <groupId>com.esotericsoftware</groupId>
     <artifactId>kryo</artifactId>
     <version>4.0.1</version>
</dependency> 
```

接下来，让我们来实现 `SerializeUtils.java`文件中添加 的 `serialize()` 方法：

```java
 /**
     * 序列化
     * @param object 需要序列化的对象
     * @return
     */
    public static byte[] serialize(Object object) {
        Output output = new Output(4096, -1);
        new Kryo().writeClassAndObject(output, object);
        byte[] bytes = output.toBytes();
        output.close();
        return bytes;
    } 
```

接下来，我们需要一个解序列化的函数，它会接受一个字节数组作为输入，并返回一个 `Block`实例：

```java
 /**
     * 反序列化
     * @param bytes 对象对应的字节数组
     * @return
     */
    public static Object deserialize(byte[] bytes) {
        Input input = new Input(bytes);
        Object obj = new Kryo().readClassAndObject(input);
        input.close();
        return obj;
    } 
```

这就是序列化部分的内容了。

### 4.5 持久化

**对于数据库中，我们设计的结构是，每个 block 的 hash 值作为 key，而 block 序列化后的数据作为 value。还需要单独存储一个"l"作为 key(也可以选择其他的字符串作为 key，比如："lasthash"等)，用于存储最后一个区块的 hash 值。这样，我们就可以根据 l 获取最后一个区块 hash，根据该 hash 可以获取到最后一个区块。然后再获取前一个区块，以此类推，直到创世区块。我们就可以获取所有的区块数据了。**

接下来我们设计一个相关的工具类`RocksDBUtils`，主要功能如下:

*   putBlock：保存区块

*   getBlock：查询区块

*   putLastBlockHash：保存最新一个区块的 Hash 值

*   getLastBlockHash：查询最新一个区块的 Hash 值

#### 4.5.1 创建`RocksDBUtils.java`文件

首先先创建一个包：`cldy.hanru.blockchain.store`，然后新建一个 java 文件，命名为：`RocksDBUtils.java`

先定义几个常量：

```java
 /**
     * 区块链数据文件
     */
    private static final String DB_FILE = "blockchain.db";

    /**
     * 区块桶前缀
     */
    private static final String BLOCKS_BUCKET_KEY = "blocks";

    /**
     * 最新一个区块的 hash
     */
    private static final String LAST_BLOCK_KEY = "l"; 
```

#### 4.5.2 获取 RocksDBUtils 实例

接下来设计一个单例，用于获取`RocksDBUtils`实例：

```java
private volatile static RocksDBUtils instance;

    /**
     * 获取 RocksDBUtils 的单例
     * @return
     */
    public static RocksDBUtils getInstance() {
        if (instance == null) {
            synchronized (RocksDBUtils.class) {
                if (instance == null) {
                    instance = new RocksDBUtils();
                }
            }
        }
        return instance;
    }

    private RocksDBUtils() {
        openDB();
        initBlockBucket();
    } 
```

上述代码，就是一般的单例实现(考虑了线程安全)。首先定义一个`RocksDBUtils`实例，私有化构造函数，添加`getInstance()`方法，用于返回`RocksDBUtils`实例对象。

#### 4.5.3 初始化数据库

接下来定义两个字段：一个是 RocksDB 对象，一个 map 集合，用于实现 bucket 存储区块。

```java
 private RocksDB db;

    /**
     * block buckets
     */
    private Map<String, byte[]> blocksBucket; 
```

> 注意：BoltDB 支持 Bucket 的特性，而 RocksDB 不支持，所以需要我们自己使用 Map 来做一个映射。

然后我们就可以定义两个方法，用于打开数据库和初始化 bucket，代码如下：

```java
 /**
     * 打开数据库
     */
    private void openDB() {
        try {
            db = RocksDB.open(DB_FILE);
        } catch (RocksDBException e) {
            throw new RuntimeException("打开数据库失败。。 ! ", e);
        }
    }
    /**
     * 初始化 blocks 数据桶
     */
    private void initBlockBucket() {
        try {
            //
            byte[] blockBucketKey = SerializeUtils.serialize(BLOCKS_BUCKET_KEY);
            byte[] blockBucketBytes = db.get(blockBucketKey);
            if (blockBucketBytes != null) {
                blocksBucket = (Map) SerializeUtils.deserialize(blockBucketBytes);
            } else {
                blocksBucket = new HashMap();
                db.put(blockBucketKey, SerializeUtils.serialize(blocksBucket));
            }
        } catch (RocksDBException e) {
            throw new RuntimeException("初始化 block 的 bucket 失败。。! ", e);
        }
    } 
```

初始化的方法，也比较直接，因为 RocksDB 数据库只支持 byte 数组，所以我们需要将要存储的 key 和 value 都转为 byte 数组类型。先判断数据库中的 map 序列化后的数据是否存在，如果存在就取出设置给 blocksBucket 这个 map。否则就要新建一个 HashMap，并序列化后存储到数据库中，方便下次使用。

#### 4.5.4 核心方法：添加区块

然后，实现 4 个核心方法，用于区块的存储。

```java
 /**
     * 保存区块
     *
     * @param block
     */
    public void putBlock(Block block) {
        try {
            blocksBucket.put(block.getHash(), SerializeUtils.serialize(block));
            db.put(SerializeUtils.serialize(BLOCKS_BUCKET_KEY), SerializeUtils.serialize(blocksBucket));
        } catch (RocksDBException e) {
            throw new RuntimeException("存储区块失败。。  ", e);
        }
    }

    /**
     * 查询区块
     *
     * @param blockHash
     * @return
     */
    public Block getBlock(String blockHash) {
        return (Block) SerializeUtils.deserialize(blocksBucket.get(blockHash));
    }

    /**
     * 保存最新一个区块的 Hash 值
     *
     * @param tipBlockHash
     */
    public void putLastBlockHash(String tipBlockHash) {
        try {
            blocksBucket.put(LAST_BLOCK_KEY, SerializeUtils.serialize(tipBlockHash));
            db.put(SerializeUtils.serialize(BLOCKS_BUCKET_KEY), SerializeUtils.serialize(blocksBucket));
        } catch (RocksDBException e) {
            throw new RuntimeException("数据库存储最新区块 hash 失败。。 ", e);
        }
    }

    /**
     * 查询最新一个区块的 Hash 值
     *
     * @return
     */
    public String getLastBlockHash() {
        byte[] lastBlockHashBytes = blocksBucket.get(LAST_BLOCK_KEY);
        if (lastBlockHashBytes != null) {
            return (String) SerializeUtils.deserialize(lastBlockHashBytes);
        }
        return "";
    } 
```

一个一个方法进行分析：

首先`putBlock()`方法，用于存储一个区块到数据库中。先将一个区块的 hash 作为 key 区块序列化后的数据作为 value 值。存入到 map 中，再将 map 序列化后存入到数据库中。

接下来，`getBlock()`方法，就用于从数据库中存储的 map 中，根据区块的 hash 读取出区块的数据，并反序列化成一个 Block 实例对象。

`putLastBlockHash()`方法，用于设置数据库中 l 对应的值，存储了最新区块的 hash 值。

`getLastBlockHash()`方法，用于获取数据库中最新区块的 hash 值。

#### 4.5.5 关闭数据库

最后，关闭数据库。

```java
 /**
     * 关闭数据库
     */
    public void closeDB() {
        try {
            db.close();
        } catch (Exception e) {
            throw new RuntimeException("关闭数据库失败。。 ", e);
        }
    } 
```

#### 4.5.6 修改 BlockChain 类

程序中的`BlockChain`，是用于存储`Block`的，修改前我们使用了`List`来存储 Block 区块。如果要实现持久化操作，就不能使用数组。所以修改 BlockChain 的字段为：`lastBlockHash`,用于存储数据库中最后一个区块的`hash`值。

对于`newBlockchain()()`函数，用于获得一个`BlockChain`对象，修改前，会创建一个新的 `Blockchain` 实例，并向其中加入创世块。而现在，我们希望它做的事情有:

我们先从数据库中获取`l`这个 key 对应的最后一个区块的 hash：lastBlockHash。

1.  判断 lastBlockHash 是否为空，如果不为空，说明区块数据存在：
    1.  创建 BlockChain 实例。
    2.  读取数据库中最后一个区块的 hash，并设置给 BlockChain 实例的 lastBlockHash 字段。
2.  如果 lastBlockHash 为空，说明区块数据不存在：
    1.  首先我们需要先创建一个创世区块
    2.  将创世区块序列化后存入到数据库中
    3.  将创世区块的 hash 保存为最后一个块的 hash
    4.  创建 BlockChain 实例，设置 lastBlockHash 为创世区块的 hash，并返回该 blockchain 实例。

具体的流程如下：

![`img.kongyixueyuan.com/05003_blockchain_flow.jpg`](img/57e304651ab08ad08c12817d0de3f294.jpg)

在`BlockChain.java`中，修改`newBlockchain()`如下：

```java
 /**
     * 创建区块链
     * @return
     */
    public static Blockchain newBlockchain() {

        String lastBlockHash = RocksDBUtils.getInstance().getLastBlockHash();
        if (StringUtils.isBlank(lastBlockHash)){
            //对应的 bucket 不存在，说明是第一次获取区块链实例
            Block genesisBlock = Block.newGenesisBlock();
            lastBlockHash = genesisBlock.getHash();
            RocksDBUtils.getInstance().putBlock(genesisBlock);
            RocksDBUtils.getInstance().putLastBlockHash(lastBlockHash);

        }
        return new Blockchain(lastBlockHash);
    } 
```

#### 4.5.7 修改`addBlock()`方法

接下来我们想要更新的是 `addBlock()`方法：现在向链中加入区块，就不是像之前向一个 List 列表中加入一个元素那么简单了。从现在开始，我们会将区块存储在数据库里面：

```java
 /**
     * 根据 block，添加区块
     * @param block
     */
    public void addBlock(Block block) {

        RocksDBUtils.getInstance().putLastBlockHash(block.getHash());
        RocksDBUtils.getInstance().putBlock(block);
        this.lastBlockHash = block.getHash();

    }

    /**
     * 根据 data 添加区块
     * @param data
     */
    public void addBlock(String data)  throws Exception{

        String lastBlockHash = RocksDBUtils.getInstance().getLastBlockHash();
        Block lastBlock = RocksDBUtils.getInstance().getBlock(lastBlockHash);
        if (StringUtils.isBlank(lastBlockHash)){
            throw new Exception("还没有数据库，无法直接添加区块。。");
        }
        this.addBlock(Block.newBlock(lastBlockHash,data,lastBlock.getHeight()+1));

    } 
```

#### 4.5.8 修改 main 函数进行测试

在`main.java`中添加测试代码如下：

```java
public static void main(String[] args) {
        //5.测试持久化
        Blockchain blockchain = Blockchain.newBlockchain();
        System.out.println(blockchain);
//        RocksDBUtils.getInstance().closeDB();

        //6.添加区块
        try {
            blockchain.addBlock("Send 1 BTC to 韩茹");
            blockchain.addBlock("Send 2 more BTC to ruby");
            blockchain.addBlock("Send 4 more BTC to 王二狗");
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            RocksDBUtils.getInstance().closeDB();
        }
} 
```

运行结果如下：

![`img.kongyixueyuan.com/05004_%E8%BF%90%E8%A1%8C%E7%BB%93%E6%9E%9C.png`](img/65c16f1adbfccd9daeb54c502575db6c.jpg)

### 4.6 检查遍历区块链

现在，产生的所有块都会被保存到一个数据库里面，所以我们可以重新打开一个链，然后向里面加入新块。但是在实现这一点后，我们失去了之前一个非常好的特性：再也无法打印区块链的区块了，因为现在不是将区块存储在一个数组，而是放到了数据库里面。让我们来解决这个问题！

#### 4.6.1 定义一个`BlockchainIterator`内部类

`RocksDB` 允许对里面的所有 key 进行迭代，但是所有的 `key` 都以字节序进行存储，而且我们想要以区块能够进入区块链中的顺序进行打印。此外，因为我们不想将所有的块都加载到内存中（因为我们的区块链数据库可能很大！或者现在可以假装它可能很大），我们将会一个一个地读取它们。故而，我们需要一个区块链迭代器（`BlockchainIterator`），因为是对于 Blockchain 实力来讲的功能，所以可以设计为内部类。

在`Blockchain.java`文件中，在`Blockchain`类里，添加`BlockChainIterator`类：

```java
/**
     * 区块链迭代器：内部类
     */
    public class BlockchainIterator{

        /**
         * 当前区块的 hash
         */
        private String currentBlockHash;
    } 
```

#### 4.6.2 hashNext()方法

`BlockchainIterator` 的工作原理：根据当前的区块 hash，判断是否存在该区块。如果有可以调用`next()`返回链中的区块。我们根据迭代器的原理，添加一个`hasNext()`方法，用于判断区块是否存在。

```java
 /**
         * 判断是否有下一个区块
         * @return
         */
        public boolean hashNext() {
            if (ByteUtils.ZERO_HASH.equals(currentBlockHash)) {
                return false;
            }
            Block lastBlock = RocksDBUtils.getInstance().getBlock(currentBlockHash);
            if (lastBlock == null) {
                return false;
            }
            // 如果是创世区块
            if (ByteUtils.ZERO_HASH.equals(lastBlock.getPrevBlockHash())) {
                return true;
            }
            return RocksDBUtils.getInstance().getBlock(lastBlock.getPrevBlockHash()) != null;
        } 
```

判断区块是否存在的根据就是，根据 currentBlockHash 获取 block 实例，判断实力是否为 null。

#### 4.6.3 添加`Next()`方法

`hasNext()`方法判断区块是否存在，`next()`用于获取区块，并将`currentBlockHash`设置为下一个区块的 hash。

在`BlockchainIterator`类中，添加`Next()`方法，代码如下:

```java
 /**
         * 迭代获取区块
         * @return
         */
        public Block next() {
            Block currentBlock = RocksDBUtils.getInstance().getBlock(currentBlockHash);
            if (currentBlock != null) {
                this.currentBlockHash = currentBlock.getPrevBlockHash();
                return currentBlock;
            }
            return null;
        } 
```

#### 4.6.4 获取`Iterator`实例

每当要对链中的块进行迭代时，我们就会创建一个迭代器，里面存储了当前迭代的块哈希（`currentBlockHash`）。

在`Blockchain`类中，添加`getBlockchainIterator()`方法用于获取`BlockchainIterator`实例，代码如下:

```java
 /**
     * 添加方法，用于获取迭代器实例
     * @return
     */
    public BlockchainIterator getBlockchainIterator() {
        return new BlockchainIterator(lastBlockHash);
    } 
```

注意，迭代器中的初始状态为链中的最后一个区块的 lastBlockHash，因此区块将从尾到头（创世块为头），也就是从最新的到最旧的进行获取。实际上，**选择一个 lastBlockHash 就是意味着给一条链“投票”**。一条链可能有多个分支，最长的那条链会被认为是主分支。在获得一个 lastBlockHash （可以是链中的任意一个块）之后，我们就可以重新构造整条链，找到它的长度和需要构建它的工作。这同样也意味着，一个 lastBlockHash 也就是区块链的一种标识符。

#### 4.6.5 `main.java`中测试

接下来我们在`main`中进行测试，此处我们注释掉上一次的测试代码，打印之前添加好的区块就好。我们写个循环遍历打印。

修改`main.java`代码的内容如下:

```java
public static void main(String[] args) {
    /*
        //5.测试持久化
        Blockchain blockchain = Blockchain.newBlockchain();
        System.out.println(blockchain);
        //RocksDBUtils.getInstance().closeDB();

        //6.添加区块
        try {
            blockchain.addBlock("Send 1 BTC to 韩茹");
            blockchain.addBlock("Send 2 more BTC to ruby");
            blockchain.addBlock("Send 4 more BTC to 王二狗");
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            RocksDBUtils.getInstance().closeDB();
        }
       */

    //7.遍历区块
        Blockchain blockchain = Blockchain.newBlockchain();
        Blockchain.BlockchainIterator iterator = blockchain.getBlockchainIterator();
        long index = 0;
        while (iterator.hashNext()) {
            Block block = iterator.next();
            System.out.println("第" + block.getHeight() + "个区块信息：");
            System.out.println("\tprevBlockHash: " + block.getPrevBlockHash());
            System.out.println("\tData: " + block.getData());
            System.out.println("\tHash: " + block.getHash());
            SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
            String date = sdf.format(new Date(block.getTimeStamp() * 1000L));
            System.out.println("\ttimeStamp:" + date);
            System.out.println();
        }
        RocksDBUtils.getInstance().closeDB();
} 
```

运行结果:

![`img.kongyixueyuan.com/05005_%E8%BF%90%E8%A1%8C%E7%BB%93%E6%9E%9C.png`](img/b7bbf1371a1f6873e1a74675aedf1fbe.jpg)

这就是数据库部分的内容了！

## 5\. 总结

通过本章节的学习，我们了解持久化的原理，并采用`RocksDB`进行区块的持久化存储。RocksDB 和我们平时使用 map 集合类似，都是通过 put()和 get()方法，进行存储和获取数据。通过对 RocksDB 数据库的操作，我们实现了创建区块添加到区块链，实际上是将区块的数据，进行序列化后，存入到数据库中。此外，我们还提供了迭代器获取每个区块，并进行区块的打印。

[项目源代码](https://github.com/rubyhan1314/BitcoinForJava)