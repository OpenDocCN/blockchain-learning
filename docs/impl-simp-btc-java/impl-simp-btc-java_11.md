# 第十一章 默克尔树 (Merkle Tree)

# 第十一章 默克尔树(Merkle Tree)

## 1\. 课程目标

1.  知道什么是 Merkle 树
2.  Merkle 树的用途
3.  比特币种如何应用 Merkle 树
4.  学会在项目中实现 Merkle 树

## 2\. 项目代码及效果展示

### 2.1 项目代码结构

![`img.kongyixueyuan.com/010_001_%E9%A1%B9%E7%9B%AE%E7%BB%93%E6%9E%841.png`](img/55bea84a162c022fbdaeda76c616582c.jpg)

### 2.2 项目运行结果

![`img.kongyixueyuan.com/010_002_%E8%BF%90%E8%A1%8C%E6%95%88%E6%9E%9C.gif`](img/abe2e7019315596b7cbcf06cdab252ba.jpg)

其实运行效果和之前的没有区别，因为我们并没有增加新的功能，只是在创建创世区块和根据转账交易挖掘新区块时，添加了 Merkle 字段。

## 3\. 创建项目

### 3.1 创建工程

打开 IntelliJ IDEA 的工作空间，将上一个项目代码目录`part8_Transaction2`，复制为`part9_Merkle`。

然后打开 IntelliJ IDEA 开发工具。

打开工程：`part9_Merkle`，并删除 target 目录。然后进行以下修改：

```java
step1：先将项目重新命名为：part9_Merkle。
step2：修改 pom.xml 配置文件。
    改为：<artifactId>part9_Merkle</artifactId>标签
    改为：<name>part9_Merkle Maven Webapp</name> 
```

> 说明：我们每一章节的项目代码，都是在上一个章节上进行添加。所以拷贝上一次的项目代码，然后进行新内容的添加或修改。

###

### 3.2 代码实现

#### 3.2.1 创建`MerkleTree.java`文件

打开`cldy.hanru.blockchain.transaction`包，创建`MerkleTree.java`文件，并添加代码如下：

```java
 package cldy.hanru.blockchain.transaction;

import cldy.hanru.blockchain.util.ByteUtils;
import lombok.Data;
import org.apache.commons.codec.digest.DigestUtils;

import java.util.ArrayList;
import java.util.List;

/**
 * 默克尔树
 * @author hanru
 */
@Data
public class MerkleTree {

    /**
     * 根节点
     */
    private Node root;
    /**
     * 叶子节点 Hash
     */
    private byte[][] leafHashes;

    /**
     * 构建叶子节点
     *
     * @param hash
     * @return
     */
    private static Node constructLeafNode(byte[] hash) {
        Node leaf = new Node();
        leaf.hash = hash;
        return leaf;
    }

    /**
     * 底部节点构建
     *
     * @param hashes
     * @return
     */
    private List<Node> bottomLevel(byte[][] hashes) {
        List<Node> parents = new ArrayList<>(hashes.length / 2);

        for (int i = 0; i < hashes.length - 1; i += 2) {
            Node leaf1 = constructLeafNode(hashes[i]);
            Node leaf2 = constructLeafNode(hashes[i + 1]);

            Node parent = constructInternalNode(leaf1, leaf2);
            parents.add(parent);
        }

        if (hashes.length % 2 != 0) {
            Node leaf = constructLeafNode(hashes[hashes.length - 1]);
            // 奇数个节点的情况，复制最后一个节点
            Node parent = constructInternalNode(leaf, leaf);
            parents.add(parent);
        }

        return parents;
    }

    /**
     * 构建内部节点
     *
     * @param leftChild
     * @param rightChild
     * @return
     */
    private Node constructInternalNode(Node leftChild, Node rightChild) {
        Node parent = new Node();
        if (rightChild == null) {
            parent.hash = leftChild.hash;
        } else {
            parent.hash = internalHash(leftChild.hash, rightChild.hash);
        }
        parent.left = leftChild;
        parent.right = rightChild;
        return parent;
    }
    /**
     * 计算内部节点 Hash
     *
     * @param leftChildHash
     * @param rightChildHash
     * @return
     */
    private byte[] internalHash(byte[] leftChildHash, byte[] rightChildHash) {
        byte[] mergedBytes = ByteUtils.merge(leftChildHash, rightChildHash);
        return DigestUtils.sha256(mergedBytes);
    }

    /**
     * 构建一个层级节点
     *
     * @param children
     * @return
     */
    private List<Node> internalLevel(List<Node> children) {
        List<Node> parents = new ArrayList<>(children.size() / 2);
        for (int i = 0; i < children.size() - 1; i += 2) {
            Node child1 = children.get(i);
            Node child2 = children.get(i + 1);

            Node parent = constructInternalNode(child1, child2);
            parents.add(parent);
        }

        // 内部节点奇数个，只对 left 节点进行计算
        if (children.size() % 2 != 0) {
            Node child = children.get(children.size() - 1);
            Node parent = constructInternalNode(child, null);
            parents.add(parent);
        }

        return parents;
    }

    /**
     * 从底部叶子节点开始往上构建整个 Merkle Tree
     *
     * @param leafHashes
     */
    private void constructTree(byte[][] leafHashes) {
        if (leafHashes == null || leafHashes.length < 1) {
            throw new RuntimeException("ERROR:Fail to construct merkle tree ! leafHashes data invalid ! ");
        }
        this.leafHashes = leafHashes;
        List<Node> parents = bottomLevel(leafHashes);
        while (parents.size() > 1) {
            parents = internalLevel(parents);
        }
        root = parents.get(0);
    }

    public MerkleTree(byte[][] leafHashes) {
        constructTree(leafHashes);
    }

    /**
     * step1：构建节点类
     * Merkle Tree 节点
     */
    @Data
    public static class Node {
        private byte[] hash;
        private Node left;
        private Node right;
    }
} 
```

