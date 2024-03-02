# 第十章 UTXO 集优化

# 第十章 交易(Transactions)（2）

在这个系列文章的一开始，我们就提到了，区块链是一个分布式数据库。不过在之前的文章中，我们选择性地跳过了“分布式”这个部分，而是将注意力都放到了“数据库”部分。到目前为止，我们几乎已经实现了一个区块链数据库的所有元素(基本原型，工作量证明，持久化，命令行接口，交易，地址，数字签名等)。今天，我们将会分析之前跳过的一些机制。而在下一篇文章中，我们将会开始讨论区块链的分布式特性。

> 本文的代码实现变化很大。

## 1\. 课程目标

1.  知道什么是奖励
2.  知道什么是 UTXO 集
3.  学会 UTXO 集优化的原理
4.  项目中修改代码实现 UTXO 集查询余额
5.  项目中修改代码实现 UTXO 集进行转账交易

## 2\. 项目代码及效果展示

### 2.1 项目代码结构

![`img.kongyixueyuan.com/010_001_%E9%A1%B9%E7%9B%AE%E7%BB%93%E6%9E%84.png`](img/e88dcc523832040af625f39efdcd915a.jpg)

### 2.2 项目运行结果

创建钱包地址，创建创世区块效果图：

![`img.kongyixueyuan.com/010_006_%E6%95%88%E6%9E%9C%E5%9B%BE1.gif`](img/bb427de8c7c052a22d0d729a8e3fc10e.jpg)

使用 UTXOSet 优化后的转账，查询余额效果图：

![`img.kongyixueyuan.com/010_007_%E6%95%88%E6%9E%9C%E5%9B%BE2.gif`](img/d0d2e7fd39d65dce76f8324aee0fea90.jpg)

## 3\. 创建项目

###3.1 创建工程

打开 IntelliJ IDEA 的工作空间，将上一个项目代码目录`part7_Signature`，复制为`part8_Transaction2`。

然后打开 IntelliJ IDEA 开发工具。

打开工程：`part8_Transaction2`，并删除 target 目录。然后进行以下修改：

```java
step1：先将项目重新命名为：part8_Transaction2。
step2：修改 pom.xml 配置文件。
    改为：<artifactId>part8_Transaction2</artifactId>标签
    改为：<name>part8_Transaction2 Maven Webapp</name> 
```

> 说明：我们每一章节的项目代码，都是在上一个章节上进行添加。所以拷贝上一次的项目代码，然后进行新内容的添加或修改。

###

### 3.2 代码实现

#### 3.2.1 修改 java 文件：`RocksDBUtils.java`

打开`cldy.hanru.blockchain.store`包，修改`RocksDBUtils.java`文件：

修改步骤：

```java
修改步骤：
step1：添加 String CHAINSTATE_BUCKET_KEY = "chainstate";
step2：添加 private Map<String, byte[]> chainstateBucket;
step3：添加 initChainStateBucket()方法
step4：添加 cleanChainStateBucket()方法
step5：添加 putUTXOs()
step6：添加 getUTXOs()
step7：添加 deleteUTXOs()
step8：修改 RocksDBUtils()构造函数 
```

修改完后代码如下：

```java
package cldy.hanru.blockchain.store;

import cldy.hanru.blockchain.block.Block;
import cldy.hanru.blockchain.transaction.TXOutput;
import cldy.hanru.blockchain.util.SerializeUtils;
import lombok.Getter;
import lombok.extern.slf4j.Slf4j;
import org.rocksdb.RocksDB;
import org.rocksdb.RocksDBException;

import java.util.HashMap;
import java.util.Map;

/**
 * 数据库存储的工具类
 * @author hanru
 */
@Slf4j
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
     * 链状态桶 Key
     */
    private static final String CHAINSTATE_BUCKET_KEY = "chainstate";

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
        initChainStateBucket();
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

    //-------------------UTXOSet---------------------

    /**
     * chainstate buckets
     */
    @Getter
    private Map<String, byte[]> chainstateBucket;

    /**
     * 初始化 blocks 数据桶
     */
    private void initChainStateBucket() {
        try {
            byte[] chainstateBucketKey = SerializeUtils.serialize(CHAINSTATE_BUCKET_KEY);
            byte[] chainstateBucketBytes = db.get(chainstateBucketKey);
            if (chainstateBucketBytes != null) {
                chainstateBucket = (Map) SerializeUtils.deserialize(chainstateBucketBytes);
            } else {
                chainstateBucket = new HashMap<>();
                db.put(chainstateBucketKey, SerializeUtils.serialize(chainstateBucket));
            }
        } catch (RocksDBException e) {
            log.error("Fail to init chainstate bucket ! ", e);
            throw new RuntimeException("Fail to init chainstate bucket ! ", e);
        }
    }

    /**
     * 清空 chainstate bucket
     */
    public void cleanChainStateBucket() {
        try {
            chainstateBucket.clear();
        } catch (Exception e) {
            log.error("Fail to clear chainstate bucket ! ", e);
            throw new RuntimeException("Fail to clear chainstate bucket ! ", e);
        }
    }

    /**
     * 保存 UTXO 数据
     *
     * @param key   交易 ID
     * @param utxos UTXOs
     */
    public void putUTXOs(String key, TXOutput[] utxos) {
        try {
            chainstateBucket.put(key, SerializeUtils.serialize(utxos));
            db.put(SerializeUtils.serialize(CHAINSTATE_BUCKET_KEY), SerializeUtils.serialize(chainstateBucket));
        } catch (Exception e) {
            log.error("Fail to put UTXOs into chainstate bucket ! key=" + key, e);
            throw new RuntimeException("Fail to put UTXOs into chainstate bucket ! key=" + key, e);
        }
    }

    /**
     * 查询 UTXO 数据
     *
     * @param key 交易 ID
     */
    public TXOutput[] getUTXOs(String key) {
        byte[] utxosByte = chainstateBucket.get(key);
        if (utxosByte != null) {
            return (TXOutput[]) SerializeUtils.deserialize(utxosByte);
        }
        return null;
    }

    /**
     * 删除 UTXO 数据
     *
     * @param key 交易 ID
     */
    public void deleteUTXOs(String key) {
        try {
            chainstateBucket.remove(key);
            db.put(SerializeUtils.serialize(CHAINSTATE_BUCKET_KEY), SerializeUtils.serialize(chainstateBucket));
        } catch (Exception e) {
            log.error("Fail to delete UTXOs by key ! key=" + key, e);
            throw new RuntimeException("Fail to delete UTXOs by key ! key=" + key, e);
        }
    }

} 
```

#### 3.2.2 创建`UTXOSet.java`文件

打开`cldy.hanru.blockchain.transaction`包，创建`UTXOSet.java`文件。

添加代码如下：

