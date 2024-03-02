# 第七章 UTXO 模型

# 第七章 交易(Transactions)（1）

交易（transaction）是比特币的核心所在，而区块链唯一的目的，也正是为了能够安全可靠地存储交易。在区块链中，交易一旦被创建，就没有任何人能够再去修改或是删除它。今天，我们将会开始实现交易。不过，由于交易是很大的话题，我们把它分为两部分来讲：在今天这个部分，我们会实现交易的基本框架。在第二部分，我们会继续讨论它的一些细节。

## 1\. 课程目标

1.  了解什么是交易
2.  了解什么是输入和输出
3.  学会创建转账交易
4.  学会 UTXO 模型
5.  学会查询余额

## 2\. 项目代码及效果展示

### 2.1 项目代码结构

![`img.kongyixueyuan.com/007_001_%E9%A1%B9%E7%9B%AE%E7%BB%93%E6%9E%84.png`](img/0f01f1d4ccb901b68ba8df11a919352c.jpg)

### 2.2 项目运行结果

创建创世区块效果图：

![`img.kongyixueyuan.com/007_007_%E8%BF%90%E8%A1%8C%E7%BB%93%E6%9E%9C.gif`](img/c7c9a2a8d5eb26ed150588db7aed14e2.jpg)

转账效果图：

![`img.kongyixueyuan.com/007_008_%E8%BD%AC%E8%B4%A6.gif`](img/d14101492029784799db3c968efe96f1.jpg)

查询余额效果图：

![`img.kongyixueyuan.com/007_009_%E4%BD%99%E9%A2%9D.gif`](img/188cf1162fc62b42ba206879203ff939.jpg)

## 3\. 创建项目

### 3.1 创建工程

打开 IntelliJ IDEA 的工作空间，将上一个项目代码目录`part4_CLI`，复制为`part5_Transaction`。

然后打开 IntelliJ IDEA 开发工具。

打开工程：`part5_Transaction`，并删除 target 目录。然后进行以下修改：

```java
step1：先将项目重新命名为：part5_Transaction。
step2：修改 pom.xml 配置文件。
    改为：<artifactId>part5_Transaction</artifactId>标签
    改为：<name>part5_Transaction Maven Webapp</name> 
```

> 说明：我们每一章节的项目代码，都是在上一个章节上进行添加。所以拷贝上一次的项目代码，然后进行新内容的添加或修改。

### 3.2 代码实现

#### 3.2.1 创建 java 文件：`Transaction.java`

打开`cldy.hanru.blockchain`目录下，新建一个包：`transaction`。然后新建一个 java 文件，命名为：`Transaction.java`。在`Transaction.java`文件中编写代码如下：

```java
package cldy.hanru.blockchain.transaction;

import cldy.hanru.blockchain.block.Blockchain;
import cldy.hanru.blockchain.util.SerializeUtils;
import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;
import org.apache.commons.codec.binary.Hex;
import org.apache.commons.codec.digest.DigestUtils;
import org.apache.commons.lang3.ArrayUtils;
import org.apache.commons.lang3.StringUtils;

import java.util.Iterator;
import java.util.Map;

/**
 * @author hanru
 */
@Data
@AllArgsConstructor
@NoArgsConstructor
public class Transaction {
    private static final int SUBSIDY = 10;

    /**
     * 交易的 Hash
     */
    private byte[] txId;
    /**
     * 交易输入
     */
    private TXInput[] inputs;
    /**
     * 交易输出
     */
    private TXOutput[] outputs;

    /**
     * 设置交易 ID
     */
    private void setTxId() {
        this.setTxId(DigestUtils.sha256(SerializeUtils.serialize(this)));
    }

    /**
     * 创建 CoinBase 交易
     *
     * @param to   收账的钱包地址
     * @param data 解锁脚本数据
     * @return
     */
    public static Transaction newCoinbaseTX(String to, String data) {
        if (StringUtils.isBlank(data)) {
            data = String.format("Reward to '%s'", to);
        }
        // 创建交易输入
        TXInput txInput = new TXInput(new byte[]{}, -1, data);
        // 创建交易输出
        TXOutput txOutput = new TXOutput(SUBSIDY, to);
        // 创建交易
        Transaction tx = new Transaction(null, new TXInput[]{txInput}, new TXOutput[]{txOutput});
        // 设置交易 ID
        tx.setTxId();
        return tx;
    }

    /**
     * 从 from 向  to 支付一定的 amount 的金额
     *
     * @param from       支付钱包地址
     * @param to         收款钱包地址
     * @param amount     交易金额
     * @param blockchain 区块链
     * @return
     */
    public static Transaction newUTXOTransaction(String from, String to, int amount, Blockchain blockchain) throws Exception {
        SpendableOutputResult result = blockchain.findSpendableOutputs(from, amount);
        int accumulated = result.getAccumulated();
        Map<String, int[]> unspentOuts = result.getUnspentOuts();

        if (accumulated < amount) {
            throw new Exception("ERROR: Not enough funds");
        }
        Iterator<Map.Entry<String, int[]>> iterator = unspentOuts.entrySet().iterator();

        TXInput[] txInputs = {};
        while (iterator.hasNext()) {
            Map.Entry<String, int[]> entry = iterator.next();
            String txIdStr = entry.getKey();
            int[] outIdxs = entry.getValue();
            byte[] txId = Hex.decodeHex(txIdStr);
            for (int outIndex : outIdxs) {
                txInputs = ArrayUtils.add(txInputs, new TXInput(txId, outIndex, from));
            }
        }

        TXOutput[] txOutput = {};
        txOutput = ArrayUtils.add(txOutput, new TXOutput(amount, to));
        if (accumulated > amount) {
            txOutput = ArrayUtils.add(txOutput, new TXOutput((accumulated - amount), from));
        }

        Transaction newTx = new Transaction(null, txInputs, txOutput);
        newTx.setTxId();
        return newTx;
    }

    /**
     * 是否为 Coinbase 交易
     *
     * @return
     */
    public boolean isCoinbase() {
        return this.getInputs().length == 1
                && this.getInputs()[0].getTxId().length == 0
                && this.getInputs()[0].getTxOutputIndex() == -1;
    }
} 
```

#### 3.2.2 创建`TXInput.java`文件

在`cldy.hanru.blockchain.transaction`包下，新建 java 文件：`TXInput.java`文件。

添加代码如下：

```java
package cldy.hanru.blockchain.transaction;

import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

/**
 * @author hanru
 */
@AllArgsConstructor
@NoArgsConstructor
@Data
public class TXInput {

    /**
     * 交易 Id 的 hash 值
     */
    private byte[] txId;
    /**
     * 交易输出索引
     */
    private int txOutputIndex;
    /**
     * 解锁脚本
     */
    private String scriptSig;

    /**
     * 判断解锁数据是否能够解锁交易输出
     *
     * @param unlockingData
     * @return
     */
    public boolean canUnlockOutputWith(String unlockingData) {
        return this.getScriptSig().endsWith(unlockingData);
    }
} 
```

#### 3.2.3 创建`TXOutput.java`文件

在`cldy.hanru.blockchain.transaction`包下，新建 java 文件：`TXOutput.java`文件。

添加代码如下：

```java
package cldy.hanru.blockchain.transaction;

import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

/**
 * @author hanru
 */
@Data
@AllArgsConstructor
@NoArgsConstructor
public class TXOutput {

    /**
     * 数值金额
     */
    private int value;
    /**
     * 锁定脚本
     */
    private String scriptPubKey;

    /**
     * 判断解锁数据是否能够解锁交易输出
     *
     * @param unlockingData
     * @return
     */
    public boolean canBeUnlockedWith(String unlockingData) {
        return this.getScriptPubKey().endsWith(unlockingData);
    }
} 
```

#### 3.2.4 创建`SpendableOutputResult.java`文件

在`cldy.hanru.blockchain.transaction`包下，新建 java 文件：`SpendableOutputResult.java`文件。

添加代码如下：

```java
package cldy.hanru.blockchain.transaction;

import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.util.Map;

/**
 * @author hanru
 */
@AllArgsConstructor
@NoArgsConstructor
@Data
public class SpendableOutputResult {

    /**
     * 交易时的支付金额
     */
    private int accumulated;
    /**
     * 未花费的交易
     */
    private Map<String, int[]> unspentOuts;
} 
```