#### 3.2.2 修改`Block.java`文件

打开`cldy.hanru.blockchain.block`包，修改`Block.java`文件。

修改步骤：

```java
修改步骤：
step1：添加 HashTransactions()方法，生成交易列表产生的 Merkle 树根的 hash 
```

修改完后代码如下：

```java
package cldy.hanru.blockchain.block;

import java.time.Instant;
import java.util.ArrayList;
import java.util.List;

import cldy.hanru.blockchain.transaction.MerkleTree;
import cldy.hanru.blockchain.transaction.Transaction;
import cldy.hanru.blockchain.util.ByteUtils;
import lombok.NoArgsConstructor;
import org.apache.commons.codec.binary.Hex;

import cldy.hanru.blockchain.pow.PowResult;
import cldy.hanru.blockchain.pow.ProofOfWork;
import lombok.AllArgsConstructor;
import lombok.Data;
import org.apache.commons.codec.digest.DigestUtils;

/**
 * 区块
 * @author hanru
 *
 */
@AllArgsConstructor
@NoArgsConstructor
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
    private List<Transaction> transactions;

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
     * @param transactions
     * @return
     */
    public static Block newBlock(String previousHash, List<Transaction> transactions,long height) {
        Block block = new Block("", previousHash, transactions, Instant.now().getEpochSecond(),height,0);
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
     * @param coinbase
     * @return
     */
    public static Block newGenesisBlock(Transaction coinbase) {
        List<Transaction> transactions = new ArrayList<>();
        transactions.add(coinbase);
        return Block.newBlock(ByteUtils.ZERO_HASH, transactions,0);
    }

    /**
     * 对区块中的交易信息进行 Hash 计算
     *
     * @return
     */
    public byte[] hashTransaction() {
        byte[][] txIdArrays = new byte[this.getTransactions().size()][];
//        for (int i = 0; i < this.getTransactions().size(); i++) {
//            txIdArrays[i] = this.getTransactions().get(i).getTxId();
//        }
//        return DigestUtils.sha256(ByteUtils.merge(txIdArrays));

        for (int i = 0; i < this.getTransactions().size(); i++) {
            txIdArrays[i] = this.getTransactions().get(i).getData();
        }
        return new MerkleTree(txIdArrays).getRoot().getHash();
    }

} 
```