```java
package cldy.hanru.blockchain.transaction;

import cldy.hanru.blockchain.block.Block;
import cldy.hanru.blockchain.block.Blockchain;
import cldy.hanru.blockchain.store.RocksDBUtils;
import cldy.hanru.blockchain.util.SerializeUtils;
import lombok.AllArgsConstructor;
import lombok.NoArgsConstructor;
import lombok.Synchronized;
import lombok.extern.slf4j.Slf4j;
import org.apache.commons.codec.binary.Hex;
import org.apache.commons.lang3.ArrayUtils;

import java.util.HashMap;
import java.util.Map;

/**
 * 未被花费的交易输出池
 * @author hanru
 */
@NoArgsConstructor
@AllArgsConstructor
@Slf4j
public class UTXOSet {

    private Blockchain blockchain;

    /**
     * 重建 UTXO 池索引
     */
    @Synchronized
    public void reIndex() {
        log.info("Start to reIndex UTXO set !");
        RocksDBUtils.getInstance().cleanChainStateBucket();
        Map<String, TXOutput[]> allUTXOs = blockchain.findAllUTXOs();
        for (Map.Entry<String, TXOutput[]> entry : allUTXOs.entrySet()) {
            RocksDBUtils.getInstance().putUTXOs(entry.getKey(), entry.getValue());
        }
        log.info("ReIndex UTXO set finished ! ");
    }

    /**
     * 查找钱包地址对应的所有 UTXO
     *
     * @param pubKeyHash 钱包公钥 Hash
     * @return
     */
    public TXOutput[] findUTXOs(byte[] pubKeyHash) {
        TXOutput[] utxos = {};
        Map<String, byte[]> chainstateBucket = RocksDBUtils.getInstance().getChainstateBucket();
        if (chainstateBucket.isEmpty()) {
            return utxos;
        }
        for (byte[] value : chainstateBucket.values()) {
            TXOutput[] txOutputs = (TXOutput[]) SerializeUtils.deserialize(value);
            for (TXOutput txOutput : txOutputs) {
                if (txOutput.isLockedWithKey(pubKeyHash)) {
                    utxos = ArrayUtils.add(utxos, txOutput);
                }
            }
        }
        return utxos;
    }

    /**
     * 寻找能够花费的交易
     *
     * @param pubKeyHash 钱包公钥 Hash
     * @param amount     花费金额
     */
    public SpendableOutputResult findSpendableOutputs(byte[] pubKeyHash, int amount) {
        Map<String, int[]> unspentOuts = new HashMap<>();
        int accumulated = 0;
        Map<String, byte[]> chainstateBucket = RocksDBUtils.getInstance().getChainstateBucket();
        for (Map.Entry<String, byte[]> entry : chainstateBucket.entrySet()) {
            String txId = entry.getKey();
            TXOutput[] txOutputs = (TXOutput[]) SerializeUtils.deserialize(entry.getValue());

            for (int outId = 0; outId < txOutputs.length; outId++) {
                TXOutput txOutput = txOutputs[outId];
                if (txOutput.isLockedWithKey(pubKeyHash) && accumulated < amount) {
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
     * 更新 UTXO 池
     * 当一个新的区块产生时，需要去做两件事情：
     * 1）从 UTXO 池中移除花费掉了的交易输出；
     * 2）保存新的未花费交易输出；
     *
     * @param tipBlock 最新的区块
     */
    @Synchronized
    public void update(Block tipBlock) {
        if (tipBlock == null) {
            log.error("Fail to update UTXO set ! tipBlock is null !");
            throw new RuntimeException("Fail to update UTXO set ! ");
        }
        for (Transaction transaction : tipBlock.getTransactions()) {

            // 根据交易输入排查出剩余未被使用的交易输出
            if (!transaction.isCoinbase()) {
                for (TXInput txInput : transaction.getInputs()) {
                    // 余下未被使用的交易输出
                    TXOutput[] remainderUTXOs = {};
                    String txId = Hex.encodeHexString(txInput.getTxId());
                    TXOutput[] txOutputs = RocksDBUtils.getInstance().getUTXOs(txId);

                    if (txOutputs == null) {
                        continue;
                    }

                    for (int outIndex = 0; outIndex < txOutputs.length; outIndex++) {
                        if (outIndex != txInput.getTxOutputIndex()) {
                            remainderUTXOs = ArrayUtils.add(remainderUTXOs, txOutputs[outIndex]);
                        }
                    }

                    // 没有剩余则删除，否则更新
                    if (remainderUTXOs.length == 0) {
                        RocksDBUtils.getInstance().deleteUTXOs(txId);
                    } else {
                        RocksDBUtils.getInstance().putUTXOs(txId, remainderUTXOs);
                    }
                }
            }

            // 新的交易输出保存到 DB 中
            TXOutput[] txOutputs = transaction.getOutputs();
            String txId = Hex.encodeHexString(transaction.getTxId());
            RocksDBUtils.getInstance().putUTXOs(txId, txOutputs);
        }

    }

} 
```

#### 3.2.3 修改`Blockchain.java`

打开`cldy.hanru.blockchain.block`包，修改`Blockchain.java`文件：

修改步骤：

```java
修改步骤：
step1：修改 getAllSpentTXOs()方法
step2：添加 findAllUTXOs()方法
step3：删除 findUnspentTransactions()
step4：删除 findUTXO()
step5：删除 findSpendableOutputs()
step6：修改 mineBlock(),添加返回值
step7：修改 verifyTransactions()方法，添加判断是否是 coinbase 交易 
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
import org.bouncycastle.jcajce.provider.asymmetric.ec.BCECPrivateKey;

import java.util.Arrays;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

/**
 * 区块链
 *
 * @author hanru
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
     *
     * @param address
     * @return
     */
    public static Blockchain createBlockchain(String address) {

        String lastBlockHash = RocksDBUtils.getInstance().getLastBlockHash();
        if (StringUtils.isBlank(lastBlockHash)) {
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
     *
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
    public class BlockchainIterator {

        /**
         * 当前区块的 hash
         */
        private String currentBlockHash;

        /**
         * 构造函数
         *
         * @param currentBlockHash
         */
        public BlockchainIterator(String currentBlockHash) {
            this.currentBlockHash = currentBlockHash;
        }

        /**
         * 判断是否有下一个区块
         *
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
         *
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
     *
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
    public Block mineBlock(List<Transaction> transactions) throws Exception {
        // 挖矿前，先验证交易记录
        for (Transaction tx : transactions) {
            if (!this.verifyTransactions(tx)) {
                throw new Exception("ERROR: Fail to mine block ! Invalid transaction ! ");
            }
        }
        String lastBlockHash = RocksDBUtils.getInstance().getLastBlockHash();
        Block lastBlock = RocksDBUtils.getInstance().getBlock(lastBlockHash);
        if (lastBlockHash == null) {
            throw new Exception("ERROR: Fail to get last block hash ! ");
        }
        Block block = Block.newBlock(lastBlockHash, transactions,lastBlock.getHeight()+1);
        this.addBlock(block);
        return block;

    }

    /**
     * 查找钱包地址对应的所有未花费的交易
     *
     * @param pubKeyHash 钱包公钥 Hash
     * @return
     */
    /*
    private Transaction[] findUnspentTransactions(byte[] pubKeyHash) throws Exception {
        Map<String, int[]> allSpentTXOs = this.getAllSpentTXOs(pubKeyHash);
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
//                    if (transaction.getOutputs()[outIndex].canBeUnlockedWith(address)) {
                    if (transaction.getOutputs()[outIndex].isLockedWithKey(pubKeyHash)) {
                        unspentTxs = ArrayUtils.add(unspentTxs, transaction);
                    }
                }
            }
        }
        return unspentTxs;
    }
    */

    /**
     * 查找钱包地址对应的所有 UTXO
     *
     * @param pubKeyHash 钱包公钥 Hash
     * @return
     */
    /*
    public TXOutput[] findUTXO(byte[] pubKeyHash) throws Exception {
        Transaction[] unspentTxs = this.findUnspentTransactions(pubKeyHash);
        TXOutput[] utxos = {};
        if (unspentTxs == null || unspentTxs.length == 0) {
            return utxos;
        }
        for (Transaction tx : unspentTxs) {
            for (TXOutput txOutput : tx.getOutputs()) {
//                if (txOutput.canBeUnlockedWith(address)) {
                if (txOutput.isLockedWithKey(pubKeyHash)) {
                    utxos = ArrayUtils.add(utxos, txOutput);
                }
            }
        }
        return utxos;
    }

*/
    /**
     * 寻找能够花费的交易
     *
     * @param pubKeyHash 钱包公钥 Hash
     * @param amount  花费金额
     */

    /*
    public SpendableOutputResult findSpendableOutputs(byte[] pubKeyHash, int amount) throws Exception {
        Transaction[] unspentTXs = this.findUnspentTransactions(pubKeyHash);
        int accumulated = 0;
        Map<String, int[]> unspentOuts = new HashMap<>();
        for (Transaction tx : unspentTXs) {

            String txId = Hex.encodeHexString(tx.getTxId());

            for (int outId = 0; outId < tx.getOutputs().length; outId++) {

                TXOutput txOutput = tx.getOutputs()[outId];

//                if (txOutput.canBeUnlockedWith(address) && accumulated < amount) {
                    if (txOutput.isLockedWithKey(pubKeyHash) && accumulated < amount) {
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
*/

    //----------------------Sign-------------------------

    /**
     * 依据交易 ID 查询交易信息
     *
     * @param txId 交易 ID
     * @return
     */
    private Transaction findTransaction(byte[] txId) throws Exception {

        for (BlockchainIterator iterator = this.getBlockchainIterator(); iterator.hashNext(); ) {
            Block block = iterator.next();
            for (Transaction tx : block.getTransactions()) {
                if (Arrays.equals(tx.getTxId(), txId)) {
                    return tx;
                }
            }
        }
        throw new Exception("ERROR: Can not found tx by txId ! ");
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

    /**
     * 进行交易签名
     *
     * @param tx         交易数据
     * @param privateKey 私钥
     */
    public void signTransaction(Transaction tx, BCECPrivateKey privateKey) throws Exception {
        // 先来找到这笔新的交易中，交易输入所引用的前面的多笔交易的数据
        Map<String, Transaction> prevTxMap = new HashMap<>();
        for (TXInput txInput : tx.getInputs()) {
            Transaction prevTx = this.findTransaction(txInput.getTxId());
            prevTxMap.put(Hex.encodeHexString(txInput.getTxId()), prevTx);
        }
        tx.sign(privateKey, prevTxMap);
    }

    /**
     * 交易签名验证
     *
     * @param tx
     */
    private boolean verifyTransactions(Transaction tx) throws Exception {
        if (tx.isCoinbase()) {
            return true;
        }

        Map<String, Transaction> prevTx = new HashMap<>();
        for (TXInput txInput : tx.getInputs()) {
            Transaction transaction = this.findTransaction(txInput.getTxId());
            prevTx.put(Hex.encodeHexString(txInput.getTxId()), transaction);
        }
        try {
            return tx.verify(prevTx);
        } catch (Exception e) {
            throw new Exception("Fail to verify transaction ! transaction invalid ! ");
        }
    }

    //------------------------UTXOSet--------------------------

    /**
     * 从交易输入中查询区块链中所有已被花费了的交易输出
     *
     * @return 交易 ID 以及对应的交易输出下标地址
     */

    private Map<String, int[]> getAllSpentTXOs() {
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
//                    if (txInput.canUnlockOutputWith(address)) {
//                    if (txInput.usesKey(pubKeyHash)) {
                    String inTxId = Hex.encodeHexString(txInput.getTxId());
                    int[] spentOutIndexArray = spentTXOs.get(inTxId);
                    if (spentOutIndexArray == null) {
                        spentTXOs.put(inTxId, new int[]{txInput.getTxOutputIndex()});
                    } else {
                        spentOutIndexArray = ArrayUtils.add(spentOutIndexArray, txInput.getTxOutputIndex());
                    }
                    spentTXOs.put(inTxId, spentOutIndexArray);
                }
//                }
            }
        }
        return spentTXOs;
    }

    /**
     * 查找所有的 unspent transaction outputs
     *
     * @return
     */
    public Map<String, TXOutput[]> findAllUTXOs() {
        Map<String, int[]> allSpentTXOs = this.getAllSpentTXOs();
        Map<String, TXOutput[]> allUTXOs = new HashMap<>();
        // 再次遍历所有区块中的交易输出
        for (BlockchainIterator blockchainIterator = this.getBlockchainIterator(); blockchainIterator.hashNext(); ) {
            Block block = blockchainIterator.next();
            for (Transaction transaction : block.getTransactions()) {

                String txId = Hex.encodeHexString(transaction.getTxId());

                int[] spentOutIndexArray = allSpentTXOs.get(txId);
                TXOutput[] txOutputs = transaction.getOutputs();
                for (int outIndex = 0; outIndex < txOutputs.length; outIndex++) {
                    if (spentOutIndexArray != null && ArrayUtils.contains(spentOutIndexArray, outIndex)) {
                        continue;
                    }
                    TXOutput[] UTXOArray = allUTXOs.get(txId);
                    if (UTXOArray == null) {
                        UTXOArray = new TXOutput[]{txOutputs[outIndex]};
                    } else {
                        UTXOArray = ArrayUtils.add(UTXOArray, txOutputs[outIndex]);
                    }
                    allUTXOs.put(txId, UTXOArray);
                }
            }
        }
        return allUTXOs;
    }

} 
```

