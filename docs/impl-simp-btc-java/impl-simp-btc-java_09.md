# 第九章 数字签名(digital signature)

# 第九章 数字签名(digital signature)

在数学和密码学中，有一个数字签名（digital signature）的概念，算法可以保证：

1.  当数据从发送方传送到接收方时，数据不会被修改；
2.  数据由某一确定的发送方创建；
3.  发送方无法否认发送过数据这一事实。

通过在数据上应用签名算法（也就是对数据进行签名），你就可以得到一个签名，这个签名晚些时候会被验证。生成数字签名需要一个私钥，而验证签名需要一个公钥。签名有点类似于印章，比方说我做了一幅画，完了用印章一盖，就说明了这幅画是我的作品。给数据生成签名，就是给数据盖了章。

## 1\. 课程目标

1.  知道什么是数字签名

2.  知道为什么要进行签名和验签

3.  学会如何进行签名

4.  学会在何处使用签名

## 2\. 项目代码及效果展示

### 2.1 项目代码结构

![`img.kongyixueyuan.com/009_001_%E9%A1%B9%E7%9B%AE%E7%BB%93%E6%9E%84.png`](img/72fc00b7b30bef5a9382052a7b20ae3e.jpg)

### 2.2 项目运行结果

![`img.kongyixueyuan.com/009_002_%E8%BF%90%E8%A1%8C%E6%95%88%E6%9E%9C.gif`](img/9f98dfde9e7989c3740ce3066fdf2dfb.jpg)

其实运行效果和之前的没有区别，因为我们并没有增加新的功能，只是在创建交易的时候，添加了数字签名，在创建新区块的时候，进行签名验证。

## 3\. 创建项目

### 3.1 创建工程

打开 IntelliJ IDEA 的工作空间，将上一个项目代码目录`part6_Wallet`，复制为`part7_Signature`。

然后打开 IntelliJ IDEA 开发工具。

打开工程：`part7_Signature`，并删除 target 目录。然后进行以下修改：

```java
step1：先将项目重新命名为：part7_Signature。
step2：修改 pom.xml 配置文件。
    改为：<artifactId>part7_Signature</artifactId>标签
    改为：<name>part7_Signature Maven Webapp</name> 
```

> 说明：我们每一章节的项目代码，都是在上一个章节上进行添加。所以拷贝上一次的项目代码，然后进行新内容的添加或修改。

### 3.2 代码实现

#### 3.2.1 修改`TXOutput.java`文件

打开`cldy.hanru.blockchain.transaction`包，修改`TXOutput.java`文件。

修改步骤：

```java
修改步骤：
step1：修改字段 pubKeyHash
step2：添加方法 isLockedWithKey()
step3：添加方法 newTXOutput() 
```

修改完后代码如下：

```java
package cldy.hanru.blockchain.transaction;

import cldy.hanru.blockchain.util.Base58Check;
import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.util.Arrays;

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
//    private String scriptPubKey;

    /**
     * 公钥 Hash
     */
    private byte[] pubKeyHash;
    /**
     * 判断解锁数据是否能够解锁交易输出
     *
     * @param unlockingData
     * @return
     */
//    public boolean canBeUnlockedWith(String unlockingData) {
//
//        return this.getScriptPubKey().endsWith(unlockingData);
//    }

    /**
     * 检查交易输出是否能够使用指定的公钥
     *
     * @param pubKeyHash
     * @return
     */
    public boolean isLockedWithKey(byte[] pubKeyHash) {
        return Arrays.equals(this.getPubKeyHash(), pubKeyHash);
    }

    /**
     * 创建交易输出
     *
     * @param value
     * @param address
     * @return
     */
    public static TXOutput newTXOutput(int value, String address) {
        // 反向转化为 byte 数组
        byte[] versionedPayload = Base58Check.base58ToBytes(address);
        byte[] pubKeyHash = Arrays.copyOfRange(versionedPayload, 1, versionedPayload.length);
        return new TXOutput(value, pubKeyHash);
    }
} 
```

#### 3.2.2 修改`TXInput.java`文件

打开`cldy.hanru.blockchain.transaction`包，修改`TXInput.java`文件。

修改步骤：

```java
修改步骤：
step1：修改字段 signature 和 pubKey
step2：添加方法 usesKey() 
```

修改完后代码如下：