#### 3.2.3 修改`blockchain.sh`脚本文件

最后修改`blockchain.sh`脚本文件，修改后内容如下：

```java
#!/bin/bash

set -e

# Check if the jar has been built.
if [ ! -e target/part9_Merkle-jar-with-dependencies.jar ]; then
  echo "Compiling blockchain project to a JAR"
  mvn package -DskipTests
fi

java -jar target/part9_Merkle-jar-with-dependencies.jar "$@" 
```

##

## 4\. MerkleTree 讲解

### 4.1 什么是 Merkle 树

假设你已经知道了什么是哈希算法以及哈希是用来干啥的。

网络传输数据的时候，A 收到 B 的传过来的文件，需要确认收到的文件有没有损坏。如何解决？

有一种方法是 B 在传文件之前先把文件的 hash 结果给 A，A 收到文件再计算一次哈希然后和收到的哈希比较就知道文件有无损坏。

但是当文件很大的时候，往往需要把文件拆分很多的数据块各自传输，这个时候就需要知道每个数据块的哈希值。怎么办呢？

这种情况，可以在下载数据之前先下载一份哈希列表(hash list)，这个列表每一项对应一个数据块的哈希值。对这个 hash list 拼接后可以计算一个根 hash。实际应用中，我们只要确保从一个可信的渠道获取正确的根 hash，就可以确保下载正确的文件。

**似乎很完美了。但是还不够好！**

上面基于 hash list 的方案这样一个问题：

**有些时候我们获取(遍历)所有数据块的 hash list 代价比较大，只能获取部分节点的哈希。**

有没有一种方法可以通过部分 hash 就能校验整个文件的完整性呢？

答案是肯定的，merkle tree 能做到。它长这样子:

![`img.kongyixueyuan.com/010_003_merkle.jpg`](img/7ad6e2b1617453cc12e3d13d6dd83ac8.jpg)

特点如下:

1、数据结构是一个树，可以是二叉树，也可以是多叉树（本 BLOG 以二叉树来分析）

2、Merkle Tree 的叶子节点的 value 是数据集合的单元数据或者单元数据 HASH。

3、Merke Tree 非叶子节点 value 是其所有子节点 value 的 HASH 值。

很明显，这种结构跟 hash list 相比较，根哈希不是用所有的数据块哈希拼接起来计算的，而是通过一个层级的关系计算出来的。

**在上图中，叶子节点 node7 的 value（v7) = hash(f1),是 f1 文件的 HASH;而其父亲节点 node3 的 value = hash(v7, v8)，也就是其子节点 node7 node8 的值得 HASH**

### 4.2 其它应用场景

**文件下载**

假设我有两台机器，A 和 B，有一个文件从 A 传输到 B。B 首先获取可信的文件 merkle tree，当文件下载完毕后，B 通过自己构建的 merkle tree 根节点和获取的根节点比较，如果不一致,通过这种二叉树的结构可以在 log(N)的复杂度快速定位到出错的数据块。

**副本同步**

一个集群里的所有机器，需要保持数据的同步，如果数据不一致，需要快速的定位到不一致的节点。

可以在每台机器上针对每个区间里的数据构造一棵 Merkle Tree，这样，在两台机器间进行数据比对时，从 Merkle Tree 的根节点开始进行比对，如果根节点一样，则表示两个副本目前是一致的，不再需要任何处理；如果不一样，则遍历 Merkle Tree，定位到不一致的节点也非常快速。

**merkle tree 应用在区块链上**

比特币的完整节点保存了区块链自创世区块以来的所有信息，成为区块链的一个本地信息副本。随着新区块的产生，节点会收到网络传输的新区块信息，并对新区块信息进行验证，将其链接到现有的区块上，使得区块链本地副本信息不断的更新和扩展，如图 1 所示：