#### 3.2.4 修改`Transaction.java`文件

打开`cldy.hanru.blockchain.block`包，修改`Transaction.java`文件。

修改步骤：

```java
修改步骤：
step1：添加时间戳字段
step2：修改 newUTXOTransaction 方法，从 utxoSet 中查找转账要使用的 utxo。
step3：修改：newCoinbaseTX() 
```

修改完后代码如下：

```java
 package cldy.hanru.blockchain.transaction;

import cldy.hanru.blockchain.block.Blockchain;
import cldy.hanru.blockchain.util.AddressUtils;
import cldy.hanru.blockchain.util.SerializeUtils;
import cldy.hanru.blockchain.wallet.Wallet;
import cldy.hanru.blockchain.wallet.WalletUtils;
import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;
import org.apache.commons.codec.binary.Hex;
import org.apache.commons.codec.digest.DigestUtils;
import org.apache.commons.lang3.ArrayUtils;
import org.apache.commons.lang3.StringUtils;
import org.bouncycastle.jcajce.provider.asymmetric.ec.BCECPrivateKey;
import org.bouncycastle.jce.ECNamedCurveTable;
import org.bouncycastle.jce.provider.BouncyCastleProvider;
import org.bouncycastle.jce.spec.ECParameterSpec;
import org.bouncycastle.jce.spec.ECPublicKeySpec;
import org.bouncycastle.math.ec.ECPoint;

import java.math.BigInteger;
import java.security.KeyFactory;
import java.security.PublicKey;
import java.security.Security;
import java.security.Signature;
import java.util.Arrays;
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
     * 创建日期
     */
    private long createTime;

    /**
     * 设置交易 ID
     */
    private void setTxId() {
        this.setTxId(DigestUtils.sha256(SerializeUtils.serialize(this)));
    }

    /**
     * 要签名的数据
     * @return
     */
    public byte[] getData() {
        // 使用序列化的方式对 Transaction 对象进行深度复制
        byte[] serializeBytes = SerializeUtils.serialize(this);
        Transaction copyTx = (Transaction) SerializeUtils.deserialize(serializeBytes);
        copyTx.setTxId(new byte[]{});
        return DigestUtils.sha256(SerializeUtils.serialize(copyTx));
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
//        TXInput txInput = new TXInput(new byte[]{}, -1, data);
        TXInput txInput = new TXInput(new byte[]{}, -1, null, data.getBytes());
        // 创建交易输出
//        TXOutput txOutput = new TXOutput(SUBSIDY, to);
        TXOutput txOutput = TXOutput.newTXOutput(SUBSIDY, to);
        // 创建交易
        Transaction tx = new Transaction(null, new TXInput[]{txInput}, new TXOutput[]{txOutput}, System.currentTimeMillis());
        // 设置交易 ID
        tx.setTxId();

//        tx.setTxId(tx.hash());
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

        // 获取钱包
        Wallet senderWallet = WalletUtils.getInstance().getWallet(from);
        byte[] pubKey = senderWallet.getPublicKey();
        byte[] pubKeyHash = AddressUtils.ripeMD160Hash(pubKey);

//        SpendableOutputResult result = blockchain.findSpendableOutputs(from, amount);
//        SpendableOutputResult result = blockchain.findSpendableOutputs(pubKeyHash, amount);
        SpendableOutputResult result = new UTXOSet(blockchain).findSpendableOutputs(pubKeyHash, amount);
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
//                txInputs = ArrayUtils.add(txInputs, new TXInput(txId, outIndex, from));
                txInputs = ArrayUtils.add(txInputs, new TXInput(txId, outIndex, null, pubKey));
            }
        }

        TXOutput[] txOutput = {};
//        txOutput = ArrayUtils.add(txOutput, new TXOutput(amount, to));
        txOutput = ArrayUtils.add(txOutput, TXOutput.newTXOutput(amount, to));
        if (accumulated > amount) {
//            txOutput = ArrayUtils.add(txOutput, new TXOutput((accumulated - amount), from));
            txOutput = ArrayUtils.add(txOutput, TXOutput.newTXOutput((accumulated - amount), from));

        }

        Transaction newTx = new Transaction(null, txInputs, txOutput, System.currentTimeMillis());
        newTx.setTxId();

//        newTx.setTxId(newTx.hash());

        // 进行交易签名
        blockchain.signTransaction(newTx, senderWallet.getPrivateKey());

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

    /**
     * 创建用于签名的交易数据副本，交易输入的 signature 和 pubKey 需要设置为 null
     *
     * @return
     */
    public Transaction trimmedCopy() {
        TXInput[] tmpTXInputs = new TXInput[this.getInputs().length];
        for (int i = 0; i < this.getInputs().length; i++) {
            TXInput txInput = this.getInputs()[i];
            tmpTXInputs[i] = new TXInput(txInput.getTxId(), txInput.getTxOutputIndex(), null, null);
        }

        TXOutput[] tmpTXOutputs = new TXOutput[this.getOutputs().length];
        for (int i = 0; i < this.getOutputs().length; i++) {
            TXOutput txOutput = this.getOutputs()[i];
            tmpTXOutputs[i] = new TXOutput(txOutput.getValue(), txOutput.getPubKeyHash());
        }

        return new Transaction(this.getTxId(), tmpTXInputs, tmpTXOutputs, this.getCreateTime());
    }

    /**
     * 签名
     *
     * @param privateKey 私钥
     * @param prevTxMap  前面多笔交易集合
     */
    public void sign(BCECPrivateKey privateKey, Map<String, Transaction> prevTxMap) throws Exception {
        // coinbase 交易信息不需要签名，因为它不存在交易输入信息
        if (this.isCoinbase()) {
            return;
        }
        // 再次验证一下交易信息中的交易输入是否正确，也就是能否查找对应的交易数据
        for (TXInput txInput : this.getInputs()) {
            if (prevTxMap.get(Hex.encodeHexString(txInput.getTxId())) == null) {
                throw new Exception("ERROR: Previous transaction is not correct");
            }
        }

        // 创建用于签名的交易信息的副本
        Transaction txCopy = this.trimmedCopy();

        Security.addProvider(new BouncyCastleProvider());
        Signature ecdsaSign = Signature.getInstance("SHA256withECDSA", BouncyCastleProvider.PROVIDER_NAME);
        ecdsaSign.initSign(privateKey);

        for (int i = 0; i < txCopy.getInputs().length; i++) {
            TXInput txInputCopy = txCopy.getInputs()[i];
            // 获取交易输入 TxID 对应的交易数据
            Transaction prevTx = prevTxMap.get(Hex.encodeHexString(txInputCopy.getTxId()));
            // 获取交易输入所对应的上一笔交易中的交易输出
            TXOutput prevTxOutput = prevTx.getOutputs()[txInputCopy.getTxOutputIndex()];
            txInputCopy.setPubKey(prevTxOutput.getPubKeyHash());
            txInputCopy.setSignature(null);
            // 得到要签名的数据
            byte[] signData = txCopy.getData();
            txInputCopy.setPubKey(null);

            // 对整个交易信息仅进行签名
            ecdsaSign.update(signData);

            byte[] signature = ecdsaSign.sign();

            // 将整个交易数据的签名赋值给交易输入，因为交易输入需要包含整个交易信息的签名
            // 注意是将得到的签名赋值给原交易信息中的交易输入
            this.getInputs()[i].setSignature(signature);
        }
    }

    /**
     * 验证交易信息
     *
     * @param prevTxMap 前面多笔交易集合
     * @return
     */
    public boolean verify(Map<String, Transaction> prevTxMap) throws Exception {
        // coinbase 交易信息不需要签名，也就无需验证
        if (this.isCoinbase()) {
            return true;
        }

        // 再次验证一下交易信息中的交易输入是否正确，也就是能否查找对应的交易数据
        for (TXInput txInput : this.getInputs()) {
            if (prevTxMap.get(Hex.encodeHexString(txInput.getTxId())) == null) {
                throw new Exception("ERROR: Previous transaction is not correct");
            }
        }

        // 创建用于签名验证的交易信息的副本
        Transaction txCopy = this.trimmedCopy();

        Security.addProvider(new BouncyCastleProvider());
        ECParameterSpec ecParameters = ECNamedCurveTable.getParameterSpec("secp256k1");
        KeyFactory keyFactory = KeyFactory.getInstance("ECDSA", BouncyCastleProvider.PROVIDER_NAME);
        Signature ecdsaVerify = Signature.getInstance("SHA256withECDSA", BouncyCastleProvider.PROVIDER_NAME);

        for (int i = 0; i < this.getInputs().length; i++) {
            TXInput txInput = this.getInputs()[i];
            // 获取交易输入 TxID 对应的交易数据
            Transaction prevTx = prevTxMap.get(Hex.encodeHexString(txInput.getTxId()));
            // 获取交易输入所对应的上一笔交易中的交易输出
            TXOutput prevTxOutput = prevTx.getOutputs()[txInput.getTxOutputIndex()];

            TXInput txInputCopy = txCopy.getInputs()[i];
            txInputCopy.setSignature(null);
            txInputCopy.setPubKey(prevTxOutput.getPubKeyHash());
            // 得到要签名的数据
            byte[] signData = txCopy.getData();
            txInputCopy.setPubKey(null);

            // 使用椭圆曲线 x,y 点去生成公钥 Key
            BigInteger x = new BigInteger(1, Arrays.copyOfRange(txInput.getPubKey(), 1, 33));
            BigInteger y = new BigInteger(1, Arrays.copyOfRange(txInput.getPubKey(), 33, 65));
            ECPoint ecPoint = ecParameters.getCurve().createPoint(x, y);

            ECPublicKeySpec keySpec = new ECPublicKeySpec(ecPoint, ecParameters);
            PublicKey publicKey = keyFactory.generatePublic(keySpec);
            ecdsaVerify.initVerify(publicKey);
            ecdsaVerify.update(signData);
            if (!ecdsaVerify.verify(txInput.getSignature())) {
                return false;
            }
        }
        return true;
    }

} 
```