```java
package cldy.hanru.blockchain.transaction;

import cldy.hanru.blockchain.util.AddressUtils;
import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.util.Arrays;

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
//    private String scriptSig;

    /**
     * 签名
     */
    private byte[] signature;
    /**
     * 公钥
     */
    private byte[] pubKey;

    /**
     * 判断解锁数据是否能够解锁交易输出
     *
     * @param unlockingData
     * @return
     */
//    public boolean canUnlockOutputWith(String unlockingData) {
//        return this.getScriptSig().endsWith(unlockingData);
//    }

    /**
     * 检查公钥 hash 是否用于交易输入
     *
     * @param pubKeyHash
     * @return
     */
    public boolean usesKey(byte[] pubKeyHash) {
        byte[] lockingHash = AddressUtils.ripeMD160Hash(this.getPubKey());
        return Arrays.equals(lockingHash, pubKeyHash);
    }
} 
```

#### 3.2.3 修改`Transaction.java`文件

打开`cldy.hanru.blockchain.transaction`包，修改`Transaction.java`文件。

修改步骤：

```java
修改步骤：
step1：修改 newCoinbaseTX()方法
step2：添加 getData()
step3：添加 TrimmedCopy()方法，用于备份签名和验证时交易的副本。
step4：添加 Sign()方法，用于进行交易签名
step5：添加 Verify()方法，用于签名验证 
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
        Transaction tx = new Transaction(null, new TXInput[]{txInput}, new TXOutput[]{txOutput});
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

        Transaction newTx = new Transaction(null, txInputs, txOutput);
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

        return new Transaction(this.getTxId(), tmpTXInputs, tmpTXOutputs);
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

#### 3.2.4 修改`BlockChain.java`文件

打开`cldy.hanru.blockchain.block`包，修改`BlockChain.java`文件。

修改步骤：

```java
修改步骤：
step1：修改 getAllSpentTXOs()
step2：修改 findUTXO()
step3：修改 findSpendableOutputs()
step4：添加方法 findTransaction()
step5：添加 signTransaction()
step6：添加 verifyTransactions()
step7：修改 mineBlock()，添加验证 
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
    public void mineBlock(List<Transaction> transactions) throws Exception {
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
    }

    /**
     * 从交易输入中查询区块链中所有已被花费了的交易输出
     *
     * @param pubKeyHash 钱包公钥 Hash
     * @return 交易 ID 以及对应的交易输出下标地址
     * @throws Exception
     */
    private Map<String, int[]> getAllSpentTXOs(byte[] pubKeyHash) {
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
                    if (txInput.usesKey(pubKeyHash)) {
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
//                    if (transaction.getOutputs()[outIndex].canBeUnlockedWith(address)) {
                    if (transaction.getOutputs()[outIndex].isLockedWithKey(pubKeyHash)) {
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
     * @param pubKeyHash 钱包公钥 Hash
     * @return
     */
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

    /**
     * 寻找能够花费的交易
     *
     * @param pubKeyHash 钱包公钥 Hash
     * @param amount  花费金额
     */
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

} 
```

#### 3.2.5 修改`blockchain.sh`脚本文件

最后修改`blockchain.sh`脚本文件，修改后内容如下：

```java
#!/bin/bash

set -e

# Check if the jar has been built.
if [ ! -e target/part7_Signature-jar-with-dependencies.jar ]; then
  echo "Compiling blockchain project to a JAR"
  mvn package -DskipTests
fi

java -jar target/part7_Signature-jar-with-dependencies.jar "$@" 
```

##

## 4\. 数字签名讲解

### 4.1 交易过程

一笔交易就是一个地址的比特币，转移到另一个地址。由于比特币的交易记录全部都是公开的，哪个地址拥有多少比特币，都是可以查到的。因此，支付方是否拥有足够的比特币，完成这笔交易，这是可以轻易验证的。

问题出在怎么防止其他人，冒用你的名义申报交易。举例来说，有人申报了一笔交易：地址 A 向地址 B 支付 10 个比特币。我们怎么知道这个申报是真的，数据是正确的？

比特币协议规定，申报交易的时候，除了交易金额，转出比特币的一方还必须提供以下数据。

*   上一笔交易的 Hash（你从哪里得到这些比特币）
*   本次交易双方的地址
*   支付方的公钥
*   支付方的私钥生成的数字签名

验证这笔交易是否属实，需要三步。

*   第一步，找到上一笔交易，确认支付方的比特币来源。

*   第二步，算出支付方公钥的指纹，确认与支付方的地址一致，从而保证公钥属实。

*   第三步，使用公钥去解开数字签名，保证签名属实、私钥属实。

*   经过上面三步，就可以认定这笔交易是真实的。

![`img.kongyixueyuan.com/0807_%E4%BA%A4%E6%98%93.jpg`](img/bd4fe5796213f683cc78a0a34b9a80ba.jpg)

### 4.1 数字签名

好了，现在我们已经知道了在比特币中证明用户身份的是私钥。而为了保证交易有效性，需要使用数字签名。接下来我们详细谈一下数字签名。

#### 4.1.1 数字签名的概念

所谓数字签名(Digital Signature)（又称公开密钥数字签名、电子签章）。是一种类似写在纸上的普通的物理签名，但是使用了公钥加密领域的技术实现，用于鉴别数字信息的方法。一套数字签名通常定义两种互补的运算，一个用于签名，另一个用于验证。

#### 4.1.2 数字签名如何工作

数字签名由两部分组成：第一部分是使用私钥（签名密钥）从消息（交易）创建签名的算法； 第二部分是允许任何人验证签名的算法。

![`img.kongyixueyuan.com/0804_%E7%AD%BE%E5%90%8D%E6%B5%81%E7%A8%8B.jpg`](img/6e258cf5c0c847131800b48d8c5c5dea.jpg)

#### 4.1.3 数字签名的特性

1、签名不可伪造性；

2、签名不可抵赖的（简直通俗易懂~）；

3、签名可信性，签名的识别和应用相对容易，任何人都可以验证签名的有效性；

4、签名是不可复制的，签名与原文是不可分割的整体；

5、签名消息不可篡改，因为任意比特数据被篡改，其签名便被随之改变，那么任何人可以验证而拒绝接受此签名。

#### 4.1.4 设计数字签名

为了对数据进行签名，我们需要下面两样东西：

1.  要签名的数据
2.  私钥

应用签名算法可以生成一个签名，并且这个签名会被存储在交易输入中。为了对一个签名进行验证，我们需要以下三样东西：

1.  被签名的数据
2.  签名
3.  公钥

简单来说，验证过程可以被描述为：检查签名是由被签名数据加上私钥得来，并且公钥恰好是由该私钥生成。

> 数据签名并不是加密，你无法从一个签名重新构造出数据。这有点像哈希：你在数据上运行一个哈希算法，然后得到一个该数据的唯一表示。签名与哈希的区别在于密钥对：有了密钥对，才有签名验证。但是密钥对也可以被用于加密数据：私钥用于加密，公钥用于解密数据。不过比特币并不使用加密算法。

在比特币中，每一笔交易输入都会由创建交易的人签名。在被放入到一个块之前，必须要对每一笔交易进行验证。除了一些其他步骤，验证意味着：

1.  检查交易输入有权使用来自之前交易的输出
2.  检查交易签名是正确的

如图，对数据进行签名和对签名进行验证的过程大致如下：

![`img.kongyixueyuan.com/0803_%E6%95%B0%E5%AD%97%E7%AD%BE%E5%90%8D.png`](img/f8072a79930d99ecb659f6f0939e6ba7.jpg)

现在来回顾一个交易完整的生命周期：

1.  起初，创世块里面包含了一个 coinbase 交易。在 coinbase 交易中，没有输入，所以也就不需要签名。coinbase 交易的输出包含了一个哈希过的公钥（使用的是 **RIPEMD16(SHA256(PubKey))** 算法）
2.  当一个人发送币时，就会创建一笔交易。这笔交易的输入会引用之前交易的输出。每个输入会存储一个公钥（没有被哈希）和整个交易的一个签名。
3.  比特币网络中接收到交易的其他节点会对该交易进行验证。除了一些其他事情，他们还会检查：在一个输入中，公钥哈希与所引用的输出哈希相匹配（这保证了发送方只能花费属于自己的币）；签名是正确的（这保证了交易是由币的实际拥有者所创建）。
4.  当一个矿工准备挖一个新块时，他会将交易放到块中，然后开始挖矿。
5.  当新块被挖出来以后，网络中的所有其他节点会接收到一条消息，告诉其他人这个块已经被挖出并被加入到区块链。
6.  当一个块被加入到区块链以后，交易就算完成，它的输出就可以在新的交易中被引用。

### 4.2 实现签名

交易必须被签名，因为这是比特币里面保证发送方不会花费属于其他人的币的唯一方式。如果一个签名是无效的，那么这笔交易就会被认为是无效的，因此，这笔交易也就无法被加到区块链中。

我们现在离实现交易签名还差一件事情：用于签名的数据。一笔交易的哪些部分需要签名？又或者说，要对完整的交易进行签名？选择签名的数据相当重要。因为用于签名的这个数据，必须要包含能够唯一识别数据的信息。比如，如果仅仅对输出值进行签名并没有什么意义，因为签名不会考虑发送方和接收方。

考虑到交易解锁的是之前的输出，然后重新分配里面的价值，并锁定新的输出，那么必须要签名以下数据：

1.  存储在已解锁输出的公钥哈希。它识别了一笔交易的“发送方”。
2.  存储在新的锁定输出里面的公钥哈希。它识别了一笔交易的“接收方”。
3.  新的输出值。

> 在比特币中，锁定/解锁逻辑被存储在脚本中，它们被分别存储在输入和输出的 `ScriptSig` 和 `ScriptPubKey` 字段。由于比特币允许这样不同类型的脚本，它对 `ScriptPubKey` 的整个内容进行了签名。

可以看到，我们不需要对存储在输入里面的公钥签名。因此，在比特币里， 所签名的并不是一个交易，而是一个去除部分内容的输入副本，输入里面存储了被引用输出的 `ScriptPubKey` 。

看着有点复杂，来开始写代码吧。先从修改`TXInput`和`TXOutput`的结构开始：

在`transaction`包下，修改`TXInput.go`文件，修改后代码如下：

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
     * 签名
     */
    private byte[] signature;
    /**
     * 公钥
     */
    private byte[] pubKey;

    /**
     * 检查公钥 hash 是否用于交易输入
     *
     * @param pubKeyHash
     * @return
     */
    public boolean usesKey(byte[] pubKeyHash) {
        byte[] lockingHash = AddressUtils.ripeMD160Hash(this.getPubKey());
        return Arrays.equals(lockingHash, pubKeyHash);
    }
} 
```

在这里，我们需要重新设置两个字段，`signature`表示数字签名，`pubKey`表示原始公钥(就是钱包里面的公钥)。

接下来，我们修改`TXOutput.java`文件，修改后代码如下：

```java
public class TXOutput {
    /**
     * 数值金额
     */
    private int value
    /**
     * 公钥 Hash
     */
    private byte[] pubKeyHash;

    /**
     * 检查交易输出是否能够使用指定的公钥
     *
     * @param pubKeyHash
     * @return
     */
    public boolean isLockedWithKey(byte[] pubKeyHash) {
        return Arrays.equals(this.getPubKeyHash(), pubKeyHash);
    }
} 
```

因为在给`TXInput`设置签名需要用到该`TXInput`对应的`TXOutput`的数据，所以要找到这个`TXOutput`所在的`Transaction`。现在我们修改`Blockchain.java`文件，添加一个方法`FindTransactionByTxID()`：

```java
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
```

在查找的时候，需要遍历每个区块查找里面的`Transaction`，根据`txId`判断该`Transaction`是否是我们要找的`Transaction`。

接下来在`Blockchain.java`文件中，继续添加方法，表示签名一笔交易：

```java
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
```

其实签名交易，就是给交易中的每个`TXInput`，设置`signature`字段。所以接下来在`Transaction.java`文件中，添加签名方法，代码如下：

```java
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
```

这个方法接受一个私钥和一个之前交易的 `map`。正如上面提到的，为了对一笔交易进行签名，我们需要获取交易输入所引用的输出，因为我们需要存储这些输出的交易。

这个方法，是签名的核心方法，我们来一步一步地分析该方法：

```java
// coinbase 交易信息不需要签名，因为它不存在交易输入信息
if (this.isCoinbase()) {
    return;
} 
```

`coinbase` 交易因为没有实际输入，所以没有被签名。

```java
// 创建用于签名的交易信息的副本
Transaction txCopy = this.trimmedCopy(); 
```

将会被签署的是修剪后的交易副本，而不是一个完整交易，接下来添加一个方法，用于拷贝一个交易，代码如下：

```java
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

    return new Transaction(this.getTxId(), tmpTXInputs, tmpTXOutputs);
} 
```

这个副本包含了所有的输入和输出，但是 `TXInput.signature` 和 `TXIput.pubKey` 被设置为 `null`。

然后我们需要根据私钥设置签名对象：

```java
Security.addProvider(new BouncyCastleProvider());
Signature ecdsaSign = Signature.getInstance("SHA256withECDSA", BouncyCastleProvider.PROVIDER_NAME);
ecdsaSign.initSign(privateKey); 
```

接下来，我们会迭代副本中每一个输入：

```java
for (int i = 0; i < txCopy.getInputs().length; i++) {
    TXInput txInputCopy = txCopy.getInputs()[i];
    // 获取交易输入 TxID 对应的交易数据
    Transaction prevTx = prevTxMap.get(Hex.encodeHexString(txInputCopy.getTxId()));
    // 获取交易输入所对应的上一笔交易中的交易输出
    TXOutput prevTxOutput = prevTx.getOutputs()[txInputCopy.getTxOutputIndex()];
    txInputCopy.setPubKey(prevTxOutput.getPubKeyHash());
    txInputCopy.setSignature(null);
   ...
} 
```

在每个输入中，`signature` 被设置为 `null` (仅仅是一个双重检验)，`pubcKey` 被设置为所引用输出的 `PubKeyHash`。现在，除了当前交易，其他所有交易都是“空的”，也就是说他们的 `signature` 和 `pubKey` 字段被设置为 `null`。因此，**输入是被分开签名的**，尽管这对于我们的应用并不十分紧要，但是比特币允许交易包含引用了不同地址的输入。

```java
// 得到要签名的数据
byte[] signData = txCopy.getData();
txInputCopy.setPubKey(null); 
```

`getData()` 方法对交易进行序列化，并使用 SHA-256 算法进行哈希。哈希后的结果就是我们要签名的数据。在获取完哈希，我们应该重置 `PublicKey` 字段，以便于它不会影响后面的迭代。

现在，关键点：

```java
 // 对整个交易信息仅进行签名
ecdsaSign.update(signData);
byte[] signature = ecdsaSign.sign();

// 将整个交易数据的签名赋值给交易输入，因为交易输入需要包含整个交易信息的签名
// 注意是将得到的签名赋值给原交易信息中的交易输入
this.getInputs()[i].setSignature(signature); 
```

我们对 data 进行签名。一个 ECDSA 签名就是一对数字，我们对这对数字连接起来，并存储在输入的 `Signature` 字段。

### 4.3 签名验证

现在，验证函数：

在`Transaction.java`文件中，添加验证签名方法，代码如下:

```java
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
```

这个方法十分直观。首先，我们需要同一笔交易的副本：

```java
// 创建用于签名验证的交易信息的副本
Transaction txCopy = this.trimmedCopy(); 
```

然后，我们需要相同的签名对象：

```java
Security.addProvider(new BouncyCastleProvider());
ECParameterSpec ecParameters = ECNamedCurveTable.getParameterSpec("secp256k1");
KeyFactory keyFactory = KeyFactory.getInstance("ECDSA", BouncyCastleProvider.PROVIDER_NAME);
Signature ecdsaVerify = Signature.getInstance("SHA256withECDSA", BouncyCastleProvider.PROVIDER_NAME); 
```

接下来，我们检查每个输入中的签名：

```java
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
```

这个部分跟 `sign()` 方法一模一样，因为在验证阶段，我们需要的是与签名相同的数据。

```java
// 使用椭圆曲线 x,y 点去生成公钥 Key
BigInteger x = new BigInteger(1, Arrays.copyOfRange(txInput.getPubKey(), 1, 33));
BigInteger y = new BigInteger(1, Arrays.copyOfRange(txInput.getPubKey(), 33, 65));
ECPoint ecPoint = ecParameters.getCurve().createPoint(x, y); 
```

这里我们解包存储在 `TXInput.publicKey` 中的值，因为一个公钥就是一对坐标。我们之前为了存储将它们连接在一起，现在我们需要对它们进行解包在 `initVerify()` 函数中使用。

```java
 ...
    ECPublicKeySpec keySpec = new ECPublicKeySpec(ecPoint, ecParameters);
    PublicKey publicKey = keyFactory.generatePublic(keySpec);
    ecdsaVerify.initVerify(publicKey);
    ecdsaVerify.update(signData);
    if (!ecdsaVerify.verify(txInput.getSignature())) {
        return false;
    }
    return true;
} 
```

在这里：我们使用从输入提取的公钥用于设置`ecdsaVerify`的初始化信息，并且设置了要验证的签名的数据。然后来验证签名。如果所有的输入都被验证，返回 `true`；如果有任何一个验证失败，返回 `false`.

接下来，我们在`Blockchain.java`中添加验证方法`verifyTransactions()`：

```java
/**
    * 交易签名验证
    *
    * @param tx
*/
private boolean verifyTransactions(Transaction tx) throws Exception {
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
```

在一笔交易被放入一个块之前进行验证，所以接下来我们需要修改`BlockChain.java`文件中的`MineNewBlock()`方法，

```java
/**
     * 打包交易，进行挖矿
     *
     * @param transactions
     */
public void mineBlock(Transaction[] transactions) throws Exception {
    // 挖矿前，先验证交易记录
    for (Transaction tx : transactions) {
        if (!this.verifyTransactions(tx)) {
            throw new Exception("ERROR: Fail to mine block ! Invalid transaction ! ");
        }
    }
    String lastBlockHash = RocksDBUtils.getInstance().getLastBlockHash();
    if (lastBlockHash == null) {
        throw new Exception("ERROR: Fail to get last block hash ! ");
    }
    Block block = Block.newBlock(lastBlockHash, transactions);
    this.addBlock(block);
} 
```

至此，我们已经在项目中，添加了交易的签名和签名验证。

`main.go`文件无需修改，接下来我们测试一下代码：还是进行创建钱包地址，创建创世区块，然后转账交易等。

在终端中输入以下命令:

```java
hanru:part7_Signature ruby$ ./blockchain.sh h
hanru:part7_Signature ruby$ ./blockchain.sh createaddress
hanru:part7_Signature ruby$ ./blockchain.sh createaddress
hanru:part7_Signature ruby$ ./blockchain.sh printaddresses
hanru:part7_Signature ruby$ ./blockchain.sh createblockchain -address 17aAQuo5A8xk9hV7NRp6Mc3ambV54gjMKX
hanru:part7_Signature ruby$ ./blockchain.sh getbalance -address 17aAQuo5A8xk9hV7NRp6Mc3ambV54gjMKX 
```

运行结果如下：

![`img.kongyixueyuan.com/009_003_%E8%BF%90%E8%A1%8C1.png`](img/dd52601c870948ea08598f58ed7779d4.jpg)

继续输入以下命令：

```java
hanru:part7_Signature ruby$ ./blockchain.sh send -from 17aAQuo5A8xk9hV7NRp6Mc3ambV54gjMKX -to 1EGUjAdhqWTHLxDsKrMUJyNv2KMM4zxxL2 -amount 4
hanru:part7_Signature ruby$ ./blockchain.sh getbalance -address 17aAQuo5A8xk9hV7NRp6Mc3ambV54gjMKX
hanru:part7_Signature ruby$ ./blockchain.sh getbalance -address 1EGUjAdhqWTHLxDsKrMUJyNv2KMM4zxxL2
hanru:part7_Signature ruby$ ./blockchain.sh printchain 
```

运行结果如下:

![`img.kongyixueyuan.com/009_003_%E8%BF%90%E8%A1%8C2.png`](img/d032b746169e8685a2bca2fa8b03e999.jpg)

## 5\. 总结

通过本章节的学习，我们知道了什么是签名，为何签名，以及如何签名。只有转账人才能生成的一段防伪造的字符串。通过验证该字符串，一方面证明该交易是转出方本人发起的，另一方面证明交易信息在传输过程中没有被更改。数字签名由：数字摘要和非对称加密技术组成。数字摘要把交易信息 hash 成固定长度的字符串，再用私钥对 hash 后的交易信息进行加密形成数字签名。交易中，需要将完整的交易信息和数字签名一起广播给矿工。矿工节点用转账人公钥对签名验证，验证成功说明该交易确实是转账人发起的；矿工节点将交易信息进行 hash 后与签名中的交易信息摘要进行比对，如果一致，则说明交易信息在传输过程中没有被篡改。

[项目源代码](https://github.com/rubyhan1314/BitcoinForJava)