![`img.kongyixueyuan.com/010_006_block2.jpeg`](img/178543bbc990797af4446437adb473ca.jpg)

**比特币系统的区块链中，每个区块都有一个 merkle tree**

区块链的每一个区块都包含了该区块产生期间的所有交易，并用 Merkle 树的形式表示。Merkle 树是一种 Hash 二叉树，用作快速归纳和校验大规模数据完整性的数据结构。

![`img.kongyixueyuan.com/010_005_block.png`](img/f7b3aff688984c4758e8187571d1abf1.jpg)

可以看到 merkle root 哈希值存在每一个区块的头部，通过这个 root 值连接着区块体，而区块体内就是包含着大量的交易。每个交易就相当于前面提到的数据块，交易本身有都有自己的哈希值来唯一标识自己。

在比特币网络中，Merkle 树被用来归纳一个区块的所有交易，同时生成整个交易集合的数字指纹，且提供了一种校验区块是否存在某交易的高效途径。生成一棵完整的 Merkle 树需要递归地对哈希节点对进行哈希，并将新生成的哈希节点插入到 Merkle 树中，直到只剩一个哈希节点，该节点就是 Merkle 树的根。在比特币的 Merkle 树中两次使用到了 SHA256 算法，因此其加密哈希算法也被称为 Double-SHA256。

当 N 个数据元素经过加密后插入 Merkle 树，至多计算 2*log2(N)次就能检查出任意某数据元素是否在该树中，这使得该数据结构非常高效；但同时无法从 Merkle 树内找到对应的交易，这是区块链单向验证性的特点。Merkle 树是自底向上构建的，举例说明：同一时间发生 A、B、C、D 四笔交易，起始所有交易都存于基础节点，分别进行 Hash，以交易 A 的 Hash 过程为例，得到 HA = SHA256(SHA256(交易 A))，同样得到 HB、HC、HD；然后创建二层节点 HAB= SHA256(SHA256(HA + HB))，同样得到 HCD；继续操作直到生成顶端唯一的节点——HABCD。如下图所示：

![`img.kongyixueyuan.com/010_007_merkle3.png`](img/14914540467a5590dc440f7830fb018c.jpg)

因为 Merkle 树是二叉树，所以每一层需要偶数个点，如果节点数为奇数，系统将复制一份数值使得节点数变成偶数。这种偶数个分层节点的树也被称为平衡二叉树。如下图所示，C 点就被复制了一份。

![`img.kongyixueyuan.com/010_008_merkle4.png`](img/078cb4ca14273f4b7ddcedffd928ff06.jpg)

为四个交易构造 Merkle 树的方法同样适用于为任意交易数量构造 Merkle 树。在比特币中，在单个区块中有成百上千的交易是非常普遍的，这些交易都会采用同样的方法归纳起来，产生一个仅仅 32 字节的数据作为 Merkle 根。下图出现了多个节点，其运算过程与四个节点是一致的，其实无论多少个节点都可以通过 Merkle 树归纳为 32 个字节。

![`img.kongyixueyuan.com/010_009_merkle5.png`](img/f177225fb3b476754267365f54a5cefc.jpg)

为了证明区块链内存在某一个特定的交易，一个节点只需要计算 log2(N)个 32 字节的哈希值，形成一条从特定交易到树根的认证路径或者 Merkle 路径即可。随着交易数量的急剧增加，这样的计算量就显得异常重要，因为相对于交易数量的增长，以基底为 2 的交易数量的对数的增长会缓慢许多。这使得比特币节点能够高效地产生一条 10 或者 12 个哈希值（320-384 字节）的路径，来证明了在一个巨量字节大小的区块中某笔交易的存在。

在图 5 中，一个节点能够通过生成一条仅有 4 个 32 字节哈希值长度（总 128 字节）的 Merkle 路径，来证明区块中存在一笔交易 K。该路径有 4 个哈希值 （在图 5 中由蓝色标注）HL、HIJ、HMNOP 和 HABCDEFGH。由这 4 个哈希值产生的认证路径，再通过计算另外四对哈希值 HKL、 HIJKL、HIJKLMNOP 和 Merkle 树根（在图中由虚线标注），任何节点都能证明 HK（在图中由绿色标注）包含在 Merkle 根中。