#### 3.2.5 修改`CLI.java`文件

打开`cldy.hanru.blockchain.cli`包，修改`CLI.java`文件。

修改步骤：

```java
修改步骤：
step1：修改 getBalance()方法
step2：修改 send()方法 
```

修改完后代码如下：

```java
package cldy.hanru.blockchain.cli;

import cldy.hanru.blockchain.block.Block;
import cldy.hanru.blockchain.block.Blockchain;
import cldy.hanru.blockchain.pow.ProofOfWork;
import cldy.hanru.blockchain.store.RocksDBUtils;
import cldy.hanru.blockchain.transaction.TXInput;
import cldy.hanru.blockchain.transaction.TXOutput;
import cldy.hanru.blockchain.transaction.Transaction;
import cldy.hanru.blockchain.transaction.UTXOSet;
import cldy.hanru.blockchain.util.Base58Check;
import cldy.hanru.blockchain.wallet.Wallet;
import cldy.hanru.blockchain.wallet.WalletUtils;
import lombok.extern.slf4j.Slf4j;
import org.apache.commons.cli.*;
import org.apache.commons.codec.binary.Hex;
import org.apache.commons.lang3.StringUtils;
import org.apache.commons.lang3.math.NumberUtils;

import java.text.SimpleDateFormat;
import java.util.*;

/**
 * @author hanru
 */
@Slf4j
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
        System.out.println("  createwallet - Generates a new key-pair and saves it into the wallet file");
        System.out.println("  printaddresses - print all wallet address");
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
        CommandLineParser parser = new DefaultParser();
        CommandLine cmd = null;
        try {
            cmd = parser.parse(options, args);
        } catch (ParseException e) {
            e.printStackTrace();
        }

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
                try {
                    this.getBalance(getBalanceAddress);
                } catch (Exception e) {
                    e.printStackTrace();
                }finally {
                    RocksDBUtils.getInstance().closeDB();
                }
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
                try {
                    this.send(sendFrom, sendTo, Integer.valueOf(sendAmount));
                } catch (Exception e) {
                    e.printStackTrace();
                }finally {
                    RocksDBUtils.getInstance().closeDB();
                }
                break;
            case "printchain":
                this.printChain();
                break;
            case "h":
                this.help();
                break;

            case "createwallet":
                try {
                    this.createWallet();
                } catch (Exception e) {
                    e.printStackTrace();
                }
                break;
            case "printaddresses":
                try {
                    this.printAddresses();
                } catch (Exception e) {
                    e.printStackTrace();
                }
                break;
            default:
                this.help();
        }

    }

    /**
     * 创建区块链
     *
     * @param address
     */
    private void createBlockchain(String address) {

        Blockchain blockchain = Blockchain.createBlockchain(address);
        UTXOSet utxoSet = new UTXOSet(blockchain);
        utxoSet.reIndex();
        log.info("Done ! ");
    }

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
                    System.out.printf("\t\t 交易 ID：%s\n", Hex.encodeHexString(tx.getTxId()));
                    System.out.println("\t\t 输入：");
                    for (TXInput in : tx.getInputs()) {
                        System.out.printf("\t\t\tTxID:%s\n", Hex.encodeHexString(in.getTxId()));
                        System.out.printf("\t\t\tOutputIndex:%d\n", in.getTxOutputIndex());
//                        System.out.printf("\t\t\tScriptSiq:%s\n" , in.getScriptSig());
                        System.out.printf("\t\t\tPubKey:%s\n", Hex.encodeHexString(in.getPubKey()));
                    }
                    System.out.println("\t\t 输出：");
                    for (TXOutput out : tx.getOutputs()) {
                        System.out.printf("\t\t\tvalue:%d\n", out.getValue());
//                        System.out.printf("\t\t\tScriptPubKey:%s\n" , out.getScriptPubKey());
                        System.out.printf("\t\t\tPubKeyHash:%s\n", Hex.encodeHexString(out.getPubKeyHash()));
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

        // 检查钱包地址是否合法
        try {
            Base58Check.base58ToBytes(address);
        } catch (Exception e) {
            throw new Exception("ERROR: invalid wallet address");
        }
        Blockchain blockchain = Blockchain.createBlockchain(address);
        // 得到公钥 Hash 值
        byte[] versionedPayload = Base58Check.base58ToBytes(address);
        byte[] pubKeyHash = Arrays.copyOfRange(versionedPayload, 1, versionedPayload.length);

//        TXOutput[] txOutputs = blockchain.findUTXO(address);
//        TXOutput[] txOutputs = blockchain.findUTXO(pubKeyHash);
        UTXOSet utxoSet = new UTXOSet(blockchain);
        TXOutput[] txOutputs = utxoSet.findUTXOs(pubKeyHash);
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
        // 检查钱包地址是否合法
        try {
            Base58Check.base58ToBytes(from);
        } catch (Exception e) {
            throw new Exception("ERROR: sender address invalid ! address=" + from);
        }
        // 检查钱包地址是否合法
        try {
            Base58Check.base58ToBytes(to);
        } catch (Exception e) {
            throw new Exception("ERROR: receiver address invalid ! address=" + to);
        }
        if (amount < 1) {
            throw new Exception("ERROR: amount invalid ! ");
        }

        /*
        Blockchain blockchain = Blockchain.createBlockchain(from);
        Transaction transaction = Transaction.newUTXOTransaction(from, to, amount, blockchain);

        blockchain.mineBlock(new Transaction[]{transaction});
        RocksDBUtils.getInstance().closeDB();
        System.out.println("Success!");
        */

        Blockchain blockchain = Blockchain.createBlockchain(from);
        // 新交易
        Transaction transaction = Transaction.newUTXOTransaction(from, to, amount, blockchain);
        // 奖励
        Transaction rewardTx = Transaction.newCoinbaseTX(from, "");
        List<Transaction> transactions = new ArrayList<>();
        transactions.add(transaction);
        transactions.add(rewardTx);
        Block newBlock = blockchain.mineBlock(transactions);
        new UTXOSet(blockchain).update(newBlock);
        log.info("Success!");
    }

    /**
     * 创建钱包
     *
     * @throws Exception
     */
    private void createWallet() throws Exception {
        Wallet wallet = WalletUtils.getInstance().createWallet();
        System.out.println("wallet address : " + wallet.getAddress());
    }

    /**
     * 打印钱包地址
     *
     * @throws Exception
     */
    private void printAddresses() throws Exception {
        Set<String> addresses = WalletUtils.getInstance().getAddresses();
        if (addresses == null || addresses.isEmpty()) {
            System.out.println("There isn't address");
            return;
        }
        for (String address : addresses) {
            System.out.println("Wallet address: " + address);
        }
    }
} 
```