#### 3.2.5 修改`Block.java`文件

打开`cldy.hanru.blockchain.block`包。修改`Block.java`文件。

修改步骤：

```java
修改步骤：
step1：修改字段，修改 data 为 transaction
step2：修改方法 newBlock()
step3：修改方法 newGenesisBlock()
step4：增加方法，hashTransaction() 
```

修改完后代码如下：

```java
 package cldy.hanru.blockchain.block;

import java.time.Instant;
import java.util.ArrayList;
import java.util.List;

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
        for (int i = 0; i < this.getTransactions().size(); i++) {
            txIdArrays[i] = this.getTransactions().get(i).getTxId();
        }
        return DigestUtils.sha256(ByteUtils.merge(txIdArrays));

    }

} 
```

#### 3.2.6 修改`ProofOfWork.java`文件

打开`cldy.hanru.blockchain.pow`包。修改`ProofOfWork.java`文件。

修改步骤：

```java
修改步骤：
step1：修改 prepareData()方法
    添加交易 hash
step2：修改方法，run() 
```

修改完后代码如下：

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
//        System.out.printf("开始进行挖矿：%s \n", this.getBlock().getData());
        System.out.printf("开始进行挖矿： \n");
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
//                this.getBlock().getData().getBytes(),
                this.getBlock().hashTransaction(),
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

#### 3.2.7 修改`Blockchain.java`文件

打开`cldy.hanru.blockchain.block`包。修改`Blockchain.java`文件。

修改步骤：

```java
修改步骤：
step1：修改方法 newBlockchain()内容后，并将方法名改为 createBlockchain()
step2：添加方法 mineBlock()
step3：添加方法，getAllSpentTXOs()，从交易输入中查询区块链中所有已被花费了的交易输出
step4：添加方法 findUnspentTransactions()，查找钱包地址对应的所有未花费的交易
step5：添加方法 findUTXO(),查找钱包地址对应的所有 UTXO
step6：添加方法 findSpendableOutputs(),寻找能够花费的交易 
```

修改完后代码如下：

```java
package cldy.hanru.blockchain.block;

import cldy.hanru.blockchain.store.RocksDBUtils;
import cldy.hanru.blockchain.transaction.SpendableOutputResult;
import cldy.hanru.blockchain.transaction.TXInput;
import cldy.hanru.blockchain.transaction.TXOutput;
import cldy.hanru.blockchain.transaction.Transaction;
import cldy.hanru.blockchain.util.ByteUtils;
import lombok.AllArgsConstructor;
import lombok.Data;
import org.apache.commons.codec.binary.Hex;
import org.apache.commons.lang3.ArrayUtils;
import org.apache.commons.lang3.StringUtils;

import java.util.HashMap;
import java.util.List;
import java.util.Map;

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
     * 创建区块链，createBlockchain
     * @param address
     * @return
     */
    public static Blockchain createBlockchain(String address) {

        String lastBlockHash = RocksDBUtils.getInstance().getLastBlockHash();
        if (StringUtils.isBlank(lastBlockHash)){
            //对应的 bucket 不存在，说明是第一次获取区块链实例
            // 创建 coinBase 交易
            Transaction coinbaseTX = Transaction.newCoinbaseTX(address, "");
            Block genesisBlock = Block.newGenesisBlock(coinbaseTX);
//            Block genesisBlock = Block.newGenesisBlock();
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
//    public void addBlock(String data)  throws Exception{
//
//        String lastBlockHash = RocksDBUtils.getInstance().getLastBlockHash();
//        if (StringUtils.isBlank(lastBlockHash)){
//            throw new Exception("还没有数据库，无法直接添加区块。。");
//        }
//        this.addBlock(Block.newBlock(lastBlockHash,data));
//    }

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

    /**
     * 打包交易，进行挖矿
     *
     * @param transactions
     */
    public void mineBlock(List<Transaction> transactions) throws Exception {
        String lastBlockHash = RocksDBUtils.getInstance().getLastBlockHash();
        Block lastBlock = RocksDBUtils.getInstance().getBlock(lastBlockHash);
        if (lastBlockHash == null) {
            throw new Exception("ERROR: Fail to get last block hash ! ");
        }
        Block block = Block.newBlock(lastBlockHash, transactions,lastBlock.getHeight()+1);
        this.addBlock(block);
    }

    /**
     * 从交易输入中查询区块链中所有已被花费了的交易输出
     *
     * @param address 钱包地址
     * @return 交易 ID 以及对应的交易输出下标地址
     * @throws Exception
     */
    private Map<String, int[]> getAllSpentTXOs(String address) {
        // 定义 TxId ——> spentOutIndex[]，存储交易 ID 与已被花费的交易输出数组索引值
        Map<String, int[]> spentTXOs = new HashMap<>();
        for (BlockchainIterator blockchainIterator = this.getBlockchainIterator(); blockchainIterator.hashNext(); ) {
            Block block = blockchainIterator.next();

            for (Transaction transaction : block.getTransactions()) {
                // 如果是 coinbase 交易，直接跳过，因为它不存在引用前一个区块的交易输出
                if (transaction.isCoinbase()) {
                    continue;
                }
                for (TXInput txInput : transaction.getInputs()) {
                    if (txInput.canUnlockOutputWith(address)) {
                        String inTxId = Hex.encodeHexString(txInput.getTxId());
                        int[] spentOutIndexArray = spentTXOs.get(inTxId);
                        if (spentOutIndexArray == null) {
                            spentTXOs.put(inTxId, new int[]{txInput.getTxOutputIndex()});
                        } else {
                            spentOutIndexArray = ArrayUtils.add(spentOutIndexArray, txInput.getTxOutputIndex());
                            spentTXOs.put(inTxId, spentOutIndexArray);
                        }
                    }
                }
            }
        }
        return spentTXOs;
    }
    /**
     * 查找钱包地址对应的所有未花费的交易
     *
     * @param address 钱包地址
     * @return
     */
    private Transaction[] findUnspentTransactions(String address) throws Exception {
        Map<String, int[]> allSpentTXOs = this.getAllSpentTXOs(address);
        Transaction[] unspentTxs = {};

        // 再次遍历所有区块中的交易输出
        for (BlockchainIterator blockchainIterator = this.getBlockchainIterator(); blockchainIterator.hashNext(); ) {
            Block block = blockchainIterator.next();
            for (Transaction transaction : block.getTransactions()) {

                String txId = Hex.encodeHexString(transaction.getTxId());

                int[] spentOutIndexArray = allSpentTXOs.get(txId);

                for (int outIndex = 0; outIndex < transaction.getOutputs().length; outIndex++) {
                    if (spentOutIndexArray != null && ArrayUtils.contains(spentOutIndexArray, outIndex)) {
                        continue;
                    }

                    // 保存不存在 allSpentTXOs 中的交易
                    if (transaction.getOutputs()[outIndex].canBeUnlockedWith(address)) {
                        unspentTxs = ArrayUtils.add(unspentTxs, transaction);
                    }
                }
            }
        }
        return unspentTxs;
    }

    /**
     * 查找钱包地址对应的所有 UTXO
     *
     * @param address 钱包地址
     * @return
     */
    public TXOutput[] findUTXO(String address) throws Exception {
        Transaction[] unspentTxs = this.findUnspentTransactions(address);
        TXOutput[] utxos = {};
        if (unspentTxs == null || unspentTxs.length == 0) {
            return utxos;
        }
        for (Transaction tx : unspentTxs) {
            for (TXOutput txOutput : tx.getOutputs()) {
                if (txOutput.canBeUnlockedWith(address)) {
                    utxos = ArrayUtils.add(utxos, txOutput);
                }
            }
        }
        return utxos;
    }

    /**
     * 寻找能够花费的交易
     *
     * @param address 钱包地址
     * @param amount  花费金额
     */
    public SpendableOutputResult findSpendableOutputs(String address, int amount) throws Exception {
        Transaction[] unspentTXs = this.findUnspentTransactions(address);
        int accumulated = 0;
        Map<String, int[]> unspentOuts = new HashMap<>();
        for (Transaction tx : unspentTXs) {

            String txId = Hex.encodeHexString(tx.getTxId());

            for (int outId = 0; outId < tx.getOutputs().length; outId++) {

                TXOutput txOutput = tx.getOutputs()[outId];

                if (txOutput.canBeUnlockedWith(address) && accumulated < amount) {
                    accumulated += txOutput.getValue();

                    int[] outIds = unspentOuts.get(txId);
                    if (outIds == null) {
                        outIds = new int[]{outId};
                    } else {
                        outIds = ArrayUtils.add(outIds, outId);
                    }
                    unspentOuts.put(txId, outIds);
                    if (accumulated >= amount) {
                        break;
                    }
                }
            }
        }
        return new SpendableOutputResult(accumulated, unspentOuts);
    }

    /**
     * 从 DB 从恢复区块链数据
     *
     * @return
     * @throws Exception
     */
    public static Blockchain initBlockchainFromDB() throws Exception {
        String lastBlockHash = RocksDBUtils.getInstance().getLastBlockHash();
        if (lastBlockHash == null) {
            throw new Exception("ERROR: Fail to init blockchain from db. ");
        }
        return new Blockchain(lastBlockHash);
    }

} 
```