![`img.kongyixueyuan.com/010_010_merkle6.jpeg`](img/637081ed3c8cceb5b451514ff9072e9d.jpg)

用列表表示交易数量对路径大小的影响，如表 1 所示，当区块大小由 16 笔交易（4KB）急剧增加至 65535 笔交易（16MB）时，为证明交易存在的 Merkle 路径长度增长极其缓慢，仅仅从 128 字节到 512 字节。有了 Merkle 树，一个节点能够仅下载区块头（80 字节/区块），然后从一个节点回溯就能认证一笔交易的存在，而不需要存储或者传输区块链中大多数内容。这被称作简单支付验证（SPV）节点，它不需要下载整个区块，而通过 Merkle 路径去验证交易的存在。

### 4.3 比特币中的 Merkle 树

截止到目前为止，完整的比特币数据库（也就是区块链）需要超过 220 Gb 的磁盘空间。因为比特币的去中心化特性，网络中的每个节点必须是独立，自给自足的，也就是每个节点必须存储一个区块链的完整副本。随着越来越多的人使用比特币，这条规则变得越来越难以遵守：因为不太可能每个人都去运行一个全节点。并且，由于节点是网络中的完全参与者，它们负有相关责任：节点必须验证交易和区块。另外，要想与其他节点交互和下载新块，也有一定的网络流量需求。

在中本聪的比特币原始论文中，对这个问题也有一个解决方案：简易支付验证（Simplified Payment Verification, SPV）。SPV 是一个比特币轻节点，它不需要下载整个区块链，也**不需要验证区块和交易**。相反，它会在区块链查找交易（为了验证支付），并且需要连接到一个全节点来检索必要的数据。这个机制允许在仅运行一个全节点的情况下有多个轻钱包。

**但是轻量级的钱包没有完整的数据，如何验证一笔支付的合法性呢?**

merkle tree 就起到了关键的作用。

当然 SPV 有它的局限性(这个有空再写文章细讲)。

为了实现 SPV，需要有一个方式来检查是否一个区块包含了某笔交易，而无须下载整个区块。这就是 Merkle 树所要完成的事情。

比特币用 Merkle 树来获取交易哈希，哈希被保存在区块头中，并会用于工作量证明系统。到目前为止，我们只是将一个块里面的每笔交易哈希连接了起来，将在上面应用了 SHA-256 算法。虽然这是一个用于获取区块交易唯一表示的一个不错的途径，但是它没有利用到 Merkle 树。

![`img.kongyixueyuan.com/010_004_merkle.png`](img/dc2afa14ef78eaeeaba55282f4de510c.jpg)

每个块都会有一个 Merkle 树，它从叶子节点（树的底部）开始，一个叶子节点就是一个交易哈希（比特币使用双 SHA256 哈希）。叶子节点的数量必须是双数，但是并非每个块都包含了双数的交易。因为，如果一个块里面的交易数为单数，那么就将最后一个叶子节点（也就是 Merkle 树的最后一个交易，不是区块的最后一笔交易）复制一份凑成双数。

从下往上，两两成对，连接两个节点哈希，将组合哈希作为新的哈希。新的哈希就成为新的树节点。重复该过程，直到仅有一个节点，也就是树根。根哈希然后就会当做是整个块交易的唯一标示，将它保存到区块头，然后用于工作量证明。

Merkle 树的好处就是一个节点可以在不下载整个块的情况下，验证是否包含某笔交易。并且这些只需要一个交易哈希，一个 Merkle 树根哈希和一个 Merkle 路径。

### 4.4 Merkle 树实现

在本项目中我们可以在 Block 结构中添加 Merkle 树根 Hash，用于工作量证明，但是我们没有实现 spv 验证。

首先我们从创建 Merkle 结构体开始，现在`cldy.hanru.blockchain.transaction`包下创建`MerkleTree.java`文件，并添加代码如下：