#### 3.2.6 修改`blockchain.sh`脚本文件

最后修改`blockchain.sh`脚本文件，修改后内容如下：

```java
#!/bin/bash

set -e

# Check if the jar has been built.
if [ ! -e target/part8_Transaction2-jar-with-dependencies.jar ]; then
  echo "Compiling blockchain project to a JAR"
  mvn package -DskipTests
fi

java -jar target/part8_Transaction2-jar-with-dependencies.jar "$@" 
```

##

## 4\. 优化内容讲解

### 4.1 奖励 Reward

在上一篇文章中，我们略过的一个小细节是挖矿奖励。现在，我们已经可以来完善这个细节了。

挖矿奖励，实际上就是一笔`coinbase` 交易。当一个挖矿节点开始挖出一个新块时，它会将交易从队列中取出，并在前面附加一笔`coinbase`交易。`coinbase` 交易只有一个输出，里面包含了矿工的公钥哈希。

实现奖励，非常简单，更新`CLI`类即可，修改 `send()` 方法：

```java
/**
     * 转账
     *
     * @param from
     * @param to
     * @param amount
     */
    private void send(String from, String to, int amount) throws Exception {
        // 检查钱包地址是否合法
        try {
            Base58Check.base58ToBytes(from);
        } catch (Exception e) {
            throw new Exception("ERROR: sender address invalid ! address=" + from);
        }
        // 检查钱包地址是否合法
        try {
            Base58Check.base58ToBytes(to);
        } catch (Exception e) {
            throw new Exception("ERROR: receiver address invalid ! address=" + to);
        }
        if (amount < 1) {
            throw new Exception("ERROR: amount invalid ! ");
        }

        Blockchain blockchain = Blockchain.createBlockchain(from);
        // 新交易
        Transaction transaction = Transaction.newUTXOTransaction(from, to, amount, blockchain);
        // 奖励
        Transaction rewardTx = Transaction.newCoinbaseTX(from, "");
        List<Transaction> transactions = new ArrayList<>();
        transactions.add(transaction);
        transactions.add(rewardTx);
        Block newBlock = blockchain.mineBlock(transactions);
        new UTXOSet(blockchain).update(newBlock);
        log.info("Success!");
    } 
```

现在我们的逻辑调整为 ，每次转账都会给与奖励，而奖励通过 coinbase 交易实现，为了让这些 coinbase 交易的交易 ID 不同，我们需要在`Transaction`类添加时间字段：

```java
public class Transaction {
    private static final int SUBSIDY = 10;

...

    /**
     * 创建日期
     */
    private long createTime;
} 
```

接下来就需要修改创建 coinbase 交易和创建转账交易的方法，添加创建日期：

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
//        TXInput txInput = new TXInput(new byte[]{}, -1, data);
        TXInput txInput = new TXInput(new byte[]{}, -1, null, data.getBytes());
        // 创建交易输出
//        TXOutput txOutput = new TXOutput(SUBSIDY, to);
        TXOutput txOutput = TXOutput.newTXOutput(SUBSIDY, to);
        // 创建交易
        Transaction tx = new Transaction(null, new TXInput[]{txInput}, new TXOutput[]{txOutput}, System.currentTimeMillis());
        // 设置交易 ID
        tx.setTxId();

        return tx;
    } 
```

以及：

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

        // 获取钱包
        Wallet senderWallet = WalletUtils.getInstance().getWallet(from);
        byte[] pubKey = senderWallet.getPublicKey();
        byte[] pubKeyHash = AddressUtils.ripeMD160Hash(pubKey);

//        SpendableOutputResult result = blockchain.findSpendableOutputs(from, amount);
        SpendableOutputResult result = blockchain.findSpendableOutputs(pubKeyHash, amount);
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
//                txInputs = ArrayUtils.add(txInputs, new TXInput(txId, outIndex, from));
                txInputs = ArrayUtils.add(txInputs, new TXInput(txId, outIndex, null, pubKey));
            }
        }

        TXOutput[] txOutput = {};
//        txOutput = ArrayUtils.add(txOutput, new TXOutput(amount, to));
        txOutput = ArrayUtils.add(txOutput, TXOutput.newTXOutput(amount, to));
        if (accumulated > amount) {
//            txOutput = ArrayUtils.add(txOutput, new TXOutput((accumulated - amount), from));
            txOutput = ArrayUtils.add(txOutput, TXOutput.newTXOutput((accumulated - amount), from));

        }

        Transaction newTx = new Transaction(null, txInputs, txOutput, System.currentTimeMillis());
        newTx.setTxId();

        // 进行交易签名
        blockchain.signTransaction(newTx, senderWallet.getPrivateKey());

        return newTx;
    } 
```

在我们的实现中，创建交易的第一个人同时挖出了新块，所以会得到一笔奖励。

在项目添加了奖励之后，我们进行测试，首先在终端创建 3 个地址：

```java
hanru:part8_Transaction2 ruby$ ./blockchain.sh h
hanru:part8_Transaction2 ruby$ ./blockchain.sh createwallet
hanru:part8_Transaction2 ruby$ ./blockchain.sh createwallet
hanru:part8_Transaction2 ruby$ ./blockchain.sh printaddresses 
```

运行结果如下：

![`img.kongyixueyuan.com/010_002_%E5%A5%96%E5%8A%B11.png`](img/6d0d44f7b60e84e5253ca2f7eae5f944.jpg)

接下来我们创建创世区块，并进行一次转账：