#### 3.2.8 修改修改`CLI.java`文件

打开`cldy.hanru.blockchain.cli`包。修改`CLI.java`文件。

修改步骤：

```java
修改步骤：
step1：修改构造函数
step2：修改 help()方法
step3：修改 createBlockchain()方法
step4：添加 getBalance()方法
step5：添加 send()方法
step6：修改 printChain()方法 
```

修改完后 CLI.java 代码如下：

```java
package cldy.hanru.blockchain.cli;

import cldy.hanru.blockchain.block.Block;
import cldy.hanru.blockchain.block.Blockchain;
import cldy.hanru.blockchain.pow.ProofOfWork;
import cldy.hanru.blockchain.store.RocksDBUtils;
import cldy.hanru.blockchain.transaction.TXInput;
import cldy.hanru.blockchain.transaction.TXOutput;
import cldy.hanru.blockchain.transaction.Transaction;
import org.apache.commons.cli.*;
import org.apache.commons.codec.binary.Hex;
import org.apache.commons.lang3.StringUtils;
import org.apache.commons.lang3.math.NumberUtils;

import java.text.SimpleDateFormat;
import java.util.Arrays;
import java.util.Date;

public class CLI {
    private String[] args;
    private Options options = new Options();

    public CLI(String[] args) {
        this.args = args;
        Option helpCmd = Option.builder("h").desc("show help").build();
        options.addOption(helpCmd);

        Option address = Option.builder("address").hasArg(true).desc("Source wallet address").build();
        Option sendFrom = Option.builder("from").hasArg(true).desc("Source wallet address").build();
        Option sendTo = Option.builder("to").hasArg(true).desc("Destination wallet address").build();
        Option sendAmount = Option.builder("amount").hasArg(true).desc("Amount to send").build();

        options.addOption(address);
        options.addOption(sendFrom);
        options.addOption(sendTo);
        options.addOption(sendAmount);
    }

    /**
     * 打印帮助信息
     */
    private void help() {
        System.out.println("Usage:");
        System.out.println("  getbalance -address ADDRESS - Get balance of ADDRESS");
        System.out.println("  createblockchain -address ADDRESS - Create a blockchain and send genesis block reward to ADDRESS");
        System.out.println("  printchain - Print all the blocks of the blockchain");
        System.out.println("  send -from FROM -to TO -amount AMOUNT - Send AMOUNT of coins from FROM address to TO");
        System.exit(0);
    }

    /**
     * 验证入参
     *
     * @param args
     */
    private void validateArgs(String[] args) {
        if (args == null || args.length < 1) {
            help();
        }
    }

    /**
     * 命令行解析入口
     */
    public void run() {
        this.validateArgs(args);
        try {
            CommandLineParser parser = new DefaultParser();
            CommandLine cmd = parser.parse(options, args);
            switch (args[0]) {
                case "createblockchain":
                    String createblockchainAddress = cmd.getOptionValue("address");
                    if (StringUtils.isBlank(createblockchainAddress)) {
                        help();
                    }
                    this.createBlockchain(createblockchainAddress);
                    break;
                case "getbalance":
                    String getBalanceAddress = cmd.getOptionValue("address");
                    if (StringUtils.isBlank(getBalanceAddress)) {
                        help();
                    }
                    this.getBalance(getBalanceAddress);
                    break;
                case "send":
                    String sendFrom = cmd.getOptionValue("from");
                    String sendTo = cmd.getOptionValue("to");
                    String sendAmount = cmd.getOptionValue("amount");
                    if (StringUtils.isBlank(sendFrom) ||
                            StringUtils.isBlank(sendTo) ||
                            !NumberUtils.isDigits(sendAmount)) {
                        help();
                    }
                    this.send(sendFrom, sendTo, Integer.valueOf(sendAmount));
                    break;
                case "printchain":
                    this.printChain();
                    break;
                case "h":
                    this.help();
                    break;
                default:
                    this.help();
            }
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            RocksDBUtils.getInstance().closeDB();
        }
    }

    /**
     * 创建创世块
     */
//        private void createBlockchainWithGenesisBlock () {
//            Blockchain.newBlockchain();
//        }

    /**
     * 创建区块链
     *
     * @param address
     */
    private void createBlockchain(String address) {
        Blockchain.createBlockchain(address);
        System.out.println("Done ! ");
    }

    /**
     * 添加区块
     *
     * @param data
     */
//        private void addBlock (String data) throws Exception {
//            Blockchain blockchain = Blockchain.newBlockchain();
//            blockchain.addBlock(data);
//        }

    /**
     * 打印出区块链中的所有区块
     */
    private void printChain() {
//            Blockchain blockchain = Blockchain.newBlockchain();
        Blockchain blockchain = null;
        try {
            blockchain = Blockchain.initBlockchainFromDB();
        } catch (Exception e) {
            e.printStackTrace();
        }

        Blockchain.BlockchainIterator iterator = blockchain.getBlockchainIterator();
        long index = 0;
        while (iterator.hashNext()) {
            Block block = iterator.next();
            System.out.println("第" + block.getHeight() + "个区块信息：");

            if (block != null) {
                boolean validate = ProofOfWork.newProofOfWork(block).validate();
                System.out.println("validate = " + validate);
                System.out.println("\tprevBlockHash: " + block.getPrevBlockHash());
//                    System.out.println("\tData: " + block.getData());
                System.out.println("\tTransaction: ");
                for (Transaction tx : block.getTransactions()) {
                    System.out.printf("\t\t 交易 ID：%s\n" , Hex.encodeHexString(tx.getTxId()));
                    System.out.println("\t\t 输入：");
                    for (TXInput in : tx.getInputs()) {
                        System.out.printf("\t\t\tTxID:%s\n" , Hex.encodeHexString(in.getTxId()));
                        System.out.printf("\t\t\tOutputIndex:%d\n" , in.getTxOutputIndex());
                        System.out.printf("\t\t\tScriptSiq:%s\n" , in.getScriptSig());

                    }
                    for (TXOutput out : tx.getOutputs()) {
                        System.out.printf("\t\t\tvalue:%d\n" , out.getValue());
                        System.out.printf("\t\t\tScriptPubKey:%s\n" , out.getScriptPubKey());
                    }

                }

                System.out.println("\tHash: " + block.getHash());
                SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
                String date = sdf.format(new Date(block.getTimeStamp() * 1000L));
                System.out.println("\ttimeStamp:" + date);
                System.out.println();
            }
        }
    }

    /**
     * 查询钱包余额
     *
     * @param address 钱包地址
     */
    private void getBalance(String address) throws Exception {
        Blockchain blockchain = Blockchain.createBlockchain(address);
        TXOutput[] txOutputs = blockchain.findUTXO(address);
        int balance = 0;
        if (txOutputs != null && txOutputs.length > 0) {
            for (TXOutput txOutput : txOutputs) {
                balance += txOutput.getValue();
            }
        }
        System.out.printf("Balance of '%s': %d\n", address, balance);
    }

    /**
     * 转账
     *
     * @param from
     * @param to
     * @param amount
     */
    private void send(String from, String to, int amount) throws Exception {
        Blockchain blockchain = Blockchain.createBlockchain(from);
        Transaction transaction = Transaction.newUTXOTransaction(from, to, amount, blockchain);
        blockchain.mineBlock(new Transaction[]{transaction});
        RocksDBUtils.getInstance().closeDB();
        System.out.println("Success!");
    }
} 
```