```java
/**
 * 默克尔树
 * @author hanru
 */
@Data
public class MerkleTree {
    /**
     * 根节点
     */
    private Node root;
    /**
     * 叶子节点 Hash
     */
    private byte[][] leafHashes;
/**
     * step1：构建节点类
     * Merkle Tree 节点
     */
    @Data
    public static class Node {
        private byte[] hash;
        private Node left;
        private Node right;
    }
} 
```

因为 Merkle 的叶子节点和其他节点不同，叶子节点没有根节点，所以我们单独设计方法用于创建叶子节点，所以在该 java 文件那种， 继续添加方法如下：

```java
/**
     * 构建叶子节点
     *
     * @param hash
     * @return
     */
    private static Node constructLeafNode(byte[] hash) {
        Node leaf = new Node();
        leaf.hash = hash;
        return leaf;
    }

    /**
     * 底部节点构建
     *
     * @param hashes
     * @return
     */
    private List<Node> bottomLevel(byte[][] hashes) {
        List<Node> parents = new ArrayList<>(hashes.length / 2);

        for (int i = 0; i < hashes.length - 1; i += 2) {
            Node leaf1 = constructLeafNode(hashes[i]);
            Node leaf2 = constructLeafNode(hashes[i + 1]);

            Node parent = constructInternalNode(leaf1, leaf2);
            parents.add(parent);
        }

        if (hashes.length % 2 != 0) {
            Node leaf = constructLeafNode(hashes[hashes.length - 1]);
            // 奇数个节点的情况，复制最后一个节点
            Node parent = constructInternalNode(leaf, leaf);
            parents.add(parent);
        }

        return parents;
    } 
```

接下来，我们需要提供方法用于创建非叶子节点，代码如下：

```java
 /**
     * 构建内部节点
     *
     * @param leftChild
     * @param rightChild
     * @return
     */
    private Node constructInternalNode(Node leftChild, Node rightChild) {
        Node parent = new Node();
        if (rightChild == null) {
            parent.hash = leftChild.hash;
        } else {
            parent.hash = internalHash(leftChild.hash, rightChild.hash);
        }
        parent.left = leftChild;
        parent.right = rightChild;
        return parent;
    }
    /**
     * 计算内部节点 Hash
     *
     * @param leftChildHash
     * @param rightChildHash
     * @return
     */
    private byte[] internalHash(byte[] leftChildHash, byte[] rightChildHash) {
        byte[] mergedBytes = ByteUtils.merge(leftChildHash, rightChildHash);
        return DigestUtils.sha256(mergedBytes);
    }

    /**
     * 构建一个层级节点
     *
     * @param children
     * @return
     */
    private List<Node> internalLevel(List<Node> children) {
        List<Node> parents = new ArrayList<>(children.size() / 2);
        for (int i = 0; i < children.size() - 1; i += 2) {
            Node child1 = children.get(i);
            Node child2 = children.get(i + 1);

            Node parent = constructInternalNode(child1, child2);
            parents.add(parent);
        }

        // 内部节点奇数个，只对 left 节点进行计算
        if (children.size() % 2 != 0) {
            Node child = children.get(children.size() - 1);
            Node parent = constructInternalNode(child, null);
            parents.add(parent);
        }

        return parents;
    } 
```

先用`constructInternalNode()`方法，根据传入的左右节点，创建出当前的节点。然后再调用`internalLevel()`方法，循环生成这一层级的节点。

接下来我们需要使用循环创建每一层节点，最后生成一个 Merkle 树根节点。(注意，此处使用递归算法实现也可以)。所以我们在该 java 文件中继续添加代码如下：

```java
 /**
     * 从底部叶子节点开始往上构建整个 Merkle Tree
     *
     * @param leafHashes
     */
    private void constructTree(byte[][] leafHashes) {
        if (leafHashes == null || leafHashes.length < 1) {
            throw new RuntimeException("ERROR:Fail to construct merkle tree ! leafHashes data invalid ! ");
        }
        this.leafHashes = leafHashes;
        List<Node> parents = bottomLevel(leafHashes);
        while (parents.size() > 1) {
            parents = internalLevel(parents);
        }
        root = parents.get(0);
    } 
```