```java
hanru:part8_Transaction2 ruby$ ./blockchain.sh createblockchain -address 1FSwdBA55ne6a3QWLnzH3PtPtbDbNHBkXt
hanru:part8_Transaction2 ruby$ ./blockchain.sh getbalance -address 1FSwdBA55ne6a3QWLnzH3PtPtbDbNHBkXt
hanru:part8_Transaction2 ruby$ ./blockchain.sh send -from 1FSwdBA55ne6a3QWLnzH3PtPtbDbNHBkXt -to 19a5Lfex3v9EgFUuNG7n5qR2DaXe5RzUT2 -amount 4
hanru:part8_Transaction2 ruby$ ./blockchain.sh getbalance  -address 1FSwdBA55ne6a3QWLnzH3PtPtbDbNHBkXt
hanru:part8_Transaction2 ruby$ ./blockchain.sh getbalance  -address 19a5Lfex3v9EgFUuNG7n5qR2DaXe5RzUT2 
```

运行结果如下：

![`img.kongyixueyuan.com/010_003_%E5%A5%96%E5%8A%B12.png`](img/0ac3ce8fa1c8d3043522102d6f4f6117.jpg)

> 说明：
> 
> 1.  比如创世区块中 1FSwdBA55ne6a3QWLnzH3PtPtbDbNHBkXt 地址有 10 个 Token，
> 2.  转账给 19a5Lfex3v9EgFUuNG7n5qR2DaXe5RzUT2 地址 4 个 Token，
> 3.  那么 1FSwdBA55ne6a3QWLnzH3PtPtbDbNHBkXt 的余额是 16(转账找零 6 个，加上得到奖励 10 个），
> 4.  19a5Lfex3v9EgFUuNG7n5qR2DaXe5RzUT2 的余额是 4。

### 4.2 UTXO 集

在持久化的章节中，我们研究了 Bitcoin Core 是如何在一个数据库中存储块的，并且了解到区块被存储在 `blocks` 数据库，交易输出被存储在 `chainstate` 数据库。会回顾一下 `chainstate` 的结构：

1.  `c` + 32 字节的交易哈希 -> 该笔交易的未花费交易输出记录
2.  `B` + 32 字节的块哈希 -> 未花费交易输出的块哈希

在之前那篇文章中，虽然我们已经实现了交易，但是并没有使用 `chainstate` 来存储交易的输出。所以，接下来我们继续完成这部分。

`chainstate` 不存储交易。它所存储的是 UTXO 集，也就是未花费交易输出的集合。除此以外，它还存储了“数据库表示的未花费交易输出的块哈希”，不过我们会暂时略过块哈希这一点，因为我们还没有用到块高度（但是我们会在接下来的文章中继续改进）。

那么，我们为什么需要 UTXO 集呢？

来思考一下我们早先实现的在`Blockchain`类中的`findUnspentTransactions()` 方法：

```java
 /**
     * 查找钱包地址对应的所有未花费的交易
     *
     * @param pubKeyHash 钱包公钥 Hash
     * @return
     */

private Transaction[] findUnspentTransactions(byte[] pubKeyHash) throws Exception {
    Map<String, int[]> allSpentTXOs = this.getAllSpentTXOs(pubKeyHash);
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
                //if (transaction.getOutputs()[outIndex].canBeUnlockedWith(address)) {
                if (transaction.getOutputs()[outIndex].isLockedWithKey(pubKeyHash)) {
                    unspentTxs = ArrayUtils.add(unspentTxs, transaction);
                }
            }
        }
    }
    return unspentTxs;
} 
```

这个函数找到有未花费输出的交易。由于交易被保存在区块中，所以它会对区块链里面的每一个区块进行迭代，检查里面的每一笔交易。截止 2017 年 9 月 18 日，在比特币中已经有 485，860 个块，整个数据库所需磁盘空间超过 140 Gb。这意味着一个人如果想要验证交易，必须要运行一个全节点。此外，验证交易将会需要在许多块上进行迭代。

整个问题的解决方案是有一个仅有未花费输出的索引，这就是 UTXO 集要做的事情：这是一个从所有区块链交易中构建（对区块进行迭代，但是只须做一次）而来的缓存，然后用它来计算余额和验证新的交易。截止 2017 年 9 月，UTXO 集大概才有 2.7 Gb。

好了，让我们来想一下，为了实现 UTXOs 池我们需要做哪些事情。当前，有下列方法被用于查找交易信息：

1.  **Blockchain.getAllSpentTXOs** —— 查询所有已被花费的交易输出。它需要遍历区块链中所有区块中交易信息。

2.  **Blockchain.findUnspentTransactions** —— 查询包含未被花费的交易输出的交易信息。它也需要遍历区块链中所有区块中交易信息。

3.  **Blockchain.findSpendableOutputs** —— 该方法用于新的交易创建之时。它需要找到足够多的交易输出，以满足所需支付的金额。需要调用 **Blockchain.findUnspentTransactions** 方法。

4.  **Blockchain.findUTXO** —— 查询钱包地址所对应的所有未花费交易输出，然后用于计算钱包余额。需要调用

    **Blockchain.findUnspentTransactions** 方法。

5.  **Blockchain.findTransaction** —— 通过交易 ID 查询交易信息。它需要遍历所有的区块直到找到交易信息为止。

如你所见，上面这些方法都需要去遍历数据库中的所有区块。由于 UTXOs 池只存储未被花费的交易输出，而不会存储所有的交易信息，因此我们不会对有 **Blockchain.findTransaction** 进行优化。

那么，我们需要下列这些方法：

1.  **Blockchain.findUTXO** —— 通过遍历所有的区块来找到所有未被花费的交易输出.
2.  **UTXOSet.reindex** —— 调用上面 **findUTXO** 方法，然后将查询结果存储在数据库中。也即需要进行缓存的地方。
3.  **UTXOSet.findSpendableOutputs** —— 与 **Blockchain.findSpendableOutputs** 类似，区别在于会使用 UTXO 池。
4.  **UTXOSet.findUTXO** —— 与**Blockchain.findUTXO** 类似，区别在于会使用 UTXO 池。
5.  **Blockchain.findTransaction** —— 逻辑保持不变。

这样，两个使用最频繁的方法将从现在开始使用缓存！

接下来，我们个通过代码实现一下：

首先，我们在`cldy.hanru.blockchain.transaction`包下再创建一个 java 文件，命名为：`UTXOSet.java`。添加一个类：

```java
/**
 * 未被花费的交易输出池
 * @author hanru
 */
@NoArgsConstructor
@AllArgsConstructor
@Slf4j
public class UTXOSet {

    private Blockchain blockchain;

} 
```

我们将会使用一个单一数据库，但是我们会将 UTXO 集从存储在不同的 bucket 中。因此，`UTXOSet` 跟 `Blockchain` 一起。

因为我们需要有个 bucket 来存储所有的 UTXO，所以我们修改`RocksDBUtils.java`文件，添加对应的存储操作方法。

首先先定义存储 UTXO 的 bucket

```java
/**
     * 链状态桶 Key
     */
private static final String CHAINSTATE_BUCKET_KEY = "chainstate";
/**
     * chainstate buckets
     */
@Getter
private Map<String, byte[]> chainstateBucket; 
```

接下来定义一些工具方法，`initChainStateBucket()`方法用于初始化 bucket，并在该类的构造函数中调用：

```java
/**
     * 初始化 blocks 数据桶
     */
    private void initChainStateBucket() {
        try {
            byte[] chainstateBucketKey = SerializeUtils.serialize(CHAINSTATE_BUCKET_KEY);
            byte[] chainstateBucketBytes = db.get(chainstateBucketKey);
            if (chainstateBucketBytes != null) {
                chainstateBucket = (Map) SerializeUtils.deserialize(chainstateBucketBytes);
            } else {
                chainstateBucket = new HashMap();
                db.put(chainstateBucketKey, SerializeUtils.serialize(chainstateBucket));
            }
        } catch (RocksDBException e) {
            log.error("Fail to init chainstate bucket ! ", e);
            throw new RuntimeException("Fail to init chainstate bucket ! ", e);
        }
    } 
```

`cleanChainStateBucket()`方法用于清空 bucket：

```java
/**
 * 清空 chainstate bucket
 */
public void cleanChainStateBucket() {
    try {
        chainstateBucket.clear();
    } catch (Exception e) {
        log.error("Fail to clear chainstate bucket ! ", e);
        throw new RuntimeException("Fail to clear chainstate bucket ! ", e);
    }
} 
```

最后定义 3 个方法，用于表示添加、获取、删除 UTXO：

```java
 /**
     * 保存 UTXO 数据
     *
     * @param key   交易 ID
     * @param utxos UTXOs
     */
    public void putUTXOs(String key, TXOutput[] utxos) {
        try {
            chainstateBucket.put(key, SerializeUtils.serialize(utxos));
            db.put(SerializeUtils.serialize(CHAINSTATE_BUCKET_KEY), SerializeUtils.serialize(chainstateBucket));
        } catch (Exception e) {
            log.error("Fail to put UTXOs into chainstate bucket ! key=" + key, e);
            throw new RuntimeException("Fail to put UTXOs into chainstate bucket ! key=" + key, e);
        }
    }

    /**
     * 查询 UTXO 数据
     *
     * @param key 交易 ID
     */
    public TXOutput[] getUTXOs(String key) {
        byte[] utxosByte = chainstateBucket.get(key);
        if (utxosByte != null) {
            return (TXOutput[]) SerializeUtils.deserialize(utxosByte);
        }
        return null;
    }

    /**
     * 删除 UTXO 数据
     *
     * @param key 交易 ID
     */
    public void deleteUTXOs(String key) {
        try {
            chainstateBucket.remove(key);
            db.put(SerializeUtils.serialize(CHAINSTATE_BUCKET_KEY), SerializeUtils.serialize(chainstateBucket));
        } catch (Exception e) {
            log.error("Fail to delete UTXOs by key ! key=" + key, e);
            throw new RuntimeException("Fail to delete UTXOs by key ! key=" + key, e);
        }
    } 
```

工具类准备好后，我们在 UTXOSet.java 中，继续定义一个`reIndex()`方法：

```java
/**
     * 重建 UTXO 池索引
     */
@Synchronized
public void reIndex() {
    log.info("Start to reIndex UTXO set !");
    RocksDBUtils.getInstance().cleanChainStateBucket();
    Map<String, TXOutput[]> allUTXOs = blockchain.findAllUTXOs();
    for (Map.Entry<String, TXOutput[]> entry : allUTXOs.entrySet()) {
        RocksDBUtils.getInstance().putUTXOs(entry.getKey(), entry.getValue());
    }
    log.info("ReIndex UTXO set finished ! ");
} 
```

这个方法初始化了 UTXO 集。首先，先清除之前的 bucket。然后从区块链中获取所有的未花费输出，最终将输出保存到 bucket 中。该方法也用于重置 UTXO 集。

在`Blockchain.java`中添加方法，用于查询所有的未花费输出：

```java
/**
     * 查找所有的 unspent transaction outputs
     *
     * @return
     */
    public Map<String, TXOutput[]> findAllUTXOs() {
        Map<String, int[]> allSpentTXOs = this.getAllSpentTXOs();
        Map<String, TXOutput[]> allUTXOs = new HashMap();
        // 再次遍历所有区块中的交易输出
        for (BlockchainIterator blockchainIterator = this.getBlockchainIterator(); blockchainIterator.hashNext(); ) {
            Block block = blockchainIterator.next();
            for (Transaction transaction : block.getTransactions()) {

                String txId = Hex.encodeHexString(transaction.getTxId());

                int[] spentOutIndexArray = allSpentTXOs.get(txId);
                TXOutput[] txOutputs = transaction.getOutputs();
                for (int outIndex = 0; outIndex < txOutputs.length; outIndex++) {
                    if (spentOutIndexArray != null && ArrayUtils.contains(spentOutIndexArray, outIndex)) {
                        continue;
                    }
                    TXOutput[] UTXOArray = allUTXOs.get(txId);
                    if (UTXOArray == null) {
                        UTXOArray = new TXOutput[]{txOutputs[outIndex]};
                    } else {
                        UTXOArray = ArrayUtils.add(UTXOArray, txOutputs[outIndex]);
                    }
                    allUTXOs.put(txId, UTXOArray);
                }
            }
        }
        return allUTXOs;
    } 
```

接下来，修改`CLI.java`文件，修改`createBlockchain()`方法代码如下:

```java
/**
     * 创建区块链
     *
     * @param address
     */
    private void createBlockchain(String address) {

        Blockchain blockchain = Blockchain.createBlockchain(address);
        UTXOSet utxoSet = new UTXOSet(blockchain);
        utxoSet.reIndex();
        log.info("Done ! ");
    } 
```

当创建创世区块后，就会立刻进行初始化 UTXO 集。目前，即使这里看起来有点“杀鸡用牛刀”，因为一条链开始的时候，只有一个块，里面只有一笔交易。

因为创建了创世区块，有`CoinBase`交易的 10 个 Token，这是一个未花费的`TxOutput`，找到之后可以存储到 UTXO 集中。

### 4.3 优化查询余额

有了 UTXO 集，我们想要查询余额，可以不用遍历整个区块链的所有区块，而是直接查找 UTXO 集，找出对应地址的 utxo，进行累加即可。

接下来，我们在`UTXOSet.java`中，添加`findUTXOs()`,用于查找指定账户的所有的 utxo，代码如下：

```java
/**
     * 查找钱包地址对应的所有 UTXO
     *
     * @param pubKeyHash 钱包公钥 Hash
     * @return
     */
    public TXOutput[] findUTXOs(byte[] pubKeyHash) {
        TXOutput[] utxos = {};
        Map<String, byte[]> chainstateBucket = RocksDBUtils.getInstance().getChainstateBucket();
        if (chainstateBucket.isEmpty()) {
            return utxos;
        }
        for (byte[] value : chainstateBucket.values()) {
            TXOutput[] txOutputs = (TXOutput[]) SerializeUtils.deserialize(value);
            for (TXOutput txOutput : txOutputs) {
                if (txOutput.isLockedWithKey(pubKeyHash)) {
                    utxos = ArrayUtils.add(utxos, txOutput);
                }
            }
        }
        return utxos;
    } 
```

接下来修改，修改`CLI.java`文件，不再通过 blockchain 对象调用原来的查询方法了，改用 utxoSet 对象进行查询余额，代码如下：

```java
/**
     * 查询钱包余额
     *
     * @param address 钱包地址
     */
    private void getBalance(String address) throws Exception {

        // 检查钱包地址是否合法
        try {
            Base58Check.base58ToBytes(address);
        } catch (Exception e) {
            throw new Exception("ERROR: invalid wallet address");
        }
        Blockchain blockchain = Blockchain.createBlockchain(address);
        // 得到公钥 Hash 值
        byte[] versionedPayload = Base58Check.base58ToBytes(address);
        byte[] pubKeyHash = Arrays.copyOfRange(versionedPayload, 1, versionedPayload.length);

//        TXOutput[] txOutputs = blockchain.findUTXO(address);
//        TXOutput[] txOutputs = blockchain.findUTXO(pubKeyHash);
        UTXOSet utxoSet = new UTXOSet(blockchain);
        TXOutput[] txOutputs = utxoSet.findUTXOs(pubKeyHash);
        int balance = 0;
        if (txOutputs != null && txOutputs.length > 0) {
            for (TXOutput txOutput : txOutputs) {
                balance += txOutput.getValue();
            }
        }
        System.out.printf("Balance of '%s': %d\n", address, balance);
    } 
```

接下来，我们测试一下，先进行一次转账(转账我们还没有使用 UTXO 集优化)，然后再查询余额。在终端中输入以下命令：

```java
# 首先重新编译程序
hanru:part8_Transaction2 ruby$ mvn package
# 创建新的钱包地址
hanru:part8_Transaction2 ruby$ ./blockchain.sh createwallet
# 转账，19a5Lfex3v9EgFUuNG7n5qR2DaXe5RzUT2 账户因为之前的转账操作，有 4 个 Token
hanru:part8_Transaction2 ruby$ ./blockchain.sh send -from 19a5Lfex3v9EgFUuNG7n5qR2DaXe5RzUT2 -to 17mW1XamRdFZBsnmfh3DxbC5pW1AttJznn -amount 3
# 查询余额
hanru:part8_Transaction2 ruby$ ./blockchain.sh getbalance  -address 19a5Lfex3v9EgFUuNG7n5qR2DaXe5RzUT2
hanru:part8_Transaction2 ruby$ ./blockchain.sh getbalance  -address 17mW1XamRdFZBsnmfh3DxbC5pW1AttJznn 
```

运行结果如下：

![`img.kongyixueyuan.com/010_004_%E6%9F%A5%E8%AF%A2%E4%BD%99%E9%A2%9D.png`](img/4016110782f1d56ffbca13e47fab971a.jpg)

### 4.4 优化转账交易

因为我们已经将区块链中的 utxo 都存储到 UTXO 集中，所以 UTXO 集可以用于转账发送币。

在 UTXOSet 中，继续修改，实现转账。添加`findSpendableOutputs()`方法，代码如下：

```java
 /**
     * 寻找能够花费的交易
     *
     * @param pubKeyHash 钱包公钥 Hash
     * @param amount     花费金额
     */
    public SpendableOutputResult findSpendableOutputs(byte[] pubKeyHash, int amount) {
        Map<String, int[]> unspentOuts = Maps.newHashMap();
        int accumulated = 0;
        Map<String, byte[]> chainstateBucket = RocksDBUtils.getInstance().getChainstateBucket();
        for (Map.Entry<String, byte[]> entry : chainstateBucket.entrySet()) {
            String txId = entry.getKey();
            TXOutput[] txOutputs = (TXOutput[]) SerializeUtils.deserialize(entry.getValue());

            for (int outId = 0; outId < txOutputs.length; outId++) {
                TXOutput txOutput = txOutputs[outId];
                if (txOutput.isLockedWithKey(pubKeyHash) && accumulated < amount) {
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

以上方法用于找出转账需要用的 SpendableOutputResult。

接下来，修改 Transactio 中的`newUTXOTransaction()`方法。不在通过 blockchain 查找转账所需要的 SpendableOutputResult，而是修改为 UTXOSe 来查找，修改代码如下：

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

        // 获取钱包
        Wallet senderWallet = WalletUtils.getInstance().getWallet(from);
        byte[] pubKey = senderWallet.getPublicKey();
        byte[] pubKeyHash = AddressUtils.ripeMD160Hash(pubKey);

//        SpendableOutputResult result = blockchain.findSpendableOutputs(from, amount);
//        SpendableOutputResult result = blockchain.findSpendableOutputs(pubKeyHash, amount);
        SpendableOutputResult result = new UTXOSet(blockchain).findSpendableOutputs(pubKeyHash, amount);

       ...

        // 进行交易签名
        blockchain.signTransaction(newTx, senderWallet.getPrivateKey());

        return newTx;
    } 
```

通过转账产生交易，一定会花费之前的 utxo，再产生新的 utxo。有了 UTXO 集，也就意味着我们的数据（交易）现在已经被分开存储：实际交易被存储在区块链中，未花费输出被存储在 UTXO 集中。这样一来，我们就需要一个良好的同步机制，因为我们想要 UTXO 集时刻处于最新状态，并且存储最新交易的输出。但是我们不想每生成一个新块，就重新生成索引，因为这正是我们要极力避免的频繁区块链扫描。因此，我们需要一个机制来更新 UTXO 集。接下来，我们在`UTXOSet.java`文件中，添加一个`Update()`方法，代码如下：

```java
 /**
     * 更新 UTXO 池
     * 当一个新的区块产生时，需要去做两件事情：
     * 1）从 UTXO 池中移除花费掉了的交易输出；
     * 2）保存新的未花费交易输出；
     *
     * @param tipBlock 最新的区块
     */
    @Synchronized
    public void update(Block tipBlock) {
        if (tipBlock == null) {
            log.error("Fail to update UTXO set ! tipBlock is null !");
            throw new RuntimeException("Fail to update UTXO set ! ");
        }
        for (Transaction transaction : tipBlock.getTransactions()) {

            // 根据交易输入排查出剩余未被使用的交易输出
            if (!transaction.isCoinbase()) {
                for (TXInput txInput : transaction.getInputs()) {
                    // 余下未被使用的交易输出
                    TXOutput[] remainderUTXOs = {};
                    String txId = Hex.encodeHexString(txInput.getTxId());
                    TXOutput[] txOutputs = RocksDBUtils.getInstance().getUTXOs(txId);

                    if (txOutputs == null) {
                        continue;
                    }

                    for (int outIndex = 0; outIndex < txOutputs.length; outIndex++) {
                        if (outIndex != txInput.getTxOutputIndex()) {
                            remainderUTXOs = ArrayUtils.add(remainderUTXOs, txOutputs[outIndex]);
                        }
                    }

                    // 没有剩余则删除，否则更新
                    if (remainderUTXOs.length == 0) {
                        RocksDBUtils.getInstance().deleteUTXOs(txId);
                    } else {
                        RocksDBUtils.getInstance().putUTXOs(txId, remainderUTXOs);
                    }
                }
            }

            // 新的交易输出保存到 DB 中
            TXOutput[] txOutputs = transaction.getOutputs();
            String txId = Hex.encodeHexString(transaction.getTxId());
            RocksDBUtils.getInstance().putUTXOs(txId, txOutputs);
        }

    } 
```

虽然这个方法看起来有点复杂，但是它所要做的事情非常直观。当挖出一个新块时，应该更新 UTXO 集。更新意味着移除已花费输出，并从新挖出来的交易中加入未花费输出。如果一笔交易的输出被移除，并且不再包含任何输出，那么这笔交易也应该被移除。相当简单！

最后，在`CLI.java`文件中，修改`send()`方法，通过 utxoSet 进行转账并且转账后更新。

```java
 /**
     * 转账
     *
     * @param from
     * @param to
     * @param amount
     */
    private void send(String from, String to, int amount) throws Exception {
        // 检查钱包地址是否合法
        try {
            Base58Check.base58ToBytes(from);
        } catch (Exception e) {
            throw new Exception("ERROR: sender address invalid ! address=" + from);
        }
        // 检查钱包地址是否合法
        try {
            Base58Check.base58ToBytes(to);
        } catch (Exception e) {
            throw new Exception("ERROR: receiver address invalid ! address=" + to);
        }
        if (amount < 1) {
            throw new Exception("ERROR: amount invalid ! ");
        }

        /*
        Blockchain blockchain = Blockchain.createBlockchain(from);
        Transaction transaction = Transaction.newUTXOTransaction(from, to, amount, blockchain);

        blockchain.mineBlock(new Transaction[]{transaction});
        RocksDBUtils.getInstance().closeDB();
        System.out.println("Success!");
        */

        Blockchain blockchain = Blockchain.createBlockchain(from);
        // 新交易
        Transaction transaction = Transaction.newUTXOTransaction(from, to, amount, blockchain);
        // 奖励
        Transaction rewardTx = Transaction.newCoinbaseTX(from, "");
        List<Transaction> transactions = new ArrayList<>();
        transactions.add(transaction);
        transactions.add(rewardTx);
        Block newBlock = blockchain.mineBlock(transactions);
        new UTXOSet(blockchain).update(newBlock);
        log.info("Success!");
    } 
```

现在，让我们来进行最后的测试，在终端输入一下命令：

```java
# 首先重新编译程序
hanru:part8_Transaction2 ruby$ mvn package
# 查询余额
hanru:part8_Transaction2 ruby$ ./blockchain.sh getbalance  -address 19a5Lfex3v9EgFUuNG7n5qR2DaXe5RzUT2
hanru:part8_Transaction2 ruby$ ./blockchain.sh getbalance  -address 17mW1XamRdFZBsnmfh3DxbC5pW1AttJznn
# 转账，19a5Lfex3v9EgFUuNG7n5qR2DaXe5RzUT2 账户因为之前的转账操作，有 11 个 Token
hanru:part8_Transaction2 ruby$ ./blockchain.sh send -from 19a5Lfex3v9EgFUuNG7n5qR2DaXe5RzUT2 -to 17mW1XamRdFZBsnmfh3DxbC5pW1AttJznn -amount 8
# 查询余额
hanru:part8_Transaction2 ruby$ ./blockchain.sh getbalance  -address 19a5Lfex3v9EgFUuNG7n5qR2DaXe5RzUT2
hanru:part8_Transaction2 ruby$ ./blockchain.sh getbalance  -address 17mW1XamRdFZBsnmfh3DxbC5pW1AttJznn 
```

执行结果如下：

![`img.kongyixueyuan.com/010_005_%E8%BD%AC%E8%B4%A6.png`](img/727f3bf62acd9a4640fbd4053af7a6b8.jpg)

## 5\. 总结

本章节中，我们并没有在项目中新增功能，只是做了一些优化。首先加入了奖励 Reward 机制。虽然程序中我们实现的比较建议，仅仅是给发起转账的人奖励 10 个 Token，(如果一次转账多笔交易，只奖励给第一个转账人)。然后我们引入了 UTXO 集，进行项目代码优化。UTXO 集的原理，就是我们将所有区块的未花费的 utxo，单独存储到一个 bucket 中。无论是进行余额查询，还是转账操作，都无需遍历查询所有的区块，查询所有的交易去找未花费 utxo 了，只需要查询 UTXO 集即可。如果是转账操作，转账后需要及时更新 UTXO 集。

[项目源代码](https://github.com/rubyhan1314/BitcoinForJava)