#### 3.2.9 修改`blockchain.sh`脚本文件

最后修改`blockchain.sh`脚本文件，修改后内容如下：

```java
#!/bin/bash

set -e

# Check if the jar has been built.
if [ ! -e target/part5_Transaction-jar-with-dependencies.jar ]; then
  echo "Compiling blockchain project to a JAR"
  mvn package -DskipTests
fi

java -jar target/part5_Transaction-jar-with-dependencies.jar "$@" 
```

## 4\. Transaction 讲解

### 4.1 交易 Transaction

**比特币交易**

如果你开发过 Web 应用程序，为了实现支付系统，你可能会在数据库中创建一些数据库表：`账户` 和 `交易记录`。账户用于存储用户的个人信息以及账户余额等信息，交易记录用于存储资金从一个账户转移到另一个账户的记录。但是在比特币中，支付系统是以一种完全不一样的方式实现的，在这里：

*   没有账户
*   没有余额
*   没有地址
*   没有 Coins（币）
*   没有发送者和接受者

由于区块链是一个公开的数据库，我们不希望存储有关钱包所有者的敏感信息。`Coins` 不会汇总到钱包中。交易不会将资金从一个地址转移到另一个地址。没有可保存帐户余额的字段或属性。只有交易信息。那比特币的交易信息里面到底存储的是什么呢？

我们来详细说明一下：

1、交易，简单地说就是把比特币从一个地址转到另一个地址，准确地说，一笔交易指一个经过签名运算的，表达价值转移的数据结构；

2、交易实质上是包含了一组输入列表和输出列表的数据结构，也就是转账记录。

*   比特币交易都是由 inputs（输入）、outputs（输出）组成。可以简单理解为发币地址是输入、收币地址是输出。
*   inputs 用来追溯上一笔交易，以便明确转出者是否有权动用这笔钱，outputs 用来进行一次新的加密，加密后只有收款者才能解密并动用这笔钱。
*   输入中包含解锁脚本（unlocking script），输出中包含锁定脚本（locking script）。
*   锁定脚本往往含有一个公钥或比特币地址，所以曾经被称为公钥脚本，在代码中常用 scriptPubKey 表示。
*   解锁脚本往往是支付方用自己的私钥所做的签名，曾被称为签名脚本，代码中用 scriptSig 表示。
*   在一个交易中，锁定脚本相当于是加密难题，解锁脚本是解开锁定脚本的题解。但解锁脚本的题解是针对上一笔交易输出中产生的加密难题的题解，而并非当前交易。

![`img.kongyixueyuan.com/0615_%E4%BA%A4%E6%98%93.png`](img/951eb80e1d750db927ead59a4940976a.jpg)

> *   交易 a 中，A 转账给 B
> *   交易 b 中，B 转账给 C
> *   交易 c 中，C 转账给 D
> *   在交易 a 中，当 A 给 B 转账时，A 给 B 出了一道加密难题，以脚本的形式附加在了转账金额末尾，该脚本锁定了其中的资产。
> *   在交易 b 中，B 要转账给 C，需要花费当初 A 转给他的资产。作为条件，B 必须解开 A 给他出的加密难题才能花费其中被锁定的资产。所以 b 交易中的输入中必须有解锁脚本。该解锁脚本用来解交易 a 中的锁定脚本。
> *   在交易 b 中，当 B 给 C 转账时，B 给 C 同样出了一道加密难题，将其中的资产进行了锁定。
> *   在交易 d 中，C 要转账给 D，就需要在输入中有解锁脚本。该解锁脚本是用来解交易 b 中的锁定脚本。

由于比特币采用的是 UTXO 模型，并非账户模型，并不直接存在“余额”这个概念，余额需要通过遍历整个交易历史得来。