到此我们就可以调用`constructTree()`方法，获取 Merkle 树根节点了。

最后，我们需要打开 Block.java 文件，修改 hashTransactions()方法，使用 Merkle 根节点表示交易列表的 hash，代码如下：

```java
 /**
     * 对区块中的交易信息进行 Hash 计算
     *
     * @return
     */
    public byte[] hashTransaction() {
        byte[][] txIdArrays = new byte[this.getTransactions().size()][];
//        for (int i = 0; i < this.getTransactions().size(); i++) {
//            txIdArrays[i] = this.getTransactions().get(i).getTxId();
//        }
//        return DigestUtils.sha256(ByteUtils.merge(txIdArrays));

        for (int i = 0; i < this.getTransactions().size(); i++) {
            txIdArrays[i] = this.getTransactions().get(i).getData();
        }
        return new MerkleTree(txIdArrays).getRoot().getHash();
    } 
```

最后，我们进行测试，先创建创世区块，在终端中输入以下命令：

```java
hanru:part9_Merkle ruby$ mvn package
hanru:part9_Merkle ruby$ ./blockchain.sh h
hanru:part9_Merkle ruby$ ./blockchain.sh createwallet
hanru:part9_Merkle ruby$ ./blockchain.sh createwallet
hanru:part9_Merkle ruby$ ./blockchain.sh printaddresses
hanru:part9_Merkle ruby$ ./blockchain.sh createblockchain -address 1B9bGLLYMAJ6F5DPUebLQf5kNLA9Jfo1Wr 
```

运行效果如下：

![`img.kongyixueyuan.com/010_011_%E8%BF%90%E8%A1%8C%E7%BB%93%E6%9E%9C.png`](img/66a7c2d269b5916c0f2efb3215a99fcb.jpg)

接下来，我们进行一次转账，转账产生交易后，会进行挖矿，区块中就会产生 Merkle 树根节点 Hash，在终端中输入以下内容：

```java
hanru:part9_Merkle ruby$ ./blockchain.sh getbalance -address 1B9bGLLYMAJ6F5DPUebLQf5kNLA9Jfo1Wr
hanru:part9_Merkle ruby$ ./blockchain.sh send -from 1B9bGLLYMAJ6F5DPUebLQf5kNLA9Jfo1Wr -to 139ztYVzj5xxyaqpZ1xinaXvB5fvd4swwT -amount 4
hanru:part9_Merkle ruby$ ./blockchain.sh getbalance -address 1B9bGLLYMAJ6F5DPUebLQf5kNLA9Jfo1Wr
hanru:part9_Merkle ruby$ ./blockchain.sh getbalance -address 139ztYVzj5xxyaqpZ1xinaXvB5fvd4swwT 
```

运行结果如下：

![`img.kongyixueyuan.com/010_012_%E8%BF%90%E8%A1%8C%E7%BB%93%E6%9E%9C.png`](img/670ea3c70ee74ddb2cce2f182ac2d4b5.jpg)

## 5\. 总结

Merkle 树被 SPV 节点广泛使用。SPV 节点不保存所有交易也不会下载整个区块，仅仅保存区块头。其使用认证路径或者 Merkle 路径来验证交易是否在区块中，而不必下载区块中所有交易。例如，一个 SPV 节点欲知其钱包中某个比特币地址即将到达的支付，该节点会在节点间的通信链接上建立起 Bloom 过滤器，限制只接受含有目标比特币地址的交易。当节点探测到某交易符合 Bloom 过滤器，它将以 Merkle 区块消息的形式发送该区块。Merkle 区块消息包含区块头和一条连接目标交易与 Merkle 根的 Merkle 路径。SPV 节点能够使用该路径找到与该交易相关的区块，进而验证对应区块中该交易是否存在。

[项目源代码](https://github.com/rubyhan1314/BitcoinForJava)