**比特币交易**：(点击 [这里](https://blockchain.info/zh-cn/tx/b6f6b339b546a13822192b06ccbdd817afea5311845f769702ae2912f7d94ab5) 在 blockchain.info 查看下图中的交易信息。)

![`img.kongyixueyuan.com/0603_%E6%AF%94%E7%89%B9%E5%B8%81%E4%BA%A4%E6%98%93.png`](img/bab08a6bd0587f86d452342a66503122.jpg)

从上图可以看出，比特币中的交易，都是由一些输入（input）和输出（output）组合而来。

在`Transaction.java`文件中，设置 Transactio 类的属性。

```java
public class Transaction {
    private static final int SUBSIDY = 10;

    /**
     * 交易的 Hash
     */
    private byte[] txId;
    /**
     * 交易输入
     */
    private TXInput[] inputs;
    /**
     * 交易输出
     */
    private TXOutput[] outputs;
} 
```

对于每一笔新的交易，它的输入会引用（reference）之前一笔交易的输出（这里有个例外，coinbase 交易），引用就是花费的意思。所谓引用之前的一个输出，也就是将之前的一个输出包含在另一笔交易的输入当中，就是花费之前的交易输出。交易的输出，就是币实际存储的地方。下面的图示阐释了交易之间的互相关联：

![`img.kongyixueyuan.com/0604_inputoutput.png`](img/f3eb4ca1b796376c50d139e3f52affef.jpg)

注意：

1.  有一些输出并没有被关联到某个输入上
2.  一笔交易的输入可以引用之前多笔交易的输出
3.  一个输入必须引用一个输出

贯穿本文，我们将会使用像“钱（money）”，“币（coin）”，“花费（spend）”，“发送（send）”，“账户（account）” 等等这样的词。但是在比特币中，其实并不存在这样的概念。交易仅仅是通过一个脚本（script）来锁定（lock）一些值（value），而这些值只可以被锁定它们的人解锁（unlock）。

每一笔比特币交易都会创造输出，输出都会被区块链记录下来。给某个人发送比特币，实际上意味着创造新的 UTXO 并注册到那个人的地址，可以为他所用。

### 4.2 交易输出

在`TXOutput.java`中，添加 TXOutput 类的属性。

```java
public class TXOutput {

    /**
     * 数值金额
     */
    private int value;
    /**
     * 锁定脚本
     */
    private String scriptPubKey;
} 
```

输出主要包含两部分：

1.  一定量的比特币(`value`)
2.  一个锁定脚本(`scriptPubKey`)，要花这笔钱，必须要解锁该脚本。

实际上，正是输出里面存储了“币”（注意，也就是上面的 `Value` 字段）。而这里的存储，指的是用一个数学难题对输出进行锁定，这个难题被存储在 `scriptPubKey` 里面。在内部，比特币使用了一个叫做 *Script* 的脚本语言，用它来定义锁定和解锁输出的逻辑。虽然这个语言相当的原始（这是为了避免潜在的黑客攻击和滥用而有意为之），并不复杂，但是我们也并不会在这里讨论它的细节。

> 在比特币中，`value` 字段存储的是 *satoshi* 的数量，而不是 BTC 的数量。一个 *satoshi* 等于一亿分之一的 BTC(0.00000001 BTC)，这也是比特币里面最小的货币单位（就像是 1 分的硬币）。

由于还没有实现地址（address），所以目前我们会避免涉及逻辑相关的完整脚本。`ScriptPubKey` 将会存储一个任意的字符串（用户定义的钱包地址）。

> 顺便说一下，有了一个这样的脚本语言，也意味着比特币其实也可以作为一个智能合约平台。

关于输出，非常重要的一点是：它们是**不可再分的（indivisible）**。也就是说，你无法仅引用它的其中某一部分。要么不用，如果要用，必须一次性用完。当一个新的交易中引用了某个输出，那么这个输出必须被全部花费。如果它的值比需要的值大，那么就会产生一个找零，找零会返还给发送方。这跟现实世界的场景十分类似，当你想要支付的时候，如果一个东西值 1 美元，而你给了一个 5 美元的纸币，那么你会得到一个 4 美元的找零。

### 4.3 交易输入

在`TXInput.java`中，添加 TXInput 类的属性。

```java
public class TXInput {

    /**
     * 交易 Id 的 hash 值
     */
    private byte[] txId;
    /**
     * 交易输出索引
     */
    private int txOutputIndex;
    /**
     * 解锁脚本
     */
    private String scriptSig;
} 
```

正如之前所提到的，一个输入引用了之前交易的一个输出：

`txId` 存储的是之前交易的 ID，

`txOutputIndex` 存储的是该输出在那笔交易中所有输出的索引（因为一笔交易可能有多个输出，需要有信息指明是具体的哪一个）。

`scriptSig` 是一个脚本，提供了可解锁输出结构里面 `scriptPubKey` 字段的数据。如果 `scriptSig` 提供的数据是正确的，那么输出就会被解锁，然后被解锁的值就可以被用于产生新的输出；如果数据不正确，输出就无法被引用在输入中，或者说，无法使用这个输出。这种机制，保证了用户无法花费属于其他人的币。

再次强调，由于我们还没有实现地址，所以目前 `scriptSig` 将仅仅存储一个用户自定义的任意钱包地址。我们会在下一篇文章中实现公钥（public key）和签名（signature）。

来简要总结一下。输出，就是 “币” 存储的地方。每个输出都会带有一个解锁脚本，这个脚本定义了解锁该输出的逻辑。每笔新的交易，必须至少有一个输入和输出。一个输入引用了之前一笔交易的输出，并提供了解锁数据（也就是 `scriptSig` 字段），该数据会被用在输出的解锁脚本中解锁输出，解锁完成后即可使用它的值去产生新的输出。

每一笔输入都是之前一笔交易的输出，那么假设从某一笔交易开始不断往前追溯，它所涉及的输入和输出到底是谁先存在呢？换个说法，这是个鸡和蛋谁先谁后的问题，是先有蛋还是先有鸡呢？

### 4.4 CoinBase 交易

在比特币中，是先有蛋，然后才有鸡。输入引用输出的逻辑，是经典的“蛋还是鸡”问题：输入先产生输出，然后输出使得输入成为可能。在比特币中，最先有输出，然后才有输入。换而言之，第一笔交易只有输出，没有输入。

当矿工挖出一个新的块时，它会向新的块中添加一个 **coinbase** 交易。coinbase 交易是一种特殊的交易，它不需要引用之前一笔交易的输出。它“凭空”产生了币（也就是产生了新币），这是矿工获得挖出新块的奖励，也可以理解为“发行新币”。

在区块链的最初，也就是第一个块，叫做创世块。正是这个创世块，产生了区块链最开始的输出。对于创世块，不需要引用之前的交易输出。因为在创世块之前根本不存在交易，也就没有不存在交易输出。

在`Transaction.java`文件 中，添加一个方法，用于创建一个 coinbase 交易，代码如下：

```java
 /**
     * 创建 CoinBase 交易
     *
     * @param to   收账的钱包地址
     * @param data 解锁脚本数据
     * @return
     */
    public static Transaction newCoinbaseTX(String to, String data) {
        if (StringUtils.isBlank(data)) {
            data = String.format("Reward to '%s'", to);
        }
        // 创建交易输入
        TXInput txInput = new TXInput(new byte[]{}, -1, data);
        // 创建交易输出
        TXOutput txOutput = new TXOutput(SUBSIDY, to);
        // 创建交易
        Transaction tx = new Transaction(null, new TXInput[]{txInput}, new TXOutput[]{txOutput});
        // 设置交易 ID
        tx.setTxId();
        return tx;
    }

   /**
     * 设置交易 ID
     */
    private void setTxId() {
        this.setTxId(DigestUtils.sha256(SerializeUtils.serialize(this)));
    } 
```

coinbase 交易只有一个输出，没有输入。在我们的实现中，它表现为 `txId` 为空，`txOutputIndex` 等于 -1。并且，在当前实现中，coinbase 交易也没有在 `scriptSig` 中存储脚本，而只是存储了一个任意的字符串 `data`。

> 在比特币中，第一笔 coinbase 交易包含了如下信息：“The Times 03/Jan/2009 Chancellor on brink of second bailout for banks”。[可点击这里查看](https://blockchain.info/tx/4a5e1e4baab89f3a32518a88c31bc87f618f76673e2cc77ab2127b7afdeda33b?show_adv=true).

`subsidy` 是挖出新块的奖励金。在比特币中，实际并没有存储这个数字，而是基于区块总数进行计算而得：区块总数除以 210000 就是 `subsidy`。挖出创世块的奖励是 50 BTC，每挖出 `210000` 个块后，奖励减半。在我们的实现中，这个奖励值将会是一个常量，只是目前我们代码中还没有加入挖矿奖励金。

### 4.5 将交易保存到区块链

从现在开始，每个块必须存储至少一笔交易。如果没有交易，也就不可能出新的块。这意味着我们应该移除 `Block` 的 `data`字段，取而代之的是存储交易。

在`Block.java`文件中，修改 Block 类的属性，代码如下:

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

} 
```

接下来，`newBlock()` 和 `newGenesisBlock()` 也必须做出相应改变，修改后代码如下：

```java
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
```

以及如下:

```java
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
```

接下来，我们修改 createBlockchain()方法，现在，这个方法会接受一个地址作为参数，这个地址将会被用来接收挖出创世块的奖励。

```java
 /**
     * 创建区块链，createBlockchain
     * @param address
     * @return
     */
    public static Blockchain createBlockchain(String address) {

        String lastBlockHash = RocksDBUtils.getInstance().getLastBlockHash();
        if (StringUtils.isBlank(lastBlockHash)){
            //对应的 bucket 不存在，说明是第一次获取区块链实例
            // 创建 coinBase 交易
            Transaction coinbaseTX = Transaction.newCoinbaseTX(address, "");
            Block genesisBlock = Block.newGenesisBlock(coinbaseTX);
//            Block genesisBlock = Block.newGenesisBlock();
            lastBlockHash = genesisBlock.getHash();
            RocksDBUtils.getInstance().putBlock(genesisBlock);
            RocksDBUtils.getInstance().putLastBlockHash(lastBlockHash);

        }
        return new Blockchain(lastBlockHash);
    } 
```

接下来在`BlockChain.java`文件中，添加挖掘新区块的方法：

```java
/**
     * 打包交易，进行挖矿
     *
     * @param transactions
     */
    public void mineBlock(List<Transaction> transactions) throws Exception {
        String lastBlockHash = RocksDBUtils.getInstance().getLastBlockHash();
        Block lastBlock = RocksDBUtils.getInstance().getBlock(lastBlockHash);
        if (lastBlockHash == null) {
            throw new Exception("ERROR: Fail to get last block hash ! ");
        }
        Block block = Block.newBlock(lastBlockHash, transactions,lastBlock.getHeight()+1);
        this.addBlock(block);
    } 
```

### 4.6 工作量证明

工作量证明算法必须要将存储在区块里面的交易考虑进去，从而保证区块链交易存储的一致性和可靠性。所以，我们必须修改`ProofOfWork.java`文件中的 `prepareData()`方法，修改后代码如下：

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
//                this.getBlock().getData().getBytes(),
                this.getBlock().hashTransaction(),
                ByteUtils.toBytes(this.getBlock().getTimeStamp()),
                ByteUtils.toBytes(TARGET_BITS),
                ByteUtils.toBytes(nonce)
        );

    } 
```

接下来我们需要在 Block.java 文件中，添加一个 Block 对象的方法，hashTransactions()，代码如下：

```java
 /**
     * 对区块中的交易信息进行 Hash 计算
     *
     * @return
     */
    public byte[] hashTransaction() {
        byte[][] txIdArrays = new byte[this.getTransactions().size()][];
        for (int i = 0; i < this.getTransactions().size(); i++) {
            txIdArrays[i] = this.getTransactions().get(i).getTxId();
        }
        return DigestUtils.sha256(ByteUtils.merge(txIdArrays));

    } 
```

通过哈希提供数据的唯一表示，这种做法我们已经不是第一次遇到了。我们想要通过仅仅一个哈希，就可以识别一个块里面的所有交易。为此，先获得每笔交易的哈希，然后将它们关联起来，最后获得一个连接后的组合哈希。

> 比特币使用了一个更加复杂的技术：它将一个块里面包含的所有交易表示为一个 [Merkle tree](https://en.wikipedia.org/wiki/Merkle_tree) ，然后在工作量证明系统中使用树的根哈希（root hash）。这个方法能够让我们快速检索一个块里面是否包含了某笔交易，即只需 root hash 而无需下载所有交易即可完成判断。

来检查一下到目前为止是否正确：

首先先创建一个创世区块，运行效果如下：

![`img.kongyixueyuan.com/007_002_%E5%88%9B%E4%B8%96%E8%BF%90%E8%A1%8C.png`](img/902050303012e1f18fc3c3ac5712e40f.jpg)

打印区块信息，效果如下:

![`img.kongyixueyuan.com/007_003_%E6%89%93%E5%8D%B0%E5%8C%BA%E5%9D%97.png`](img/63209b99ba7187d2623e22fd53c2c5a1.jpg)

很好！我们已经获得了第一笔挖矿奖励。

### 4.7 发送币

现在，我们想要给其他人发送一些币。为此，我们需要创建一笔新的交易，将它放到一个块里，然后挖出这个块。之前我们只实现了 coinbase 交易（这是一种特殊的交易），现在我们需要一种通用的普通交易。这部分相当的复杂，接下来我们一步一步来实现。

之前已经提到过，每笔交易，都包含输入和输出。输入要引用之前的未花费的输出。

我们模拟这样一个场景：我们用 hanru 的地址创建一个创世区块，创建 CoinBase 交易后，hanru 有 10 个 Token。然后我们进行转账，hanru 转给 wangergou4 个 Token。

![`img.kongyixueyuan.com/0607_%E8%BD%AC%E8%B4%A6.png`](img/16d54b51f501c38173c05a2f14f68dfb.jpg)

> 说明：
> 
> 1.因为创建了创世区块，所以产生了 CoinBase 交易，它的 Input 是空的，Output 表示未花费的输出。理解为 hanru 有 10 个 Token(Unspent)。
> 
> 2.hanru 给 wangergou 转账，那么就表示 hanru 要花费掉自己的未花费的 output，创建 input，产生新的 output，创建普通交易。
> 
> 3.Input 中的 TxID 字段，表示引用的未花费的 output 所在的交易 ID，Vout 表示引用的未花费的 output 在所在交易的下标(我们 Transaction 中的 output 采用数组存储)。ScriptSiq 目前仅仅当做账户名(实际上并没有账户名这个东西，因为我们没有学习地址，暂且这样理解)。

综上，要想实现真正的转账，就要找到该账户下的未花费的 output，但是因为 output 中仅仅设置了 Value 和 ScriptPubKey 两个字段，所以我们在转账的时候要知道该 output 所在的交易 ID 和下标。

### 4.8 查找未花费的 UTXO

不管要实现转账还是查询余额，我们需要找出某个账户下所有的未花费交易输出（unspent transactions outputs, UTXO）。**未花费（unspent）** 指的是这个输出还没有被包含在任何交易的输入中，或者说没有被任何输入引用。

> UTXO：unspend transaction output.（未被花费的交易输出）
> 
> 在比特币的世界里既没有账户，也没有余额，只有分散到区块链里的 UTXO.

*UTXO* 是理解比特币交易原理的关键所在，我们先来看一段场景：

场景：假设你过去分别向 A、B、C 这三个比特币用户购买了 BTC，从 A 手中购买了 3.5 个 BTC，从 B 手中购买了 4.5 个 BTC，从 C 手中购买了 2 个 BTC，现在你的比特币钱包里面恰好剩余 10 个 BTC。

问题：这个 10 个 BTC 是真正的 10 个 BTC 吗？其实不是，这句话可能听起来有点怪。（什么！我钱包里面的 BTC 不是真正的 BTC，你不要吓我……）

解释：前面提到过在比特币的交易系统当中，并不存在账户、余额这些概念，所以，你的钱包里面的 10 个 BTC，并不是说钱包余额为 10 个 BTC。而是说，这 10 个 BTC 其实是由你的比特币地址（钱包地址|公钥）锁定了的散落在各个区块和各个交易里面的 UTXO 的总和。

UTXO 是比特币交易的基本单位，每笔交易都会产生 UTXO，一个 UTXO 可以是一“聪”的任意倍。给某人发送比特币实际上是创造新的 UTXO，绑定到那个人的钱包地址，并且能被他用于新的支付。

一般的比特币交易由 `交易输入` 和 `交易输出` 两部分组成。A 向你支付 3.5 个 BTC 这笔交易，实际上产生了一个新的 UTXO，这个新的 UTXO 等于 3.5 个 BTC（3.5 亿聪），并且锁定到了你的比特币钱包地址上。

假如你要给你女（男）朋友转 1.5 BTC，那么你的钱包会从可用的 UTXO 中选取一个或多个可用的个体来拼凑出一个大于或等于一笔交易所需的比特币量。比如在这个假设场景里面，你的钱包会选取你和 C 的交易中的 UTXO 作为 交易输入，input = 2BTC，这里会生成两个新的交易输出，一个输出（UTXO = 1.5 BTC）会被绑定到你女（男）朋友的钱包地址上，另一个输出（UTXO = 0.5 BTC）会作为找零，重新绑定到你的钱包地址上。

我们需要找到所有未花费的交易输出（UTXO）。*Unspent(未花费)* 意味着这些交易输出从未被交易输入所指向。

我们先说明一下：无论是转账还是查询余额时，我们并不需要知道整个区块链上所有的 UTXO，只需要关注那些我们能够解锁的那些 UTXO（目前我们还没有实现密钥，所以我们将会使用用户定义的地址来代替）。首先，让我们定义在输入和输出上的锁定和解锁方法：

```java
 /**
     * TXInput
     * 判断解锁数据是否能够解锁交易输出
     *
     * @param unlockingData
     * @return
     */
    public boolean canUnlockOutputWith(String unlockingData) {
        return this.getScriptSig().endsWith(unlockingData);
    } 
```

和：

```java
 /**
     *    TXOutput 
     *    判断解锁数据是否能够解锁交易输出
     *
     * @param unlockingData
     * @return
     */
    public boolean canBeUnlockedWith(String unlockingData) {
        return this.getScriptPubKey().endsWith(unlockingData);
    }
} 
```

在这里，我们只是将 script 字段与 address 进行了比较。在后续文章我们基于私钥实现了地址以后，会对这部分进行改进。

好了，现在我们针对于查找未花费的 UTXO 算法，一步一步进行说明，其实这一步相当困难。接下来的两个方法，用于找出未花费的交易。

首先遍历数据库，获取每个 Block 区块。由于交易被存储在区块里，所以我们不得不检查区块链里的每个 Block 中的每一笔交易。对于该笔交易，依次记录里面已经花费 TXInput，然后根据 TXInput，找出未花费的 Output，记录该交易。

在`BlockChain.java`文件中，添加 getAllSpentTXOs()方法，用于查找所有已经花费的 Output。代码如下:

```java
 /**
     * 从交易输入中查询区块链中所有已被花费了的交易输出
     *
     * @param address 钱包地址
     * @return 交易 ID 以及对应的交易输出下标地址
     * @throws Exception
     */
    private Map<String, int[]> getAllSpentTXOs(String address) {
        // 定义 TxId ——> spentOutIndex[]，存储交易 ID 与已被花费的交易输出数组索引值
        Map<String, int[]> spentTXOs = new HashMap<>();
        for (BlockchainIterator blockchainIterator = this.getBlockchainIterator(); blockchainIterator.hashNext(); ) {
            Block block = blockchainIterator.next();

            for (Transaction transaction : block.getTransactions()) {
                // 如果是 coinbase 交易，直接跳过，因为它不存在引用前一个区块的交易输出
                if (transaction.isCoinbase()) {
                    continue;
                }
                for (TXInput txInput : transaction.getInputs()) {
                    if (txInput.canUnlockOutputWith(address)) {
                        String inTxId = Hex.encodeHexString(txInput.getTxId());
                        int[] spentOutIndexArray = spentTXOs.get(inTxId);
                        if (spentOutIndexArray == null) {
                            spentTXOs.put(inTxId, new int[]{txInput.getTxOutputIndex()});
                        } else {
                            spentOutIndexArray = ArrayUtils.add(spentOutIndexArray, txInput.getTxOutputIndex());
                            spentTXOs.put(inTxId, spentOutIndexArray);
                        }
                    }
                }
            }
        }
        return spentTXOs;
    } 
```

首先创建一个 map(`Map<String, int[]> spentTXOs = new HashMap<>();`)，用于存储该交易中的所有的 Input 信息，表示已经花费。

```java
for (Transaction transaction : block.getTransactions()) {
    if (transaction.isCoinbase()) {
        continue;
    }
    ...
} 
```

因为 CoinBase 交易中没有 Input，所以我们需要判断是否是 CoinBase 交易，如果不是那么要存储该交易中的 Input 信息。

```java
for (TXInput txInput : transaction.getInputs()) {
    if (txInput.canUnlockOutputWith(address)) {
        String inTxId = Hex.encodeHexString(txInput.getTxId());
        int[] spentOutIndexArray = spentTXOs.get(inTxId);
        if (spentOutIndexArray == null) {
            spentTXOs.put(inTxId, new int[]{txInput.getTxOutputIndex()});
        } else {
            spentOutIndexArray = ArrayUtils.add(spentOutIndexArray, txInput.getTxOutputIndex());
            spentTXOs.put(inTxId, spentOutIndexArray);
        }
    }
} 
```

然后判断交易中的每个 Input，如果该 Input 被一个地址锁定，并且这个地址恰好是我们要找的地址，那么这个输入就是我们想要的。我们需要将它存入到 map 中。

接下来，我们在`Blockchain.java`中，添加`findUnspentTransactions()`方法，用于获取所有未花费的交易。代码如下:

```java
 /**
     * 查找钱包地址对应的所有未花费的交易
     *
     * @param address 钱包地址
     * @return
     */
    private Transaction[] findUnspentTransactions(String address) throws Exception {
        Map<String, int[]> allSpentTXOs = this.getAllSpentTXOs(address);
        Transaction[] unspentTxs = {};

        // 再次遍历所有区块中的交易输出
        for (BlockchainIterator blockchainIterator = this.getBlockchainIterator(); blockchainIterator.hashNext(); ) {
            Block block = blockchainIterator.next();
            for (Transaction transaction : block.getTransactions()) {

                String txId = Hex.encodeHexString(transaction.getTxId());

                int[] spentOutIndexArray = allSpentTXOs.get(txId);

                for (int outIndex = 0; outIndex < transaction.getOutputs().length; outIndex++) {
                    if (spentOutIndexArray != null && ArrayUtils.contains(spentOutIndexArray, outIndex)) {
                        continue;
                    }

                    // 保存不存在 allSpentTXOs 中的交易
                    if (transaction.getOutputs()[outIndex].canBeUnlockedWith(address)) {
                        unspentTxs = ArrayUtils.add(unspentTxs, transaction);
                    }
                }
            }
        }
        return unspentTxs;
    } 
```

接下来我们重新遍历每一个区块，获取所有的交易，根据 txId，获取 allSpentTXOs 这个 map 中对应的数组。我们需要判断交易中的每个 Output，如果该 Output 被一个地址锁定，并且这个地址恰好是我们要找的地址，那么这个输出就是我们想要的。不过在获取它之前，我们需要对比存储了 Input 信息的 map，检查该输出是否已经被包含在一个交易的输入中，也就是检查它是否已经被花费了。

如果 output 的下标以及当前交易的 TxID，和 map 中存储的信息没有对应，那么表示该 Output 没有被花费，那么我们需要存储该 Output 所在的交易。

接下来，创建一个类，用于表示本次交易需要的信息。我们在`cldy.hanru.blockchain.transaction`包下，新建一个 java 文件，命名为`SpendableOutputResult.java`，并添加代码如下：

```java
package cldy.hanru.blockchain.transaction;

import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.util.Map;

/**
 * @author hanru
 */
@AllArgsConstructor
@NoArgsConstructor
@Data
public class SpendableOutputResult {

    /**
     * 交易时的支付金额
     */
    private int accumulated;
    /**
     * 未花费的交易
     */
    private Map<String, int[]> unspentOuts;
} 
```

该类用于存储一次转账所需要的花费的 UTXO，以及这些 UTXO 的金额总额。

最后，在`Blockchain.java`中再添加一个方法，`findSpendableOutputs()`用于查找本次交易需要的 SpendableOutputResult，代码如下：

```java
/**
     * 寻找能够花费的交易
     *
     * @param address 钱包地址
     * @param amount  花费金额
     */
    public SpendableOutputResult findSpendableOutputs(String address, int amount) throws Exception {
        Transaction[] unspentTXs = this.findUnspentTransactions(address);
        int accumulated = 0;
        Map<String, int[]> unspentOuts = new HashMap<>();
        for (Transaction tx : unspentTXs) {

            String txId = Hex.encodeHexString(tx.getTxId());

            for (int outId = 0; outId < tx.getOutputs().length; outId++) {

                TXOutput txOutput = tx.getOutputs()[outId];

                if (txOutput.canBeUnlockedWith(address) && accumulated < amount) {
                    accumulated += txOutput.getValue();

                    int[] outIds = unspentOuts.get(txId);
                    if (outIds == null) {
                        outIds = new int[]{outId};
                    } else {
                        outIds = ArrayUtils.add(outIds, outId);
                    }
                    unspentOuts.put(txId, outIds);
                    if (accumulated >= amount) {
                        break;
                    }
                }
            }
        }
        return new SpendableOutputResult(accumulated, unspentOuts);
    } 
```

我们使用 accumulated 字段来表示本次转账要使用的 UTXO 的金额总和。再定义`Map<String, int[]> unspentOuts = new HashMap<>();`用于存储本次转账要使用的 Output。

首先根据 address，找出可以本次转账可以使用的交易数组 unspentTXs。遍历该数组，对于每一个交易，我们比较里面的每个 Output。如果改 Output 可以被指定的 address 解锁，并且 accumulated 的值小于本次要转行的金额 amount，那么表示本次转账可以使用这个 Output，那么需要将它存入对应的 map 里。

### 4.9 创建转账交易

根据终端命令获取到转账信息，比如`send -from hanru -to wangergou -amount 4`，表示韩茹要转账给王二狗 4 个 Token。那么我们可以根据上一步的方法`findSpendableOutputs()`找出本次转账要使用的 SpendableOutputResult 实例。

```java
/**
     * 从 from 向  to 支付一定的 amount 的金额
     *
     * @param from       支付钱包地址
     * @param to         收款钱包地址
     * @param amount     交易金额
     * @param blockchain 区块链
     * @return
     */
    public static Transaction newUTXOTransaction(String from, String to, int amount, Blockchain blockchain) throws Exception {
        SpendableOutputResult result = blockchain.findSpendableOutputs(from, amount);
        int accumulated = result.getAccumulated();
        Map<String, int[]> unspentOuts = result.getUnspentOuts();

        if (accumulated < amount) {
            throw new Exception("ERROR: Not enough funds");
        }
        Iterator<Map.Entry<String, int[]>> iterator = unspentOuts.entrySet().iterator();

        TXInput[] txInputs = {};
        while (iterator.hasNext()) {
            Map.Entry<String, int[]> entry = iterator.next();
            String txIdStr = entry.getKey();
            int[] outIdxs = entry.getValue();
            byte[] txId = Hex.decodeHex(txIdStr);
            for (int outIndex : outIdxs) {
                txInputs = ArrayUtils.add(txInputs, new TXInput(txId, outIndex, from));
            }
        }

        TXOutput[] txOutput = {};
        txOutput = ArrayUtils.add(txOutput, new TXOutput(amount, to));
        if (accumulated > amount) {
            txOutput = ArrayUtils.add(txOutput, new TXOutput((accumulated - amount), from));
        }

        Transaction newTx = new Transaction(null, txInputs, txOutput);
        newTx.setTxId();
        return newTx;
    } 
```

根据 SpendableOutputResult 实例获取 accumulated，如果小于本次要转账的金额 amount。那么表示余额不足，无法实现本次转账。需要结束程序。

接下来，我们创建交易。

我们根据 SpendableOutputResult 实例获取 unspentOuts，获取本次转账要使用的 UTXO。然后根据这些 UTXO，创建 txInput，并添加到 txInputs 中。

然后创建 txOutput，一个是转账金额的去向，一个转账产生的找零，并添加到 txOutputs 中。

根据 txInputs，txOutputs 创建交易，并设置 TxID。

### 4.10 根据转账交易创建区块

在此处，我们直接根据转账信息，创建交易，根据交易进行挖矿产生新的区块，并将新区块添加到数据库中，表示上链。

在 BlockChain.java 文件中，添加`mineBlock()`方法，用于挖掘新的区块，代码如下 :

```java
 /**
     * 打包交易，进行挖矿
     *
     * @param transactions
     */
    public void mineBlock(List<Transaction> transactions) throws Exception {
        String lastBlockHash = RocksDBUtils.getInstance().getLastBlockHash();
        Block lastBlock = RocksDBUtils.getInstance().getBlock(lastBlockHash);
        if (lastBlockHash == null) {
            throw new Exception("ERROR: Fail to get last block hash ! ");
        }
        Block block = Block.newBlock(lastBlockHash, transactions,lastBlock.getHeight()+1);
        this.addBlock(block);
    } 
```

首先根据转账信息创建交易，查询数据库获取最后一个区块的信息，创建新的区块，并存储到数据库中。

转账，意味着创建一笔新的交易并且通过挖矿的方式将其存入区块中。但是，比特币不会像我们这样做，它会把新的交易记录先存到内存池中，当一个矿工准备去开采一个区块时，它会把打包内存池中的所有交易信息，并且创建一个候选区块。只有当这个包含所有交易信息的候选区块被成功开采并且被添加到区块链上时，这些交易信息才算被确认。

### 4.11 重新实现 CLI

现在的 CLI 相较于之前的实现，改动有点大。

首先修改 CLI 的构造函数，设置几个 Option，用于接收命令后的参数信息：

```java
public CLI(String[] args) {
        this.args = args;

        Option helpCmd = Option.builder("h").desc("show help").build();
        options.addOption(helpCmd);

        Option address = Option.builder("address").hasArg(true).desc("Source wallet address").build();
        Option sendFrom = Option.builder("from").hasArg(true).desc("Source wallet address").build();
        Option sendTo = Option.builder("to").hasArg(true).desc("Destination wallet address").build();
        Option sendAmount = Option.builder("amount").hasArg(true).desc("Amount to send").build();

        options.addOption(address);
        options.addOption(sendFrom);
        options.addOption(sendTo);
        options.addOption(sendAmount);
    } 
```

接下来，修改 run()方法：

```java
 /**
     * 命令行解析入口
     */
    public void run() {
        this.validateArgs(args);
        try {
            CommandLineParser parser = new DefaultParser();
            CommandLine cmd = parser.parse(options, args);

            switch (args[0]) {
                case "createblockchain":
                    String createblockchainAddress = cmd.getOptionValue("address");
                    if (StringUtils.isBlank(createblockchainAddress)) {
                        help();
                    }
                    this.createBlockchain(createblockchainAddress);
                    break;
                case "getbalance":
                    String getBalanceAddress = cmd.getOptionValue("address");
                    if (StringUtils.isBlank(getBalanceAddress)) {
                        help();
                    }
                    this.getBalance(getBalanceAddress);
                    break;
                case "send":
                    String sendFrom = cmd.getOptionValue("from");
                    String sendTo = cmd.getOptionValue("to");
                    String sendAmount = cmd.getOptionValue("amount");
                    if (StringUtils.isBlank(sendFrom) ||
                            StringUtils.isBlank(sendTo) ||
                            !NumberUtils.isDigits(sendAmount)) {
                        help();
                    }
                    this.send(sendFrom, sendTo, Integer.valueOf(sendAmount));
                    break;
                case "printchain":
                    this.printChain();
                    break;
                case "h":
                    this.help();
                    break;
                default:
                    this.help();
            }
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            RocksDBUtils.getInstance().closeDB();
        }
    } 
```

我们通过分支语句判断终端输入的命名，然后执行不同的功能。

接下来，添加 send()方法，用于实现转账：

```java
 /**
     * 转账
     *
     * @param from
     * @param to
     * @param amount
     */
    private void send(String from, String to, int amount) throws Exception {
        Blockchain blockchain = Blockchain.createBlockchain(from);
        Transaction transaction = Transaction.newUTXOTransaction(from, to, amount, blockchain);
        List<Transaction> transactions = new ArrayList<>();
        transactions.add(transaction);
        blockchain.mineBlock(transactions);
        RocksDBUtils.getInstance().closeDB();
        System.out.println("Success!");
    } 
```

现在我们可以测试一下了，打开终端执行以下命令：

```java
hanru:part5_Transaction ruby$ ./blockchain.sh send -from hanru -to wangergou -amount 4 
```

运行效果如下：

![`img.kongyixueyuan.com/007_004_%E8%BD%AC%E8%B4%A6.png`](img/03703d21ce67f56eebaf4f99991acf06.jpg)

查看新的区块，在终端输入以下命令：

```java
hanru:day03_05_Transaction ruby$ ./bc printchain 
```

运行效果如下：

![`img.kongyixueyuan.com/007_005_%E6%89%93%E5%8D%B0.png`](img/628504c54ff96d6a316ea4daea4c5e16.jpg)

### 4.12 查询余额

根据终端输入命令，查询指定账户的余额。

```java
hanru:part5_Transaction ruby$ ./blockchain.sh getbalance -address hanru
hanru:part5_Transaction ruby$ ./blockchain.sh getbalance -address wangergou 
```

那么就需要获取韩茹账户下的所有的未花费的 UTXO，然后累加所有的 Value，得到的总和就是余额。

首先在`BlockChain.java`中添加方法`findUTXO()`，用于查询指定账户的所有未花费的 Output，代码如下：

```java
 /**
     * 查找钱包地址对应的所有 UTXO
     *
     * @param address 钱包地址
     * @return
     */
    public TXOutput[] findUTXO(String address) throws Exception {
        Transaction[] unspentTxs = this.findUnspentTransactions(address);
        TXOutput[] utxos = {};
        if (unspentTxs == null || unspentTxs.length == 0) {
            return utxos;
        }
        for (Transaction tx : unspentTxs) {
            for (TXOutput txOutput : tx.getOutputs()) {
                if (txOutput.canBeUnlockedWith(address)) {
                    utxos = ArrayUtils.add(utxos, txOutput);
                }
            }
        }
        return utxos;
    } 
```

接下来，在 CLI.java 文件中，添加方法`getBalance()`，先获取该用户的所有的 UTXO，然后累加求和，就是余额，代码如下：

```java
 /**
     * 查询钱包余额
     *
     * @param address 钱包地址
     */
    private void getBalance(String address) throws Exception {
        Blockchain blockchain = Blockchain.createBlockchain(address);
        TXOutput[] txOutputs = blockchain.findUTXO(address);
        int balance = 0;
        if (txOutputs != null && txOutputs.length > 0) {
            for (TXOutput txOutput : txOutputs) {
                balance += txOutput.getValue();
            }
        }
        System.out.printf("Balance of '%s': %d\n", address, balance);
    } 
```

现在可以进行代码测试，在终端输入以下命令：

```java
hanru:part5_Transaction ruby$ ./blockchain.sh getbalance -address hanru

hanru:part5_Transaction ruby$ ./blockchain.sh getbalance -address wangergou 
```

运行效果如下：

![`img.kongyixueyuan.com/007_006_%E4%BD%99%E9%A2%9D.png`](img/7cc76629465a79bade53e9a26679ce32.jpg)

## 5\. 总结

通过本章节的学习，我们知道了什么是转账交易的输入，输出，以及转账的原理。为了能够实现转账交易和查询余额，我们需要学习 UTXO 模型，以及统计未花费的 Output 的算法。

虽然不容易，但是现在终于实现交易了！不过，我们依然缺少了一些像比特币那样的一些关键特性：

1.  地址（address）。我们还没有基于私钥（private key）的真实地址。
2.  奖励（reward）。现在挖矿是肯定无法盈利的！
3.  UTXO 集。获取余额需要扫描整个区块链，而当区块非常多的时候，这么做就会花费很长时间。并且，如果我们想要验证后续交易，也需要花费很长时间。而 UTXO 集就是为了解决这些问题，加快交易相关的操作。
4.  内存池（mempool）。在交易被打包到块之前，这些交易被存储在内存池里面。在我们目前的实现中，一个块仅仅包含一笔交易，这是相当低效的。

[项目源代码](https://github.com/rubyhan1314/BitcoinForJava)