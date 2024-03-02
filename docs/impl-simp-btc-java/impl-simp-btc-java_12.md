# 第十二章 实现简易版区块链-网络

# 第十二章 网络(network)

到目前为止，我们所构建的原型已经具备了区块链所有的关键特性：匿名，安全，随机生成的地址；区块链数据存储；工作量证明系统；可靠地存储交易。尽管这些特性都不可或缺，但是仍有不足。能够使得这些特性真正发光发热，使得加密货币成为可能的，是**网络（network）**。如果实现的这样一个区块链仅仅运行在单一节点上，有什么用呢？如果只有一个用户，那么这些基于密码学的特性，又有什么用呢？正是由于网络，才使得整个机制能够运转和发光发热。

你可以将这些区块链特性认为是规则（rule），类似于人类在一起生活，繁衍生息建立的规则，一种社会安排。区块链网络就是一个程序社区，里面的每个程序都遵循同样的规则，正是由于遵循着同一个规则，才使得网络能够长存。类似的，当人们都有着同样的想法，就能够将拳头攥在一起构建一个更好的生活。如果有人遵循着不同的规则，那么他们就将生活在一个分裂的社区（州，公社，等等）中。同样的，如果有区块链节点遵循不同的规则，那么也会形成一个分裂的网络。

**重点在于**：如果没有网络，或者大部分节点都不遵守同样的规则，那么规则就会形同虚设，毫无用处！

> 声明：不幸的是，本文并没有实现一个真实的 P2P 网络原型。但是会展示一个最常见的场景，这个场景涉及不同类型的节点。继续改进这个场景，将它实现为一个 P2P 网络，对你来说是一个很好的挑战和实践！除了本文的场景，我们也无法保证在其他场景将会正常工作。抱歉！
> 
> 本文的代码实现变化很大。

## 1\. 课程目标

1.  学会通过动态设置环境变量设置 NODE_ID
2.  了解节点之间的工作和消息传递的原理
3.  主节点和钱包节点以及矿工节点之间通信的流程
4.  项目中通过代码实现消息传递

## 2\. 项目代码及效果展示

### 2.1 项目代码结构

![`img.kongyixueyuan.com/012_001_%E9%A1%B9%E7%9B%AE%E7%BB%93%E6%9E%84.gif`](img/b0f41e310a79a39c4a5d1768f7502227.jpg)

### 2.2 项目运行结果

钱包节点同步数据，效果图 1：

![`img.kongyixueyuan.com/012_023_%E8%BF%90%E8%A1%8C%E7%BB%93%E6%9E%9C1.gif`](img/03b6231e3e04a0b4604cfe68036bb2fd.jpg)

矿工节点挖矿，效果图 2：

![`img.kongyixueyuan.com/012_024_%E8%BF%90%E8%A1%8C%E7%BB%93%E6%9E%9C2.gif`](img/dac9864868a3910308723110e834c72c.jpg)

## 3\. 创建项目

### 3.1 创建工程

打开 IntelliJ IDEA 的工作空间，将上一个项目代码目录`part9_Merkle`，复制为`part10_Network`。

然后打开 IntelliJ IDEA 开发工具。

打开工程：`part10_Network`，并删除 target 目录。然后进行以下修改：

```java
step1：先将项目重新命名为：part10_Network。
step2：修改 pom.xml 配置文件。
    改为：<artifactId>part10_Network</artifactId>标签
    改为：<name>part10_Network Maven Webapp</name> 
```

> 说明：我们每一章节的项目代码，都是在上一个章节上进行添加。所以拷贝上一次的项目代码，然后进行新内容的添加或修改。

###

### 3.2 代码实现

#### 3.2.1 修改`CLI.java`文件

打开`cldy.hanru.blockchain.cli`包。修改`CLI.java`文件。

修改步骤：

```java
修改步骤：
step1：修改 Run()方法，
    设置 NODE_ID
    添加新的命令，用于启动节点，
    以及修改转账 send 命令。 
```

修改完后代码如下：

```java
package cldy.hanru.blockchain.cli;

import cldy.hanru.blockchain.block.Block;
import cldy.hanru.blockchain.block.Blockchain;
import cldy.hanru.blockchain.net.Server;
import cldy.hanru.blockchain.net.ServerSend;
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
        Option mine = Option.builder("mine").hasArg(true).desc("mine a block").build();

        options.addOption(address);
        options.addOption(sendFrom);
        options.addOption(sendTo);
        options.addOption(sendAmount);
        options.addOption(mine);
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
        System.out.println("  send -from FROM -to TO -amount AMOUNT -mine MINENOW - Send AMOUNT of coins from FROM address to TO");
        System.out.println("  startnode -address ADDRESS -- 启动节点服务器，并且指定挖矿奖励的地址.");
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

        /*
    获取节点 ID
    解释：返回当前进程的环境变量 varname 的值,若变量没有定义时返回 nil
    export NODE_ID=8888

    每次打开一个终端，都需要设置 NODE_ID 的值。
    变量名 NODE_ID，可以更改别的。
     */

        Map<String, String> map = System.getenv();

        String nodeID = map.get("NODE_ID");
        if (nodeID == "") {
            System.out.println("NODE_ID 环境变量没有设置。。");
            System.exit(0);
        }

        System.out.println("NODE_ID：" + nodeID);

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
                this.createBlockchain(createblockchainAddress, nodeID);
                break;
            case "getbalance":
                String getBalanceAddress = cmd.getOptionValue("address");
                if (StringUtils.isBlank(getBalanceAddress)) {
                    help();
                }
                try {
                    this.getBalance(getBalanceAddress, nodeID);
                } catch (Exception e) {
                    e.printStackTrace();
                }finally {
                    RocksDBUtils.getInstance(nodeID).closeDB();
                }
                break;
            case "send":
                String sendFrom = cmd.getOptionValue("from");
                String sendTo = cmd.getOptionValue("to");
                String sendAmount = cmd.getOptionValue("amount");
                String mineNow = cmd.getOptionValue("mine"); //是否立即挖矿
                if (StringUtils.isBlank(sendFrom) ||
                        StringUtils.isBlank(sendTo) ||
                        !NumberUtils.isDigits(sendAmount)) {
                    help();
                }
                try {
                    this.send(sendFrom, sendTo, Integer.valueOf(sendAmount), nodeID, Boolean.parseBoolean(mineNow));
                } catch (Exception e) {
                    e.printStackTrace();
                } finally {
                    RocksDBUtils.getInstance(nodeID).closeDB();
                }
                break;
            case "printchain":
                this.printChain(nodeID);
                break;
            case "h":
                this.help();
                break;

            case "createwallet":
                try {
                    this.createWallet(nodeID);
                } catch (Exception e) {
                    e.printStackTrace();
                }
                break;
            case "printaddresses":
                try {
                    this.printAddresses(nodeID);
                } catch (Exception e) {
                    e.printStackTrace();
                }
                break;
            case "startnode":
                String minerAddress = cmd.getOptionValue("address");
                try {
                    start(nodeID, minerAddress);
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
    private void createBlockchain(String address, String nodeID) {

        Blockchain blockchain = Blockchain.createBlockchain(address,nodeID);
        UTXOSet utxoSet = new UTXOSet(blockchain);
        utxoSet.reIndex();
        log.info("Done ! ");
    }

    /**
     * 打印出区块链中的所有区块
     */
    private void printChain( String nodeID) {
//            Blockchain blockchain = Blockchain.newBlockchain();
        Blockchain blockchain = null;
        try {
            blockchain = Blockchain.initBlockchainFromDB(nodeID);
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
    private void getBalance(String address, String nodeID) throws Exception {

        // 检查钱包地址是否合法
        try {
            Base58Check.base58ToBytes(address);
        } catch (Exception e) {
            throw new Exception("ERROR: invalid wallet address");
        }
        Blockchain blockchain = Blockchain.createBlockchain(address,nodeID);
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
    private void send(String from, String to, int amount,String nodeID, boolean mineNow) throws Exception {
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

        Blockchain blockchain = Blockchain.createBlockchain(from,nodeID);
        // 新交易
        Transaction transaction = Transaction.newUTXOTransaction(from, to, amount, blockchain, nodeID);
        if(mineNow){
            // 奖励
            Transaction rewardTx = Transaction.newCoinbaseTX(from, "");
            List<Transaction> transactions = new ArrayList<>();
            transactions.add(transaction);
            transactions.add(rewardTx);
            Block newBlock = blockchain.mineBlock(transactions);
            new UTXOSet(blockchain).update(newBlock);
            log.info("Success!");
        }else{
            //矿工节点处理。。
            System.out.println("由矿工节点处理。。。。");
            ServerSend.sendTx(Server.knowNodes.get(0), transaction);
        }

    }

    /**
     * 创建钱包
     *
     * @throws Exception
     */
    private void createWallet(String nodeID) throws Exception {
        Wallet wallet = WalletUtils.getInstance(nodeID).createWallet();
        System.out.println("wallet address : " + wallet.getAddress());
    }

    /**
     * 打印钱包地址
     *
     * @throws Exception
     */
    private void printAddresses(String nodeID) throws Exception {
        Set<String> addresses = WalletUtils.getInstance(nodeID).getAddresses();
        if (addresses == null || addresses.isEmpty()) {
            System.out.println("There isn't address");
            return;
        }
        for (String address : addresses) {
            System.out.println("Wallet address: " + address);
        }
    }

    /**
     * 启动节点
     * @param nodeID
     * @param minerAddress
     * @throws Exception
     */
    private void start(String nodeID, String minerAddress) throws Exception {

        System.out.println("minerAddress:" + minerAddress);
        if (null == minerAddress || minerAddress == "") {

        } else {
            // 检查钱包地址是否合法
            try {
                Base58Check.base58ToBytes(minerAddress);
            } catch (Exception e) {
                throw new Exception("ERROR: receiver address invalid ! address=" + minerAddress);
            }
        }

        System.out.printf("启动服务器：localhost:%s\n", nodeID);
        Server.startServer(nodeID, minerAddress);

    }
} 
```

#### 3.2.2 修改`WalletUtils.java`文件

打开`cldy.hanru.blockchain.wallet`。修改`WalletUtils.java`文件。

修改步骤：

```java
修改步骤：
step1：修改 private  static String WALLET_FILE = "wallet_%s.dat";
step2：修改 getInstance()方法，添加 NODE_ID
step3：修改构造函数：WalletUtils()，添加 NODE_ID
step4：修改 initWalletFile()方法，添加 NODE_ID 
```

修改完后代码如下：

```java
package cldy.hanru.blockchain.wallet;

import cldy.hanru.blockchain.util.Base58Check;
import lombok.AllArgsConstructor;
import lombok.Cleanup;
import lombok.Data;
import lombok.NoArgsConstructor;

import javax.crypto.Cipher;
import javax.crypto.CipherInputStream;
import javax.crypto.CipherOutputStream;
import javax.crypto.SealedObject;
import javax.crypto.spec.SecretKeySpec;
import java.io.*;
import java.util.HashMap;
import java.util.Map;
import java.util.Set;

/**
 * 钱包工具类
 *
 * @author hanru
 */
public class WalletUtils {

    /**
     * 钱包文件
     */
    private  static String WALLET_FILE = "wallet_%s.dat";
    /**
     * 加密算法
     */
    private static final String ALGORITHM = "AES";
    /**
     * 密文
     */
    private static final byte[] CIPHER_TEXT = "2oF@5sC%DNf32y!TmiZi!tG9W5rLaniD".getBytes();

    /**
     * 定义内部类：钱包存储对象
     */
    @Data
    @NoArgsConstructor
    @AllArgsConstructor
    public static class Wallets implements Serializable {

        private static final long serialVersionUID = -4824448861236743729L;
        private Map<String, Wallet> walletMap = new HashMap<>();

        /**
         * 添加钱包
         *
         * @param wallet
         */
        private void addWallet(Wallet wallet) {
            try {
                this.walletMap.put(wallet.getAddress(), wallet);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }

        /**
         * 获取所有的钱包地址
         *
         * @return
         * @throws Exception
         */
        Set<String> getAddresses() throws Exception {
            if (walletMap == null) {
                throw new Exception("ERROR: Fail to get addresses ! There isn't address ! ");
            }
            return walletMap.keySet();
        }

        /**
         * 获取钱包数据
         *
         * @param address 钱包地址
         * @return
         */
        Wallet getWallet(String address) throws Exception {
            // 检查钱包地址是否合法
            try {
                Base58Check.base58ToBytes(address);
            } catch (Exception e) {
                throw new Exception("ERROR: invalid wallet address");
            }
            Wallet wallet = walletMap.get(address);
            if (wallet == null) {
                throw new Exception("ERROR: Fail to get wallet ! wallet don't exist ! ");
            }
            return wallet;
        }
    }

    /**
     * 钱包工具实例
     */
    private volatile static WalletUtils instance;

    public static WalletUtils getInstance(String nodeID) {
        if (instance == null) {
            synchronized (WalletUtils.class) {
                if (instance == null) {
                    instance = new WalletUtils(nodeID);
                }
            }
        }
        return instance;
    }

    private WalletUtils(String nodeID) {
        initWalletFile(nodeID);
    }

    /**
     * 初始化钱包文件
     */
    private void initWalletFile(String nodeID) {
        WALLET_FILE = String.format(WALLET_FILE, nodeID);
        File file = new File(WALLET_FILE);
        if (!file.exists()) {
            this.saveToDisk(new Wallets());
        } else {
            this.loadFromDisk();
        }
    }

    /**
     * 保存钱包数据
     */
    private void saveToDisk(Wallets wallets) {
        try {
            if (wallets == null) {
                throw new Exception("ERROR: Fail to save wallet to file ! data is null ! ");
            }
            SecretKeySpec sks = new SecretKeySpec(CIPHER_TEXT, ALGORITHM);
            // Create cipher
            Cipher cipher = Cipher.getInstance(ALGORITHM);
            cipher.init(Cipher.ENCRYPT_MODE, sks);
            SealedObject sealedObject = new SealedObject(wallets, cipher);
            // Wrap the output stream
            @Cleanup CipherOutputStream cos = new CipherOutputStream(
                    new BufferedOutputStream(new FileOutputStream(WALLET_FILE)), cipher);
            @Cleanup ObjectOutputStream outputStream = new ObjectOutputStream(cos);
            outputStream.writeObject(sealedObject);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    /**
     * 加载钱包数据
     */
    private Wallets loadFromDisk() {
        try {
            SecretKeySpec sks = new SecretKeySpec(CIPHER_TEXT, ALGORITHM);
            Cipher cipher = Cipher.getInstance(ALGORITHM);
            cipher.init(Cipher.DECRYPT_MODE, sks);
            @Cleanup CipherInputStream cipherInputStream = new CipherInputStream(
                    new BufferedInputStream(new FileInputStream(WALLET_FILE)), cipher);
            @Cleanup ObjectInputStream inputStream = new ObjectInputStream(cipherInputStream);
            SealedObject sealedObject = (SealedObject) inputStream.readObject();
            return (Wallets) sealedObject.getObject(cipher);
        } catch (Exception e) {
            e.printStackTrace();
        }
        throw new RuntimeException("Fail to load wallet file from disk ! ");
    }

    /**
     * 创建钱包
     *
     * @return
     */
    public Wallet createWallet() {
        Wallet wallet = new Wallet();
        Wallets wallets = this.loadFromDisk();
        wallets.addWallet(wallet);
        this.saveToDisk(wallets);
        return wallet;
    }

    /**
     * 获取钱包数据
     *
     * @param address 钱包地址
     * @return
     */
    public Wallet getWallet(String address) throws Exception {
        Wallets wallets = this.loadFromDisk();
        return wallets.getWallet(address);
    }

    /**
     * 获取所有的钱包地址
     *
     * @return
     * @throws Exception
     */
    public Set<String> getAddresses() throws Exception {
        Wallets wallets = this.loadFromDisk();
        return wallets.getAddresses();
    }
} 
```

#### 3.2.3 修改`Transaction.java`文件

打开`cldy.hanru.blockchain.transaction`包。修改`Transaction.java`文件。

修改步骤：

```java
修改步骤：
step1：修改 newUTXOTransaction()方法，添加 NODE_ID 
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
    public static Transaction newUTXOTransaction(String from, String to, int amount, Blockchain blockchain,String nodeID) throws Exception {

        // 获取钱包
        Wallet senderWallet = WalletUtils.getInstance(nodeID).getWallet(from);
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

#### 3.2.4 修改`Blockchain.java`文件

打开`cldy.hanru.blockchain.block`包。修改`Blockchain.java`文件。

修改步骤：

```java
修改步骤：
step1：添加字段 private String nodeID;
step2：修改：public static Blockchain createBlockchain(String address,String nodeID)，添加 NODE_ID
step3：修改：public static Blockchain initBlockchainFromDB(String nodeID)，添加 NODE_ID 
```

修改完后代码如下：

```java
package cldy.hanru.blockchain.block;

import cldy.hanru.blockchain.store.RocksDBUtils;
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

import java.util.*;

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
     * 当前节点的 nodeID
     */
    private String nodeID;

    /**
     * 创建区块链，createBlockchain
     *
     * @param address
     * @return
     */
    public static Blockchain createBlockchain(String address, String nodeID) {

        String lastBlockHash = RocksDBUtils.getInstance(nodeID).getLastBlockHash();
        if (StringUtils.isBlank(lastBlockHash)) {
            //对应的 bucket 不存在，说明是第一次获取区块链实例
            // 创建 coinBase 交易
            Transaction coinbaseTX = Transaction.newCoinbaseTX(address, "");
            Block genesisBlock = Block.newGenesisBlock(coinbaseTX);
//            Block genesisBlock = Block.newGenesisBlock();
            lastBlockHash = genesisBlock.getHash();
            RocksDBUtils.getInstance(nodeID).putBlock(genesisBlock);
            RocksDBUtils.getInstance(nodeID).putLastBlockHash(lastBlockHash);

        }
        return new Blockchain(lastBlockHash, nodeID);
    }

    /**
     * 根据 block，添加区块
     *
     * @param block
     */
    public void addBlock(Block block) {

        RocksDBUtils.getInstance(nodeID).putLastBlockHash(block.getHash());
        RocksDBUtils.getInstance(nodeID).putBlock(block);
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
            Block lastBlock = RocksDBUtils.getInstance(nodeID).getBlock(currentBlockHash);
            if (lastBlock == null) {
                return false;
            }
            // 如果是创世区块
            if (ByteUtils.ZERO_HASH.equals(lastBlock.getPrevBlockHash())) {
                return true;
            }
            return RocksDBUtils.getInstance(nodeID).getBlock(lastBlock.getPrevBlockHash()) != null;
        }

        /**
         * 迭代获取区块
         *
         * @return
         */
        public Block next() {
            Block currentBlock = RocksDBUtils.getInstance(nodeID).getBlock(currentBlockHash);
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
        String lastBlockHash = RocksDBUtils.getInstance(nodeID).getLastBlockHash();
        Block lastBlock = RocksDBUtils.getInstance(nodeID).getBlock(lastBlockHash);
        if (lastBlockHash == null) {
            throw new Exception("ERROR: Fail to get last block hash ! ");
        }
        Block block = Block.newBlock(lastBlockHash, transactions, lastBlock.getHeight() + 1);
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
    public static Blockchain initBlockchainFromDB(String nodeID) throws Exception {
        String lastBlockHash = RocksDBUtils.getInstance(nodeID).getLastBlockHash();
        if (lastBlockHash == null) {
            throw new Exception("ERROR: Fail to init blockchain from db. ");
        }
        return new Blockchain(lastBlockHash, nodeID);
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
    public boolean verifyTransactions(Transaction tx) throws Exception {
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

    //----------
    //获取最新区块的高度
    public long getBestHeight() {

        Block block = this.getBlockchainIterator().next();

        return block.getHeight();
    }

    //获取所有区块的 hash
    public List<String> getBlockHashes() {

        BlockchainIterator it = this.getBlockchainIterator();

        List<String> blockHashs = new ArrayList<>();

        while (it.hashNext()) {
            Block block = it.next();
            blockHashs.add(0, block.getHash());
        }

        return blockHashs;
    }

    //根据 hash 获取区块
    public Block getBlock(String blockHash) {
        Block block = RocksDBUtils.getInstance(this.nodeID).getBlock(blockHash);
        return block;

    }

    //节点传来的区块，保存到数据库中
    public void saveBlock(Block newBlock) {
        BlockchainIterator it = this.getBlockchainIterator();
        while (it.hashNext()) {
            Block block = it.next();
            if (block.getHash().equals(newBlock.getHash())) {
                System.out.println("区块已经存在，无需存储。。。");
                return;
            }
        }

        //存储
        RocksDBUtils.getInstance(nodeID).putBlock(newBlock);

        Block lastBlock = RocksDBUtils.getInstance(nodeID).getLastBlock();
        if (lastBlock.getHeight() < newBlock.getHeight()) {
            RocksDBUtils.getInstance(nodeID).putLastBlockHash(newBlock.getHash());
            this.lastBlockHash = newBlock.getHash();
        }
        System.out.printf("Added block %s\n", newBlock.getHash());
    }
} 
```

#### 3.2.5 新建`ServerConst.java`文件

新建`cldy.hanru.blockchain.net`包。新建`ServerConst.java`文件。并添加代码如下:

```java
package cldy.hanru.blockchain.net;

/**
 * 节点服务的常量
 * @author hanru
 */
public class ServerConst {

    public static int NODE_VERSION = 1;

    // 命令
    public static final String COMMAND_VERSION = "version";
    public static final String COMMAND_GETBLOCKS = "getblocks";
    public static final String COMMAND_ADDR = "addr";
    public static final String COMMAND_BLOCK = "block";
    public static final String COMMAND_INV = "inv";
    public static final String COMMAND_GETDATA = "getdata";
    public static final String COMMAND_TX = "tx";

    // 类型
    public static final String BLOCK_TYPE = "block";
    public static final String TX_TYPE = "tx";

} 
```

#### 3.2.6 新建`Version.java`文件

在`cldy.hanru.blockchain.net`包。新建`Version.java`文件。并添加代码如下:

```java
package cldy.hanru.blockchain.net;

import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

/**
 * 区块链版本
 * @author hanru
 */
@AllArgsConstructor
@Data
@NoArgsConstructor
public class Version {
    /**
     * 版本
     */
    public  int version ;
    /**
     * 当前节点区块的高度
     */
    private long bestHeight;
    /**
     * 当前节点的地址
     */
    private String nodeAddress;
} 
```

#### 3.2.7 新建`GetBlocks.java`文件

在`cldy.hanru.blockchain.net`包。新建`GetBlocks.java`文件。并添加代码如下:

```java
package cldy.hanru.blockchain.net;

import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

@NoArgsConstructor
@AllArgsConstructor
@Data
/**
 * @author hanru
 */
public class GetBlocks {
    //getblocks 意为 “给我看一下你有什么区块”（在比特币中，这会更加复杂）
    private String     addrFrom ;
} 
```

#### 3.2.8 新建`Inv.java`文件

在`cldy.hanru.blockchain.net`包。新建`Inv.java`文件。并添加代码如下:

```java
package cldy.hanru.blockchain.net;

import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.util.List;

@AllArgsConstructor
@NoArgsConstructor
@Data
/**
 * @author hanru
 */
public class Inv {

    String addrFrom; //自己的地址
    String type; //类型 block tx
    List<String> items; //hash 的列表
} 
```

####

#### 3.2.9 新建`GetData.java`文件

在`cldy.hanru.blockchain.net`包。新建`GetData.java`文件。并添加代码如下:

```java
package cldy.hanru.blockchain.net;

import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

@AllArgsConstructor
@NoArgsConstructor
@Data
/**
 * author hanru
 */
public class GetData {

    //用于某个块或交易的请求，它可以仅包含一个块或交易的 ID。
    String addrFrom;
    String type;
    String hash;

} 
```

####

#### 3.2.10 新建`BlockData.java`文件

在`cldy.hanru.blockchain.net`包。新建`BlockData.java`文件。并添加代码如下:

```java
package cldy.hanru.blockchain.net;

import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

@AllArgsConstructor
@NoArgsConstructor
@Data
/**
 * @author hanru
 */
public class BlockData {
    String addrFrom;
    byte[] blockData;
} 
```

####

#### 3.2.11 新建`TransactionData.java`文件

在`cldy.hanru.blockchain.net`包。新建`TransactionData.java`文件。并添加代码如下:

```java
package cldy.hanru.blockchain.net;

import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

@AllArgsConstructor
@NoArgsConstructor
@Data
/**
 * @author hanru
 */
public class TransactionData {

    String addrFrom;
    byte[] txData;

} 
```

#### 3.2.12 新建`Server.java`文件

在`cldy.hanru.blockchain.net`包。新建`Server.java`文件。并添加代码如下:

```java
package cldy.hanru.blockchain.net;

import cldy.hanru.blockchain.block.Blockchain;

import java.io.IOException;
import java.io.InputStream;
import java.net.ServerSocket;
import java.net.Socket;
import java.util.ArrayList;
import java.util.List;

/**
 * 节点服务器
 * @author hanru
 */
public class Server {
    static{
        knowNodes = new ArrayList<>();
        Server.knowNodes.add("localhost:3000");
    }

    //存储节点的地址
    public static List<String> knowNodes ;

    public static String nodeAddress; //全局变量，节点地址

    public static String minerAddress ="";

    public static void startServer(String nodeID, String minerAdd) {
//        knowNodes.add("localhost:3000");//localhost:3000 主节点的地址
        System.out.println("knowNodes(0):"+knowNodes.get(0));
        // 当前节点的 IP 地址
        nodeAddress = String.format("localhost:%s", nodeID);
        minerAddress = minerAdd;
        System.out.printf("nodeAddress:%s,minerAddress:%s\n", nodeAddress, minerAddress);
        ServerSocket server = null;
        Socket socket = null;
        try {
            server = new ServerSocket(Integer.parseInt(nodeID));
            System.out.println("服务端已经建立，等待客户端的连接请求。。。");

            //获取 blockchain
            Blockchain bc = Blockchain.initBlockchainFromDB(nodeID);
            System.out.println("blockchain-->"+bc.getNodeID());

            // 第一个终端：端口为 3000,启动的就是主节点
            // 第二个终端：端口为 3001，钱包节点
            // 第三个终端：端口号为 3002，矿工节点
            if (!knowNodes.get(0).equals(nodeAddress)) {
                // 此节点是钱包节点或者矿工节点，需要向主节点发送请求同步数据
//                System.out.printf("knowNodes:%s\n", knowNodes.get(0));
                ServerSend.sendVersion(knowNodes.get(0), bc);
            }

            while (true) {
                // 收到的数据的格式是固定的，12 字节+长度+ 结构体字节数组
                socket = server.accept();//该方法是阻塞式
                System.out.println("已有客户端连入。。客户端的 ip：" + socket.getInetAddress() + "客户端的 port：" + socket.getPort());
                //业务处理：放在子线程中
                new Thread(new ServerThread(socket, bc)).start();

            }

        } catch (IOException e) {
            e.printStackTrace();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {

            if (socket != null) {//可以省略不写
                try {
                    socket.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }
    }

    /**
     * 判断节点地址是否在集合中
     * @param addr
     * @return
     */
    public static boolean nodeIsKnown(String addr) {

        for (String address : knowNodes) {
            if (addr == address) {
                return true;
            }
        }
        return false;

    }

} 
```

#### 3.2.13 新建`ServerSend.java`文件

在`cldy.hanru.blockchain.net`包。新建`ServerSend.java`文件。并添加代码如下:

```java
package cldy.hanru.blockchain.net;

import cldy.hanru.blockchain.block.Block;
import cldy.hanru.blockchain.block.Blockchain;
import cldy.hanru.blockchain.transaction.Transaction;
import cldy.hanru.blockchain.util.ByteUtils;
import cldy.hanru.blockchain.util.CommandUtils;
import cldy.hanru.blockchain.util.SerializeUtils;

import java.util.List;

/**
 * 发送消息
 * @author hanru
 */
public class ServerSend {

    //发送版本
    public static void sendVersion(String toAddress, Blockchain bc) {
        long bestHeight = bc.getBestHeight();
        Version version = new Version(ServerConst.NODE_VERSION, bestHeight, Server.nodeAddress);
        byte[] payload = SerializeUtils.serialize(version);
        System.out.println("sendVersion:"+payload.length);
        //version
        ServerSocketClient.sendData(toAddress, ServerConst.COMMAND_VERSION, payload);
    }

    //COMMAND_GETBLOCKS
    public static void sendGetBlocks(String toAddress) {
        GetBlocks getBlocks = new GetBlocks(Server.nodeAddress);
        byte[] payload = SerializeUtils.serialize(getBlocks);
        System.out.println("sendGetBlocks:"+payload.length);
        ServerSocketClient.sendData(toAddress, ServerConst.COMMAND_GETBLOCKS, payload);

    }

    // 主节点将自己的所有的区块 hash 发送给钱包节点
    //COMMAND_BLOCK
    public static void sendInv(String toAddress, String kind, List<String> blockHashs) {
        Inv inv = new Inv(Server.nodeAddress, kind, blockHashs);
        byte[] payload = SerializeUtils.serialize(inv);
        System.out.println("sendInv:"+payload.length);
        ServerSocketClient.sendData(toAddress, ServerConst.COMMAND_INV, payload);

    }

    //COMMAND_GETDATA
    public static void sendGetData(String toAddress, String kind, String blockHash) {
        GetData getData = new GetData(Server.nodeAddress, kind, blockHash);

        byte[] payload = SerializeUtils.serialize(getData);
        System.out.println("sendGetData:"+payload.length);
        ServerSocketClient.sendData(toAddress, ServerConst.COMMAND_GETDATA, payload);

    }

    public static void sendBlock(String toAddress, Block block) {
        BlockData blockData = new BlockData(Server.nodeAddress, SerializeUtils.serialize(block));

        byte[] payload = SerializeUtils.serialize(blockData);

        ServerSocketClient.sendData(toAddress, ServerConst.COMMAND_BLOCK, payload);

    }

    public static void sendTx(String toAddress, Transaction tx) {
        TransactionData txData = new TransactionData(Server.nodeAddress, SerializeUtils.serialize(tx));

        byte[] payload = SerializeUtils.serialize(txData);

        ServerSocketClient.sendData(toAddress, ServerConst.COMMAND_TX, payload);

    }

} 
```

#### 3.2.14 新建`ServerThread.java`文件

在`cldy.hanru.blockchain.net`包。新建`ServerThread.java`文件。并添加代码如下:

```java
package cldy.hanru.blockchain.net;

import cldy.hanru.blockchain.block.Blockchain;
import cldy.hanru.blockchain.util.ByteUtils;
import cldy.hanru.blockchain.util.CommandUtils;
import lombok.AllArgsConstructor;
import lombok.extern.slf4j.Slf4j;

import java.io.IOException;
import java.io.InputStream;
import java.net.Socket;

@Slf4j
@AllArgsConstructor
/**
 * 处理从其他节点，接收到的消息
 * @author hanru
 */
public class ServerThread implements Runnable {
    private Socket socket;
    private Blockchain bc;

    @Override
    public void run() {
        InputStream inputStream = null;
        try {
            inputStream = socket.getInputStream();
            byte[] commandData = new byte[CommandUtils.MESSAGE_TYPE_DATA_LENGTH];
            // 消息数据长度 byte array
            byte[] dataLen = new byte[4];

            if (inputStream.read(commandData) != CommandUtils.MESSAGE_TYPE_DATA_LENGTH) {
                throw new IOException("EOF in PeerMessage constructor: command");
            }

            if (inputStream.read(dataLen) != 4) {
                throw new IOException("EOF in PeerMessage constructor: dataLen");
            }

            int len = ByteUtils.toInt(dataLen);
            System.out.println("data 的长度是：" + len);
            byte[] data = new byte[len];

            if (inputStream.read(data) != len) {
                throw new IOException("EOF in PeerMessage constructor: Unexpected message data length");
            }

            String command = CommandUtils.bytesToCommand(commandData);
            System.out.printf("Receive a Message:%s\n", command);

            switch (command) {
                case ServerConst.COMMAND_VERSION:
                    ServerHandle.handleVersion(data, bc);
                    break;
                case ServerConst.COMMAND_GETBLOCKS:
                    ServerHandle.handleGetblocks(data, bc);
                    break;
                case ServerConst.COMMAND_INV:
                    ServerHandle.handleInv(data, bc);
                    break;
                case ServerConst.COMMAND_GETDATA:
                    ServerHandle.handleGetData(data, bc);
                    break;
                case ServerConst.COMMAND_BLOCK:
                    ServerHandle.handleBlock(data, bc);
                    break;
                case ServerConst.COMMAND_TX:
                    ServerHandle.handleTx(data, bc);
                    break;
                default:
                    log.info("Unknown command!");
            }

        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            if (inputStream != null) {
                try {
                    inputStream.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
            if (socket != null) {
                try {
                    socket.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }
    }
} 
```

####

#### 3.2.15 新建`ServerSocketClient.go`文件

在`cldy.hanru.blockchain.net`包。新建`ServerSocketClient.java`文件。并添加代码如下:

```java
package cldy.hanru.blockchain.net;

import cldy.hanru.blockchain.util.ByteUtils;
import cldy.hanru.blockchain.util.CommandUtils;

import java.io.IOException;
import java.io.OutputStream;
import java.net.Socket;

public class ServerSocketClient {

    public static void sendData(String toAddress, String command, byte[] payload) {
        /*
         * 第一个参数：IP：InetAddress，String，要连接的服务端的 ip
         *
         * 第二个参数：int port，要连接的服务端的程序的端口。
         */
        Socket client = null;
        OutputStream outputStream = null;
        try {
            //step1:创建客户端对象
            //指定要链接的主节点的地址和端口
            String address = toAddress.substring(0, toAddress.indexOf(":"));
            int port = Integer.parseInt(toAddress.substring(toAddress.indexOf(":") + 1));
            client = new Socket(address, port);
            System.out.println("客户端已经建立，同时向服务端发送连接请求。。" + address + "，" + port);
            //step2：从 Socket 中获取输出流
            outputStream = client.getOutputStream();
            outputStream.write(toData(command, payload));

        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            //step3：关闭资源
            if (outputStream != null) {
                try {
                    outputStream.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
            if (client != null) {//可以省略不写
                try {
                    client.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }
    }

    /**
     * 将要发送的数据进行封装：
     * 格式为：type + data length + data
     *
     * @param payload
     * @return
     */
    public static byte[] toData(String command, byte[] payload) {
        byte[] commandBytes = CommandUtils.commandToBytes(command);
        byte[] dataLenBytes = ByteUtils.toBytes(payload.length);
        System.out.println("写出 payload 长度：" + payload.length);
        byte[] bytes = ByteUtils.merge(commandBytes,dataLenBytes,payload);
        return bytes;
    }

} 
```

#### 3.2.16 新建`ServerHandle.go`文件

在`cldy.hanru.blockchain.net`包。新建`ServerHandle.java`文件。并添加代码如下:

```java
package cldy.hanru.blockchain.net;

import cldy.hanru.blockchain.block.Block;
import cldy.hanru.blockchain.block.Blockchain;
import cldy.hanru.blockchain.store.RocksDBUtils;
import cldy.hanru.blockchain.transaction.Transaction;
import cldy.hanru.blockchain.transaction.UTXOSet;
import cldy.hanru.blockchain.util.ByteUtils;
import cldy.hanru.blockchain.util.SerializeUtils;
import lombok.extern.slf4j.Slf4j;

import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

@Slf4j
/**
 * 处理消息的具体方法
 * @author hanru
 */
public class ServerHandle {

    // 存储 hash 值
    public static List<String> hashList = new ArrayList<>();

    public static Map<String, Transaction> memoryTxPool = new HashMap<>();

    public static void handleVersion(byte[] data, Blockchain bc) {

        // 反序列化
        Version version = (Version) SerializeUtils.deserialize(data);

        long bestHeight = bc.getBestHeight();          //2 1
        long foreignerBestHeight = version.getBestHeight(); // 1 2
        System.out.println("自己的版本：" + bestHeight + ",对方的版本：" + foreignerBestHeight);

        if (bestHeight > foreignerBestHeight) {
            ServerSend.sendVersion(version.getNodeAddress(), bc);
        } else if (bestHeight < foreignerBestHeight) {
            // 去向主节点要信息
            ServerSend.sendGetBlocks(version.getNodeAddress());
        }

        if (!Server.nodeIsKnown(version.getNodeAddress())) {
            Server.knowNodes.add(version.getNodeAddress());
        }
    }

    public static void handleGetblocks(byte[] data, Blockchain bc) {

        GetBlocks getBlocks = null;
        // 反序列化
        try {
             getBlocks = (GetBlocks) SerializeUtils.deserialize(data);
        }catch (ClassCastException e){
            System.out.println("handleGetblocks:"+e);
        }

        System.out.println("handleGetblocks:"+getBlocks);
        //获取当前节点的区块的 hash

        List<String> blockHashs = bc.getBlockHashes();

        //txHash blockHash
        ServerSend.sendInv(getBlocks.getAddrFrom(), ServerConst.BLOCK_TYPE, blockHashs);

    }

    public static void handleInv(byte[] data, Blockchain bc) {
        //反序列化
        Inv inv = (Inv) SerializeUtils.deserialize(data);

        if (ServerConst.BLOCK_TYPE.equals(inv.getType())) {
            String blockHash = inv.getItems().get(0);
            ServerSend.sendGetData(inv.getAddrFrom(), ServerConst.BLOCK_TYPE, blockHash);

            if (inv.getItems().size() >= 1) {
                hashList = inv.getItems().subList(1, inv.getItems().size());
            }

        } else if (ServerConst.TX_TYPE.equals(inv.getType())) {
            String txHash = inv.getItems().get(0);
            if (!memoryTxPool.containsKey(txHash)) {
                ServerSend.sendGetData(inv.getAddrFrom(), ServerConst.TX_TYPE, txHash);
            }
        }
    }

    //
    public static void handleGetData(byte[] data, Blockchain bc) {
        //反序列化
        GetData getData = (GetData) SerializeUtils.deserialize(data);
        //如果是区块
        if (ServerConst.BLOCK_TYPE.equals(getData.getType())) {
            Block block = bc.getBlock(getData.getHash());
            ServerSend.sendBlock(getData.getAddrFrom(), block);

        } else if (ServerConst.TX_TYPE.equals(getData.getType())) {
            //如果是笔交易
            Transaction tx = memoryTxPool.get(getData.getHash());
            ServerSend.sendTx(getData.getAddrFrom(), tx);
        }

    }

    public static void handleBlock(byte[] data, Blockchain bc) {
        //反序列化
        BlockData blockData = (BlockData) SerializeUtils.deserialize(data);

        Block block = (Block) SerializeUtils.deserialize(blockData.getBlockData());

        System.out.println("Recevied a new block!");

        System.out.println("handleBlock:"+bc.getNodeID());
//        bc.addBlock(block);
        bc.saveBlock(block);

        UTXOSet utxoSet = new UTXOSet(bc);
//        utxoSet.reIndex();
        utxoSet.update(block);

        if (hashList.size() > 0) {
            String blockHash = hashList.get(0);
            ServerSend.sendGetData(blockData.getAddrFrom(), ServerConst.BLOCK_TYPE, blockHash);
            hashList = hashList.subList(1, hashList.size());
        }

    }

    //
    public static void handleTx(byte[] data, Blockchain bc) {
        //反序列化
        TransactionData txData = (TransactionData) SerializeUtils.deserialize(data);

        Transaction tx = (Transaction) SerializeUtils.deserialize(txData.getTxData());

        System.out.println("Recevied a new transaction!");

        String txIdStr = ByteUtils.bytesToHexString(tx.getTxId());

        memoryTxPool.put(txIdStr, tx);

        // 说明主节点自己
        if (Server.knowNodes.get(0).equals(Server.nodeAddress)) {
            // 给矿工节点发送交易 hash
            List<String> txList = new ArrayList<>();
            txList.add(ByteUtils.bytesToHexString(tx.getTxId()));
            for (String toAddr : Server.knowNodes) {
                ServerSend.sendInv(toAddr, ServerConst.TX_TYPE, txList);
            }
        }

        // 矿工进行挖矿验证
        if (memoryTxPool.size() >= 1 && !"".equals(Server.minerAddress)) {
            UTXOSet utxoSet = new UTXOSet(bc);

            //验证数字签名。。
            try {
                if (!bc.verifyTransactions(tx)) {
                    log.info("ERROR: Invalid transaction..");
                }
            } catch (Exception e) {
                e.printStackTrace();
            }
            List<Transaction> txs = new ArrayList<>();
            txs.add(tx);
            //奖励
            Transaction coinbaseTx = Transaction.newCoinbaseTX(Server.minerAddress, "");
            txs.add(coinbaseTx);
            //挖掘新区块
            Block lastBlock = RocksDBUtils.getInstance(bc.getNodeID()).getLastBlock();

            Block newBlock = Block.newBlock(lastBlock.getHash(), txs, lastBlock.getHeight() + 1);

            RocksDBUtils.getInstance(bc.getNodeID()).putBlock(newBlock);
            RocksDBUtils.getInstance(bc.getNodeID()).putLastBlockHash(newBlock.getHash());

            utxoSet.update(newBlock);

            ServerSend.sendBlock(Server.knowNodes.get(0), newBlock);

            //从内存池中删除
            memoryTxPool.remove(txIdStr);

        }
    }

} 
```

#### 3.2.17 新建`CommandUtils.go`文件

在`cldy.hanru.blockchain.util`包。新建`CommandUtils.java`文件。并添加代码如下:

```java
package cldy.hanru.blockchain.util;

import org.apache.commons.lang3.ArrayUtils;

public class CommandUtils {

    /**
     * 规定 message type 转化后的 byte[]长度
     */
    public final static int MESSAGE_TYPE_DATA_LENGTH = 12;

    /**
     * 将消息类型字符串转化长度为 12 的字节数组
     * @param command
     * @return
     */
    public static byte[] commandToBytes(String command ){
        byte[] byteArray = new byte[MESSAGE_TYPE_DATA_LENGTH];
        for (int i = 0; i < command.getBytes().length; i++) {
            byteArray[i] = command.getBytes()[i];
        }
        return byteArray;
    }

    /**
     * 消息类型 byte[] 转化为字符串
     *
     * @param bytes
     * @return
     */
    public static String bytesToCommand(byte[] bytes) {
        byte[] typeBytes = ArrayUtils.removeAllOccurences(bytes, (byte) 0);
        return new String(typeBytes);
    }

} 
```

## 4\. 网络讲解

### 4.1 区块链网络

区块链网络是去中心化的，这意味着没有服务器，客户端也不需要依赖服务器来获取或处理数据。在区块链网络中，有的是节点，每个节点是网络的一个完全（full-fledged）成员。节点就是一切：它既是一个客户端，也是一个服务器。这一点需要牢记于心，因为这与传统的网页应用非常不同。

区块链网络是一个 P2P（Peer-to-Peer，端到端）的网络，即节点直接连接到其他节点。它的拓扑是扁平的，因为在节点的世界中没有层级之分。下面是它的示意图：

![`img.kongyixueyuan.com/1108_p2p.png`](img/e596235fd74ec651b91feec11013f591.jpg)

要实现这样一个网络节点更加困难，因为它们必须执行很多操作。每个节点必须与很多其他节点进行交互，它必须请求其他节点的状态，与自己的状态进行比较，当状态过时时进行更新。

### 4.2 节点角色

尽管节点具有完备成熟的属性，但是它们也可以在网络中扮演不同角色。比如：

1.  矿工 这样的节点运行于强大或专用的硬件（比如 ASIC）之上，它们唯一的目标是，尽可能快地挖出新块。矿工是区块链中唯一可能会用到工作量证明的角色，因为挖矿实际上意味着解决 PoW 难题。在权益证明 PoS 的区块链中，没有挖矿。
2.  全节点 这些节点验证矿工挖出来的块的有效性，并对交易进行确认。为此，他们必须拥有区块链的完整拷贝。同时，全节点执行路由操作，帮助其他节点发现彼此。对于网络来说，非常重要的一段就是要有足够多的全节点。因为正是这些节点执行了决策功能：他们决定了一个块或一笔交易的有效性。
3.  SPV SPV 表示 Simplified Payment Verification，简单支付验证。这些节点并不存储整个区块链副本，但是仍然能够对交易进行验证（不过不是验证全部交易，而是一个交易子集，比如，发送到某个指定地址的交易）。一个 SPV 节点依赖一个全节点来获取数据，可能有多个 SPV 节点连接到一个全节点。SPV 使得钱包应用成为可能：一个人不需要下载整个区块链，但是仍能够验证他的交易。

### 4.3 网络简化

为了在目前的区块链原型中实现网络，我们不得不简化一些事情。因为我们没有那么多的计算机来模拟一个多节点的网络。当然，我们可以使用虚拟机或是 Docker 来解决这个问题，但是这会使一切都变得更复杂：你将不得不先解决可能出现的虚拟机或 Docker 问题，而我的目标是将全部精力都放在区块链实现上。所以，我们想要在一台机器上运行多个区块链节点，同时希望它们有不同的地址。为了实现这一点，我们将使用**端口号作为节点标识符**，而不是使用 IP 地址，比如将会有这样地址的节点：**127.0.0.1:3000**，**127.0.0.1:3001**，**127.0.0.1:3002** 等等。我们叫它端口节点（port node） ID，并使用环境变量 `NODE_ID` 对它们进行设置。故而，你可以打开多个终端窗口，设置不同的 `NODE_ID` 运行不同的节点。

这个方法也需要有不同的区块链和钱包文件。它们现在必须依赖于节点 ID 进行命名，比如 `blockchain_3000.db`, `blockchain_30001.db` , `wallet_3000.db`, `wallet_30001.db` 等等。

所以，当你下载 Bitcoin Core 并首次运行时，到底发生了什么呢？它必须连接到某个节点下载最新状态的区块链。考虑到你的电脑并没有意识到所有或是部分的比特币节点，那么连接到的“某个节点”到底是什么？

在 Bitcoin Core 中硬编码一个地址，已经被证实是一个错误：因为节点可能会被攻击或关机，这会导致新的节点无法加入到网络中。在 Bitcoin Core 中，硬编码了 [DNS seeds](https://bitcoin.org/en/glossary/dns-seed)。虽然这些并不是节点，但是 DNS 服务器知道一些节点的地址。当你启动一个全新的 Bitcoin Core 时，它会连接到一个种子节点，获取全节点列表，随后从这些节点中下载区块链。

不过在我们目前的实现中，无法做到完全的去中心化，因为会出现中心化的特点。我们会有三个节点：

1.  一个中心节点。所有其他节点都会连接到这个节点，这个节点会在其他节点之间发送数据。
2.  一个矿工节点。这个节点会在内存池中存储新的交易，当有足够的交易时，它就会打包挖出一个新块。
3.  一个钱包节点。这个节点会被用作在钱包之间发送币。但是与 SPV 节点不同，它存储了区块链的一个完整副本。

### 4.4 场景

本文的目标是实现如下场景：

1.  中心节点创建一个区块链。
2.  一个其他（钱包）节点连接到中心节点并下载区块链。
3.  另一个（矿工）节点连接到中心节点并下载区块链。
4.  钱包节点创建一笔交易。
5.  矿工节点接收交易，并将交易保存到内存池中。
6.  当内存池中有足够的交易时，矿工开始挖一个新块。
7.  当挖出一个新块后，将其发送到中心节点。
8.  钱包节点与中心节点进行同步。
9.  钱包节点的用户检查他们的支付是否成功。

这就是比特币中的一般流程。尽管我们不会实现一个真实的 P2P 网络，但是我们会实现一个真实，也是比特币最常见最重要的用户场景。

##

### 4.5 代码实现

#### 4.5.1 设置`NODE_ID`

1.如何设置`NODE_ID`

我们可以通过`System.getenv()`方法获取 环境变量的值。

在`cldy.hanru.blockchain.cli`包下，修改`CLI.java`文件中的`Run()`方法，代码如下：

```java
/**
     * 命令行解析入口
     */
    public void run() {
        this.validateArgs(args);

        /*
    获取节点 ID
    解释：返回当前进程的环境变量 varname 的值,若变量没有定义时返回 nil
    export NODE_ID=8888

    每次打开一个终端，都需要设置 NODE_ID 的值。
    变量名 NODE_ID，可以更改别的。
     */

        Map<String, String> map = System.getenv();

        String nodeID = map.get("NODE_ID");
        if (nodeID == "") {
            System.out.println("NODE_ID 环境变量没有设置。。");
            System.exit(0);
        }

        System.out.println("NODE_ID：" + nodeID);

        CommandLineParser parser = new DefaultParser();
        CommandLine cmd = null;

    ...

} 
```

接下来我们测试一下`NODE_ID`，首先打开一个终端，输入以下命令：

```java
hanru:part10_Network ruby$ export NODE_ID=9527
hanru:part10_Network ruby$ ./blockchain.sh h 
```

运行结果如下：

![`img.kongyixueyuan.com/012_002_nodeid.png`](img/46bc70d65c75648fa7673863e2670024.jpg)

#### 4.5.2 更新项目中结合`NODE_ID`

现在我们已经在程序中设置了`NODE_ID`，接下来我们需要调整程序，使用`NODE_ID`模拟不同的节点，分别创建自己的数据库文件和钱包文件等。我们需要一点一点修改：

首先修改数据库的名字：

打开 BLC 包下，`RocksDBUtils.java`文件，修改如下：

```java
public class RocksDBUtils {
    /**
     * 区块链数据文件
     */
    private static  String DB_FILE = "blockchain_%s.db";
    ...
} 
```

我们可以利用`java.lang`包下的`public static String format(String format, Object... args)`方法，来动态设置数据库文件名称和钱包文件名称。

**1\. 接下来修改创建创世区块的功能：**

step1：修改`CLI.java`中的方法，修改`private void createBlockchain(String address, String nodeID)`，添加`nodeID`，代码如下：

```java
/**
     * 创建区块链
     *
     * @param address
     */
    private void createBlockchain(String address, String nodeID) {

        Blockchain blockchain = Blockchain.createBlockchain(address,nodeID);
        UTXOSet utxoSet = new UTXOSet(blockchain);
        utxoSet.reIndex();
        log.info("Done ! ");
    } 
```

step2：修改`Blockchain.java`文件中，首先添加`nodeID`字段，代码如下：

```java
@Data
@AllArgsConstructor
public class Blockchain {

    /**
     * 最后一个区块的 hash
     */
    private String lastBlockHash;

    /**
     * 当前节点的 nodeID
     */
    private String nodeID;
    ...
    } 
```

step3：接下来修改`createBlockchain()`方法声明，添加`nodeID`，代码如下：

```java
 /**
     * 创建区块链，createBlockchain
     *
     * @param address
     * @return
     */
    public static Blockchain createBlockchain(String address, String nodeID) {

        String lastBlockHash = RocksDBUtils.getInstance(nodeID).getLastBlockHash();
        if (StringUtils.isBlank(lastBlockHash)) {
            //对应的 bucket 不存在，说明是第一次获取区块链实例
            // 创建 coinBase 交易
            Transaction coinbaseTX = Transaction.newCoinbaseTX(address, "");
            Block genesisBlock = Block.newGenesisBlock(coinbaseTX);
//            Block genesisBlock = Block.newGenesisBlock();
            lastBlockHash = genesisBlock.getHash();
            RocksDBUtils.getInstance(nodeID).putBlock(genesisBlock);
            RocksDBUtils.getInstance(nodeID).putLastBlockHash(lastBlockHash);

        }
        return new Blockchain(lastBlockHash,nodeID);
    } 
```

step4：继续修改`Blockchain.java`文件，修改`initBlockchainFromDB(String nodeID)`方法，用于初始化 Blockchain 对象，修改后代码如下：

```java
/**
     * 从 DB 从恢复区块链数据
     *
     * @return
     * @throws Exception
     */
    public static Blockchain initBlockchainFromDB(String nodeID) throws Exception {
        String lastBlockHash = RocksDBUtils.getInstance(nodeID).getLastBlockHash();
        if (lastBlockHash == null) {
            throw new Exception("ERROR: Fail to init blockchain from db. ");
        }
        return new Blockchain(lastBlockHash,nodeID);
    } 
```

step5：接下来修改`RocksDBUtils.java`文件，修改`getInstance(nodeID)`方法，添加`nodeID`，修改后代码如下：

```java
/**
     * 获取 RocksDBUtils 的单例
     * @return
     */
    public static RocksDBUtils getInstance(String nodeID) {
        if (instance == null) {
            synchronized (RocksDBUtils.class) {
                if (instance == null) {
                    instance = new RocksDBUtils(nodeID);
                }
            }
        }
        return instance;
    } 
```

step6：继续修改`RocksDBUtils.java`文件，修改构造函数，添加`nodeID`，修改后代码如下：

```java
private RocksDBUtils(String nodeID) {
        openDB(nodeID);
        initBlockBucket();
        initChainStateBucket();
    } 
```

step7：继续修改`RocksDBUtils.java`文件，修改`openDB()`方法，添加`nodeID`，修改后代码如下：

```java
/**
     * 打开数据库
     */
    private void openDB(String nodeID) {
        DB_FILE = String.format(DB_FILE,nodeID);
        try {
            db = RocksDB.open(DB_FILE);
        } catch (RocksDBException e) {
            throw new RuntimeException("打开数据库失败。。 ! ", e);
        }
    } 
```

**2\. 修改查询余额功能：**

step1：修改`CLI.java`中的方法，修改`private void getBalance(String address, String nodeID)`，添加`nodeID`，代码如下：

```java
/**
     * 查询钱包余额
     *
     * @param address 钱包地址
     */
    private void getBalance(String address, String nodeID) throws Exception {

        // 检查钱包地址是否合法
        try {
            Base58Check.base58ToBytes(address);
        } catch (Exception e) {
            throw new Exception("ERROR: invalid wallet address");
        }
        Blockchain blockchain = Blockchain.createBlockchain(address,nodeID);
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

**3\. 修改打印区块功能：**

step1：修改`CLI.java`中的方法，修改`private void printChain( String nodeID)`，添加`nodeID`，代码如下：

```java
/**
     * 打印出区块链中的所有区块
     */
    private void printChain( String nodeID) {
//            Blockchain blockchain = Blockchain.newBlockchain();
        Blockchain blockchain = null;
        try {
            blockchain = Blockchain.initBlockchainFromDB(nodeID);
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
```

**4\. 修改创建钱包功能：**

step1：修改`CLI.java`中的方法，修改`private void createWallet(String nodeID)`，添加`nodeID`，代码如下：

```java
/**
     * 创建钱包
     *
     * @throws Exception
     */
    private void createWallet(String nodeID) throws Exception {
        Wallet wallet = WalletUtils.getInstance(nodeID).createWallet();
        System.out.println("wallet address : " + wallet.getAddress());
    } 
```

step2：修改`WalletUtils.java`文件中，

先修改`walletFile`命名：

```java
public class WalletUtils {

    /**
     * 钱包文件
     */
    private  static String WALLET_FILE = "wallet_%s.dat";
    ...
} 
```

然后修改`getInstance()`方法声明，添加`nodeID`，代码如下：

```java
 public static WalletUtils getInstance(String nodeID) {
        if (instance == null) {
            synchronized (WalletUtils.class) {
                if (instance == null) {
                    instance = new WalletUtils(nodeID);
                }
            }
        }
        return instance;
    } 
```

接下来，修改构造函数，添加`nodeID`，代码如下:

```java
private WalletUtils(String nodeID) {
        initWalletFile(nodeID);
    } 
```

再然后，修改`initWalletFile()`方法声明，添加`nodeID`，代码如下:

```java
/**
     * 初始化钱包文件
     */
    private void initWalletFile(String nodeID) {
        WALLET_FILE = String.format(WALLET_FILE, nodeID);
        File file = new File(WALLET_FILE);
        if (!file.exists()) {
            this.saveToDisk(new Wallets());
        } else {
            this.loadFromDisk();
        }
    } 
```

**6\. 修改打印钱包地址功能：**

step1：修改`CLI.java`中的方法，修改`private void printAddresses(String nodeID)`，添加`nodeID`，代码如下：

```java
 /**
     * 打印钱包地址
     *
     * @throws Exception
     */
    private void printAddresses(String nodeID) throws Exception {
        Set<String> addresses = WalletUtils.getInstance(nodeID).getAddresses();
        if (addresses == null || addresses.isEmpty()) {
            System.out.println("There isn't address");
            return;
        }
        for (String address : addresses) {
            System.out.println("Wallet address: " + address);
        }
    } 
```

**7\. 修改转账交易功能：**

step1：修改`CLI.java`中的方法，修改`private void send(String from, String to, int amount,String nodeID, boolean mineNow)`，添加`nodeID`，代码如下：

```java
/**
     * 转账
     *
     * @param from
     * @param to
     * @param amount
     */
    private void send(String from, String to, int amount,String nodeID, boolean mineNow) throws Exception {
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

        Blockchain blockchain = Blockchain.createBlockchain(from,nodeID);
        // 新交易
        Transaction transaction = Transaction.newUTXOTransaction(from, to, amount, blockchain, nodeID);
        if(mineNow){
            // 奖励
            Transaction rewardTx = Transaction.newCoinbaseTX(from, "");
            List<Transaction> transactions = new ArrayList<>();
            transactions.add(transaction);
            transactions.add(rewardTx);
            Block newBlock = blockchain.mineBlock(transactions);
            new UTXOSet(blockchain).update(newBlock);
            log.info("Success!");
        }else{
            //矿工节点处理。。
            System.out.println("由矿工节点处理。。。。");
            ServerSend.sendTx(Server.knowNodes.get(0), transaction);
        }

    } 
```

step2：修改`cldy.hanru.blockchain.transaction`包下`Transaction.java`文件中，修改`newUTXOTransaction()`方法声明，添加`nodeID`，代码如下：

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
    public static Transaction newUTXOTransaction(String from, String to, int amount, Blockchain blockchain,String nodeID) throws Exception {

        // 获取钱包
        Wallet senderWallet = WalletUtils.getInstance(nodeID).getWallet(from);
        byte[] pubKey = senderWallet.getPublicKey();
        byte[] pubKeyHash = AddressUtils.ripeMD160Hash(pubKey);

...
    } 
```

step3：最后，修改`UTXOSet.java`文件中，一系列的方法。

现在，我们进行代码测试，看一下是否可以不同的`NODE_ID`，可以创建出不同的数据库和钱包文件，打开一个终端并输入以下命令：

```java
hanru:part10_Network ruby$ export NODE_ID=3000
hanru:part10_Network ruby$ ./blockchain.sh h
hanru:part10_Network ruby$ ./blockchain.sh createwallet 
```

运行结果如下：

![`img.kongyixueyuan.com/012_003_%E8%AE%BE%E7%BD%AE%E8%8A%82%E7%82%B9%E8%BF%90%E8%A1%8C%E7%BB%93%E6%9E%9C.png`](img/14659cd703ae668da2e4d80e104202cb.jpg)

接下来我们继续输入以下命令：

```java
hanru:day08_10_Net ruby$ ./bc createwallet
hanru:day08_10_Net ruby$ ./bc addresslists
hanru:day08_10_Net ruby$ ./bc createblockchain -address '16s8TuPn9PeoSGLMB395nsZ7taEAFjuu3L'
hanru:day08_10_Net ruby$ ./bc getbalance -address '16s8TuPn9PeoSGLMB395nsZ7taEAFjuu3L' 
```

运行效果如下：

![`img.kongyixueyuan.com/012_004_%E8%BF%90%E8%A1%8C%E7%BB%93%E6%9E%9C.png`](img/bc0aecdfef39b09d7298162c2b9806f9.jpg)

接下来我们实现转账以及查询余额，输入终端命令如下:

```java
hanru:part10_Network ruby$ ./blockchain.sh send -from 1G7shLHhma9Hi13itephS1H3g1m4ZTVoYJ -to 1Db14aTx34HmFM8qZkxJPxtXhs56vD7q2T -amount 4 -mine true
hanru:part10_Network ruby$ ./blockchain.sh getbalance -address 1G7shLHhma9Hi13itephS1H3g1m4ZTVoYJ
hanru:part10_Network ruby$ ./blockchain.sh getbalance -address 1Db14aTx34HmFM8qZkxJPxtXhs56vD7q2T 
```

运行效果如下：

![`img.kongyixueyuan.com/012_005_%E8%BF%90%E8%A1%8C%E7%BB%93%E6%9E%9C.png`](img/0a311b7fd71d2516b45be85983ff016b.jpg)

最后，我们再尝试以下其他的`NODE_ID`，再打开另一个终端，输入以下命令：

```java
hanru:part10_Network ruby$ export NODE_ID=3001
hanru:part10_Network ruby$ ./blockchain.sh h
hanru:part10_Network ruby$ ./blockchain.sh createwallet
hanru:part10_Network ruby$ ./blockchain.sh createblockchain -address 19o2kHP63Sn3pSCYHWajTrhRq4gKjw2GX5 
```

运行结果：

![`img.kongyixueyuan.com/012_006_%E5%8F%A6%E4%B8%80%E4%B8%AA%E8%8A%82%E7%82%B9.png`](img/cc477889e3116d3f89fc982bd0cc6ad3.jpg)

至此，我们已经将项目中结合了`NODE_ID`，可以模拟不同的节点工作了。

#### 4.5.3 版本`Version`

节点通过消息（message）进行交流。当一个新的节点开始运行时，它会从一个 DNS 种子获取几个节点，给它们发送 `version` 消息，在我们的实现看起来就像是这样：

```java
package cldy.hanru.blockchain.net;

import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

/**
 * 区块链版本
 * @author hanru
 */
@AllArgsConstructor
@Data
@NoArgsConstructor
public class Version {
    /**
     * 版本
     */
    public  int version ;
    /**
     * 当前节点区块的高度
     */
    private long bestHeight;
    /**
     * 当前节点的地址
     */
    private String nodeAddress;
} 
```

由于我们仅有一个区块链版本，所以 `Version` 字段实际并不会存储什么重要信息。`BestHeight` 存储区块链中节点的高度。`AddFrom` 存储发送者的地址。

接收到 `version` 消息的节点应该做什么呢？它会响应自己的 `version` 消息。这是一种握手：如果没有事先互相问候，就不可能有其他交流。不过，这并不是出于礼貌：`version` 用于找到一个更长的区块链。当一个节点接收到 `version` 消息，它会检查本节点的区块链是否比 `BestHeight` 的值更大。如果不是，节点就会请求并下载缺失的块。

为了接收消息，我们需要一个服务器：

首先修改`CLI.java`文件中的`run()`方法，添加启动节点命令，修改后代码如下:

```java
 /**
     * 命令行解析入口
     */
    public void run() {
        this.validateArgs(args);

        /*
    获取节点 ID
    解释：返回当前进程的环境变量 varname 的值,若变量没有定义时返回 nil
    export NODE_ID=8888

    每次打开一个终端，都需要设置 NODE_ID 的值。
    变量名 NODE_ID，可以更改别的。
     */

        Map<String, String> map = System.getenv();

        String nodeID = map.get("NODE_ID");
        if (nodeID == "") {
            System.out.println("NODE_ID 环境变量没有设置。。");
            System.exit(0);
        }

        System.out.println("NODE_ID：" + nodeID);

        CommandLineParser parser = new DefaultParser();
        CommandLine cmd = null;
        try {
            cmd = parser.parse(options, args);
        } catch (ParseException e) {
            e.printStackTrace();
        }

        switch (args[0]) {
            ...
            case "startnode":
                String minerAddress = cmd.getOptionValue("address");
                try {
                    start(nodeID, minerAddress);
                } catch (Exception e) {
                    e.printStackTrace();
                }
                break;
            default:
                this.help();
        }

    } 
```

并添加启动节点的方法：`start(String nodeID, String minerAddress)`，代码如下:

```java
/**
     * 启动节点
     * @param nodeID
     * @param minerAddress
     * @throws Exception
     */
    private void start(String nodeID, String minerAddress) throws Exception {

        System.out.println("minerAddress:" + minerAddress);
        if (null == minerAddress || minerAddress == "") {

        } else {
            // 检查钱包地址是否合法
            try {
                Base58Check.base58ToBytes(minerAddress);
            } catch (Exception e) {
                throw new Exception("ERROR: receiver address invalid ! address=" + minerAddress);
            }
        }

        System.out.printf("启动服务器：localhost:%s\n", nodeID);
        Server.startServer(nodeID, minerAddress);

    } 
```

先新建`cldy.hanru.blockchain.net`包，并`ServerConst.java`文件，存储节点常量，代码如下：

```java
package cldy.hanru.blockchain.net;

/**
 * 节点服务的常量
 * @author hanru
 */
public class ServerConst {

    public static int NODE_VERSION = 1;

    // 命令
    public static final String COMMAND_VERSION = "version";
    public static final String COMMAND_GETBLOCKS = "getblocks";
    public static final String COMMAND_ADDR = "addr";
    public static final String COMMAND_BLOCK = "block";
    public static final String COMMAND_INV = "inv";
    public static final String COMMAND_GETDATA = "getdata";
    public static final String COMMAND_TX = "tx";

    // 类型
    public static final String BLOCK_TYPE = "block";
    public static final String TX_TYPE = "tx";

} 
```

再新建`Server.java`文件，用于表示节点的服务端，添加`startServer()`方法，代码如下：

```java
public static void startServer(String nodeID, String minerAdd) {
//        knowNodes.add("localhost:3000");//localhost:3000 主节点的地址
        System.out.println("knowNodes(0):"+knowNodes.get(0));
        // 当前节点的 IP 地址
        nodeAddress = String.format("localhost:%s", nodeID);
        minerAddress = minerAdd;
        System.out.printf("nodeAddress:%s,minerAddress:%s\n", nodeAddress, minerAddress);
        ServerSocket server = null;
        Socket socket = null;
        try {
            server = new ServerSocket(Integer.parseInt(nodeID));
            System.out.println("服务端已经建立，等待客户端的连接请求。。。");

            //获取 blockchain
            Blockchain bc = Blockchain.initBlockchainFromDB(nodeID);
            System.out.println("blockchain-->"+bc.getNodeID());

            // 第一个终端：端口为 3000,启动的就是主节点
            // 第二个终端：端口为 3001，钱包节点
            // 第三个终端：端口号为 3002，矿工节点
            if (!knowNodes.get(0).equals(nodeAddress)) {
                // 此节点是钱包节点或者矿工节点，需要向主节点发送请求同步数据
//                System.out.printf("knowNodes:%s\n", knowNodes.get(0));
                ServerSend.sendVersion(knowNodes.get(0), bc);
            }

            while (true) {
                // 收到的数据的格式是固定的，12 字节+长度+ 结构体字节数组
                socket = server.accept();//该方法是阻塞式
                System.out.println("已有客户端连入。。客户端的 ip：" + socket.getInetAddress() + "客户端的 port：" + socket.getPort());
                //业务处理：放在子线程中
                new Thread(new ServerThread(socket, bc)).start();

            }

        } catch (IOException e) {
            e.printStackTrace();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {

            if (socket != null) {//可以省略不写
                try {
                    socket.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }
    } 
```

首先，我们对中心节点的地址进行硬编码：因为每个节点必须知道从何处开始初始化。`minerAddress` 参数指定了接收挖矿奖励的地址。代码片段：

```java
 if (!knowNodes.get(0).equals(nodeAddress)) {
                // 此节点是钱包节点或者矿工节点，需要向主节点发送请求同步数据
//                System.out.printf("knowNodes:%s\n", knowNodes.get(0));
                ServerSend.sendVersion(knowNodes.get(0), bc);
            } 
```

这意味着如果当前节点不是中心节点，它必须向中心节点发送 `version` 消息来查询是否自己的区块链已过时。所以新建`ServerSend.java`文件，用于表示发送消息，并添加方法：

```java
 //发送版本
    public static void sendVersion(String toAddress, Blockchain bc) {
        long bestHeight = bc.getBestHeight();
        Version version = new Version(ServerConst.NODE_VERSION, bestHeight, Server.nodeAddress);
        byte[] payload = SerializeUtils.serialize(version);
        System.out.println("sendVersion:"+payload.length);
        //version
        ServerSocketClient.sendData(toAddress, ServerConst.COMMAND_VERSION, payload);
    } 
```

修改`Blockchain.java`文件，添加方法，用于获取最新区块的高度，代码如下：

```java
//----------
//获取最新区块的高度
    public long getBestHeight() {

        Block block = this.getBlockchainIterator().next();

        return block.getHeight();
    } 
```

我们的消息，在底层就是字节序列。前 12 个字节指定了命令名（比如这里的 `version`），后面的字节会包含 **gob** 编码的消息结构。所以我们可以添加一个工具方法。

在`cldy.hanru.blockchain.util`包下，新建`CommandUtils.java`文件，添加以下工具方法：

```java
/**
     * 将消息类型字符串转化长度为 12 的字节数组
     * @param command
     * @return
     */
    public static byte[] commandToBytes(String command ){
        byte[] byteArray = new byte[MESSAGE_TYPE_DATA_LENGTH];
        for (int i = 0; i < command.getBytes().length; i++) {
            byteArray[i] = command.getBytes()[i];
        }
        return byteArray;
    } 
```

它创建一个 12 字节的缓冲区，并用命令名进行填充，将剩下的字节置为空。

接下来再添加一个方法，用于将字节数据转为命令：

```java
/**
     * 消息类型 byte[] 转化为字符串
     *
     * @param bytes
     * @return
     */
    public static String bytesToCommand(byte[] bytes) {
        byte[] typeBytes = ArrayUtils.removeAllOccurences(bytes, (byte) 0);
        return new String(typeBytes);
    } 
```

当一个节点接收到一个命令，它会运行 `bytesToCommand` 来提取命令名，并选择正确的处理器处理命令主体。

所以接下来在 net 包下，新建一个`ServerThread.java`文件，用于循环处理接收到的消息，代码如下：

```java
 @Override
    public void run() {
        InputStream inputStream = null;
        try {
            inputStream = socket.getInputStream();
            byte[] commandData = new byte[CommandUtils.MESSAGE_TYPE_DATA_LENGTH];
            // 消息数据长度 byte array
            byte[] dataLen = new byte[4];

            if (inputStream.read(commandData) != CommandUtils.MESSAGE_TYPE_DATA_LENGTH) {
                throw new IOException("EOF in PeerMessage constructor: command");
            }

            if (inputStream.read(dataLen) != 4) {
                throw new IOException("EOF in PeerMessage constructor: dataLen");
            }

            int len = ByteUtils.toInt(dataLen);
            System.out.println("data 的长度是：" + len);
            byte[] data = new byte[len];

            if (inputStream.read(data) != len) {
                throw new IOException("EOF in PeerMessage constructor: Unexpected message data length");
            }

            String command = CommandUtils.bytesToCommand(commandData);
            System.out.printf("Receive a Message:%s\n", command);

            switch (command) {
                case ServerConst.COMMAND_VERSION:
                    ServerHandle.handleVersion(data, bc);
                    break;
                case ServerConst.COMMAND_GETBLOCKS:
                    ServerHandle.handleGetblocks(data, bc);
                    break;
                case ServerConst.COMMAND_INV:
                    ServerHandle.handleInv(data, bc);
                    break;
                case ServerConst.COMMAND_GETDATA:
                    ServerHandle.handleGetData(data, bc);
                    break;
                case ServerConst.COMMAND_BLOCK:
                    ServerHandle.handleBlock(data, bc);
                    break;
                case ServerConst.COMMAND_TX:
                    ServerHandle.handleTx(data, bc);
                    break;
                default:
                    log.info("Unknown command!");
            }

        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            if (inputStream != null) {
                try {
                    inputStream.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
            if (socket != null) {
                try {
                    socket.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        } 
```

接下来，新建一个`ServerHandle.java`文件，用于处理接收到的命令。

添加 `version` 命令处理方法，代码如下：

```java
 public static void handleVersion(byte[] data, Blockchain bc) {

        // 反序列化
        Version version = (Version) SerializeUtils.deserialize(data);

        long bestHeight = bc.getBestHeight();          //2 1
        long foreignerBestHeight = version.getBestHeight(); // 1 2
        System.out.println("自己的版本：" + bestHeight + ",对方的版本：" + foreignerBestHeight);

        if (bestHeight > foreignerBestHeight) {
            ServerSend.sendVersion(version.getNodeAddress(), bc);
        } else if (bestHeight < foreignerBestHeight) {
            // 去向主节点要信息
            ServerSend.sendGetBlocks(version.getNodeAddress());
        }

        if (!Server.nodeIsKnown(version.getNodeAddress())) {
            Server.knowNodes.add(version.getNodeAddress());
        }
    } 
```

首先，我们需要对请求进行解码，提取有效信息。所有的处理器在这部分都类似，所以我们会下面的代码片段中略去这部分。

然后节点将从消息中提取的 `BestHeight` 与自身进行比较。如果自身节点的区块链更长，它会回复 `version` 消息；否则，它会发送 `getblocks` 消息。

#### 4.5.4 getblocks

`getblocks` 意为 “给我看一下你有什么区块”（在比特币中，这会更加复杂）。注意，它并没有说“把你全部的区块给我”，而是请求了一个块哈希的列表。这是为了减轻网络负载，因为区块可以从不同的节点下载，并且我们不想从一个单一节点下载数十 GB 的数据。

新建`GetBlocks.java`文件，添加代码如下：

```java
/**
 * @author hanru
 */
public class GetBlocks {
    //getblocks 意为 “给我看一下你有什么区块”（在比特币中，这会更加复杂）
    private String     addrFrom ;
} 
```

处理命令十分简单，在`ServerHandle.java`文件中，添加`handleGetblocks()`方法，代码如下：

```java
 public static void handleGetblocks(byte[] data, Blockchain bc) {

        GetBlocks getBlocks = null;
        // 反序列化
        try {
             getBlocks = (GetBlocks) SerializeUtils.deserialize(data);
        }catch (ClassCastException e){
            System.out.println("handleGetblocks:"+e);
        }

        System.out.println("handleGetblocks:"+getBlocks);
        //获取当前节点的区块的 hash

        List<String> blockHashs = bc.getBlockHashes();

        //txHash blockHash
        ServerSend.sendInv(getBlocks.getAddrFrom(), ServerConst.BLOCK_TYPE, blockHashs);

    } 
```

在我们简化版的实现中，它会返回 **所有块哈希**。

修改`Blockchain.java`文件，添加`getBlockHashes()`方法，用于获取所有区块的`hash`。代码如下：

```java
 //获取所有区块的 hash
    public List<String> getBlockHashes() {

        BlockchainIterator it = this.getBlockchainIterator();

        List<String> blockHashs = new ArrayList<>();

        while (it.hashNext()) {
            Block block = it.next();
            blockHashs.add(0, block.getHash());
        }

        return blockHashs;
    } 
```

#### 4.5.5 `inv`

比特币使用 `inv` 来向其他节点展示当前节点有什么块和交易。再次提醒，它没有包含完整的区块链和交易，仅仅是哈希而已。`Type` 字段表明了这是块还是交易。

新建`Inv.java`文件，添加代码如下：

```java
/**
 * @author hanru
 */
public class Inv {

    String addrFrom; //自己的地址
    String type; //类型 block tx
    List<String> items; //hash 的列表
} 
```

处理 `inv` 稍显复杂，在`ServerHandle.java`文件中，添加`handleInv()`方法，代码如下：

```java
 public static void handleInv(byte[] data, Blockchain bc) {
        //反序列化
        Inv inv = (Inv) SerializeUtils.deserialize(data);

        if (ServerConst.BLOCK_TYPE.equals(inv.getType())) {
            String blockHash = inv.getItems().get(0);
            ServerSend.sendGetData(inv.getAddrFrom(), ServerConst.BLOCK_TYPE, blockHash);

            if (inv.getItems().size() >= 1) {
                hashList = inv.getItems().subList(1, inv.getItems().size());
            }

        } else if (ServerConst.TX_TYPE.equals(inv.getType())) {
            String txHash = inv.getItems().get(0);
            if (!memoryTxPool.containsKey(txHash)) {
                ServerSend.sendGetData(inv.getAddrFrom(), ServerConst.TX_TYPE, txHash);
            }
        }
    } 
```

如果收到块哈希，我们想要将它们保存在 `hashList` 变量来跟踪已下载的块。这能够让我们从不同的节点下载块。在将块置于传送状态时，我们给 `inv` 消息的发送者发送 `getdata` 命令并更新 `hashList`。在一个真实的 P2P 网络中，我们会想要从不同节点来传送块。

在我们的实现中，我们永远也不会发送有多重哈希的 `inv`。这就是为什么当 `payload.Type == "tx"` 时，只会拿到第一个哈希。然后我们检查是否在内存池中已经有了这个哈希，如果没有，发送 `getdata` 消息。

#### 4.5.6 getdata

`getdata` 用于某个块或交易的请求，它可以仅包含一个块或交易的 ID。

新建`GetData.java`文件，添加结构体：

```java
/**
 * author hanru
 */
public class GetData {

    //用于某个块或交易的请求，它可以仅包含一个块或交易的 ID。
    String addrFrom;
    String type;
    String hash;

} 
```

在`ServerHandle.java`文件中，添加`handleGetData()`方法，代码如下：

```java
 public static void handleGetData(byte[] data, Blockchain bc) {
        //反序列化
        GetData getData = (GetData) SerializeUtils.deserialize(data);
        //如果是区块
        if (ServerConst.BLOCK_TYPE.equals(getData.getType())) {
            Block block = bc.getBlock(getData.getHash());
            ServerSend.sendBlock(getData.getAddrFrom(), block);

        } else if (ServerConst.TX_TYPE.equals(getData.getType())) {
            //如果是笔交易
            Transaction tx = memoryTxPool.get(getData.getHash());
            ServerSend.sendTx(getData.getAddrFrom(), tx);
        }

    } 
```

在`Blockchain.java`文件中，添加`getBlock()`方法，用于根据指定的`hash`获取对应的`block`数据，代码如下：

```java
 //根据 hash 获取区块
    public Block getBlock(String blockHash) {
        Block block = RocksDBUtils.getInstance(this.nodeID).getBlock(blockHash);
        return block;

    } 
```

这个处理也比较地直观：如果它们请求一个块，则返回块；如果它们请求一笔交易，则返回交易。注意，我们并不检查实际上是否已经有了这个块或交易。(这是一个缺陷)。

#### 4.5.7 `block` 和 `tx`

实际完成数据转移的正是这些消息：区块和交易。

新建`BlockData.java`文件，添加代码：

```java
/**
 * @author hanru
 */
public class BlockData {
    String addrFrom;
    byte[] blockData;
} 
```

再新建`TransactionData.java`文件，添加代码：

```java
/**
 * @author hanru
 */
public class TransactionData {

    String addrFrom;
    byte[] txData;

} 
```

处理 `block` 消息十分简单，在`ServerHandle.java`文件中，添加`handleBlock()`方法，代码如下：

```java
 public static void handleBlock(byte[] data, Blockchain bc) {
        //反序列化
        BlockData blockData = (BlockData) SerializeUtils.deserialize(data);

        Block block = (Block) SerializeUtils.deserialize(blockData.getBlockData());

        System.out.println("Recevied a new block!");

        System.out.println("handleBlock:"+bc.getNodeID());
//        bc.addBlock(block);
        bc.saveBlock(block);

        UTXOSet utxoSet = new UTXOSet(bc);
//        utxoSet.reIndex();
        utxoSet.update(block);

        if (hashList.size() > 0) {
            String blockHash = hashList.get(0);
            ServerSend.sendGetData(blockData.getAddrFrom(), ServerConst.BLOCK_TYPE, blockHash);
            hashList = hashList.subList(1, hashList.size());
        }

    } 
```

在`Blockchain.java`文件中，添加`saveBlock()`方法，将获取到的区块添加到数据库中。代码如下：

```java
 //节点传来的区块，保存到数据库中
    public void saveBlock(Block newBlock) {
        BlockchainIterator it = this.getBlockchainIterator();
        while (it.hashNext()) {
            Block block = it.next();
            if (block.getHash().equals(newBlock.getHash())) {
                System.out.println("区块已经存在，无需存储。。。");
                return;
            }
        }

        //存储
        RocksDBUtils.getInstance(nodeID).putBlock(newBlock);

        Block lastBlock = RocksDBUtils.getInstance(nodeID).getLastBlock();
        if (lastBlock.getHeight() < newBlock.getHeight()) {
            RocksDBUtils.getInstance(nodeID).putLastBlockHash(newBlock.getHash());
            this.lastBlockHash = newBlock.getHash();
        }
        System.out.printf("Added block %s\n", newBlock.getHash());
    } 
```

当接收到一个新块时，我们把它放到区块链里面。如果还有更多的区块需要下载，我们继续从上一个下载的块的那个节点继续请求。

处理 `tx` 消息是最困难的部分，我们一步一步来实现：

首先在`CLI.go 中`修改转账命令：

```java
//System.out.println("  send -from FROM -to TO -amount AMOUNT -mine MINENOW - Send AMOUNT of coins from FROM address to TO");

Option mine = Option.builder("mine").hasArg(true).desc("mine a block").build();
options.addOption(mine); 
```

修改`CLI.java`文件，修改转账方法:

```java
/**
     * 转账
     *
     * @param from
     * @param to
     * @param amount
     */
    private void send(String from, String to, int amount,String nodeID, boolean mineNow) throws Exception {
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

        Blockchain blockchain = Blockchain.createBlockchain(from,nodeID);
        // 新交易
        Transaction transaction = Transaction.newUTXOTransaction(from, to, amount, blockchain, nodeID);
        if(mineNow){
            // 奖励
            Transaction rewardTx = Transaction.newCoinbaseTX(from, "");
            List<Transaction> transactions = new ArrayList<>();
            transactions.add(transaction);
            transactions.add(rewardTx);
            Block newBlock = blockchain.mineBlock(transactions);
            new UTXOSet(blockchain).update(newBlock);
            log.info("Success!");
        }else{
            //矿工节点处理。。
            System.out.println("由矿工节点处理。。。。");
            ServerSend.sendTx(Server.knowNodes.get(0), transaction);
        }

    } 
```

如果转账时没有直接挖矿创建区块，可以转交给矿工节点进行挖矿，那么就需要将转账交易发送给矿工节点，接下来我们实现转账进入交易消息的处理部分，在`ServerHandle.java`文件中，添加`handleTx()`方法，代码如下：

```java
 public static void handleTx(byte[] data, Blockchain bc) {
        //反序列化
        TransactionData txData = (TransactionData) SerializeUtils.deserialize(data);

        Transaction tx = (Transaction) SerializeUtils.deserialize(txData.getTxData());

        System.out.println("Recevied a new transaction!");

        String txIdStr = ByteUtils.bytesToHexString(tx.getTxId());

        memoryTxPool.put(txIdStr, tx);

        // 说明主节点自己
        if (Server.knowNodes.get(0).equals(Server.nodeAddress)) {
            // 给矿工节点发送交易 hash
            List<String> txList = new ArrayList<>();
            txList.add(ByteUtils.bytesToHexString(tx.getTxId()));
            for (String toAddr : Server.knowNodes) {
                ServerSend.sendInv(toAddr, ServerConst.TX_TYPE, txList);
            }
        }

        // 矿工进行挖矿验证
        if (memoryTxPool.size() >= 1 && !"".equals(Server.minerAddress)) {
            UTXOSet utxoSet = new UTXOSet(bc);
            List<Transaction> txs = new ArrayList<>();
            txs.add(tx);
            //奖励
            Transaction coinbaseTx = Transaction.newCoinbaseTX(Server.minerAddress, "");
            txs.add(coinbaseTx);
            //验证数字签名。。
            try {
                if (!bc.verifyTransactions(tx)) {
                    log.info("ERROR: Invalid transaction..");
                }
            } catch (Exception e) {
                e.printStackTrace();
            }

            //挖掘新区块
            Block lastBlock = RocksDBUtils.getInstance(bc.getNodeID()).getLastBlock();

            Block newBlock = Block.newBlock(lastBlock.getHash(), txs, lastBlock.getHeight() + 1);

            RocksDBUtils.getInstance(bc.getNodeID()).putBlock(newBlock);
            RocksDBUtils.getInstance(bc.getNodeID()).putLastBlockHash(newBlock.getHash());

            utxoSet.update(newBlock);

            ServerSend.sendBlock(Server.knowNodes.get(0), newBlock);

            //从内存池中删除
            memoryTxPool.remove(txIdStr);

        }
    } 
```

首先要做的事情是将新交易放到内存池中（再次提醒，在将交易放到内存池之前，必要对其进行验证）。

```java
memoryTxPool.put(txIdStr, tx); 
```

下个片段，检查当前节点是否是中心节点。在我们的实现中，中心节点并不会挖矿。它只会将新的交易推送给网络中的其他节点。

```java
 // 说明主节点自己
        if (Server.knowNodes.get(0).equals(Server.nodeAddress)) {
            // 给矿工节点发送交易 hash
            List<String> txList = new ArrayList<>();
            txList.add(ByteUtils.bytesToHexString(tx.getTxId()));
            for (String toAddr : Server.knowNodes) {
                ServerSend.sendInv(toAddr, ServerConst.TX_TYPE, txList);
            }
        } 
```

下一个很大的代码片段是矿工节点“专属”。让我们对它进行一下分解:

`miningAddress` 只会在矿工节点上设置。如果当前节点（矿工）的内存池中有 1 笔或更多的交易，开始挖矿：

```java
 //验证数字签名。。
            try {
                if (!bc.verifyTransactions(tx)) {
                    log.info("ERROR: Invalid transaction..");
                }
            } catch (Exception e) {
                e.printStackTrace();
            } 
```

首先，内存池中所有交易都是通过验证的。无效的交易会被忽略，如果没有有效交易，则挖矿中断。

```java
List<Transaction> txs = new ArrayList<>();
            txs.add(tx);
            //奖励
            Transaction coinbaseTx = Transaction.newCoinbaseTX(Server.minerAddress, "");
            txs.add(coinbaseTx);
            //挖掘新区块
            Block lastBlock = RocksDBUtils.getInstance(bc.getNodeID()).getLastBlock();

            Block newBlock = Block.newBlock(lastBlock.getHash(), txs, lastBlock.getHeight() + 1);

            RocksDBUtils.getInstance(bc.getNodeID()).putBlock(newBlock);
            RocksDBUtils.getInstance(bc.getNodeID()).putLastBlockHash(newBlock.getHash());

            utxoSet.update(newBlock);

            ServerSend.sendBlock(Server.knowNodes.get(0), newBlock); 
```

验证后的交易被放到一个块里，同时还有附带奖励的 `coinbase` 交易。当块被挖出来以后，UTXO 集会被更新。

#### 4.5.8 结果

让我们来回顾一下上面定义的场景。

首先，在第一个终端窗口中将 `NODE_ID` 设置为 3000（`export NODE_ID=3000`）。为了让你知道什么节点执行什么操作，我会使用像 **NODE 3000** 或 **NODE 3001** 进行标识。

**NODE 3000**

首先我们打开一个终端，模拟主节点，输入以下命令，创建一个钱包地址：

```java
hanru:part10_Network ruby$ export NODE_ID=3000
hanru:part10_Network ruby$ ./blockchain.sh h
hanru:part10_Network ruby$ ./blockchain.sh createwallet 
```

运行结果如下：

![`img.kongyixueyuan.com/012_007_%E8%BF%90%E8%A1%8C%E7%BB%93%E6%9E%9C1.png`](img/7b9414265010da41560530ddccb23899.jpg)

继续输入以下命令，一个新的区块链：

```java
 hanru:part10_Network ruby$ ./blockchain.sh createblockchain -address 1AqJriaBRLHMatNSmCvSfWkjtAYKPWJssK
hanru:part10_Network ruby$ ./blockchain.sh printchain
hanru:part10_Network ruby$ cp -r blockchain_3000.db blockchain_genesis.db 
```

运行结果如下：

![`img.kongyixueyuan.com/012_008_%E4%B8%BB%E8%8A%82%E7%82%B9.png`](img/d5365047d03686a2ca3154652e82b8d2.jpg)

然后，会生成一个仅包含创世块的区块链。我们需要保存块，并在其他节点使用。创世块承担了一条链标识符的角色（在 Bitcoin Core 中，创世块是硬编码的），所以我们备份了创世区块。

**NODE 3001**

接下来，打开一个新的终端窗口，将 `node ID` 设置为 3001。这会作为一个钱包节点。

输入以下命令：

```java
hanru:part10_Network ruby$ export NODE_ID=3001
hanru:part10_Network ruby$ cp -r blockchain_genesis.db blockchain_3001.db
hanru:part10_Network ruby$ ./blockchain.sh h
hanru:part10_Network ruby$ ./blockchain.sh createwallet
hanru:part10_Network ruby$ ./blockchain.sh createwallet 
```

运行结果如下 ：

![`img.kongyixueyuan.com/012_009_%E9%92%B1%E5%8C%85%E8%8A%82%E7%82%B9.png`](img/79bac329e344c817691784af68f0a59b.jpg)

**NODE 3000**

向钱包地址发送一些币，在主节点终端输入以下命令：

```java
hanru:part10_Network ruby$ ./blockchain.sh send -from 1AqJriaBRLHMatNSmCvSfWkjtAYKPWJssK -to 1MJ82eQi56X9rPdjpbeTnitee2kzPV1KgG -amount 4 -mine true 
```

`-mine` 标志指的是块会立刻被同一节点挖出来。我们必须要有这个标志，因为初始状态时，网络中没有矿工节点。

运行结果如下：

![`img.kongyixueyuan.com/012_010_%E4%B8%BB%E8%8A%82%E7%82%B9%E8%BD%AC%E8%B4%A6.png`](img/709759d3f481c67201eef4202565904e.jpg)

继续输入命令，查询余额：

```java
hanru:part10_Network ruby$ ./blockchain.sh getbalance -address 1AqJriaBRLHMatNSmCvSfWkjtAYKPWJssK
hanru:part10_Network ruby$ ./blockchain.sh getbalance -address 1MJ82eQi56X9rPdjpbeTnitee2kzPV1KgG 
```

运行效果如下：

![`img.kongyixueyuan.com/012_011_%E6%9F%A5%E8%AF%A2%E4%BD%99%E9%A2%9D.png`](img/c376cde670b497a6c5c2728646e4fdf5.jpg)

**NODE 3001**

切换到钱包节点，输入命令查询余额：

```java
hanru:part10_Network ruby$ ./blockchain.sh getbalance -address 1AqJriaBRLHMatNSmCvSfWkjtAYKPWJssK
hanru:part10_Network ruby$ ./blockchain.sh getbalance -address 1MJ82eQi56X9rPdjpbeTnitee2kzPV1KgG 
```

运行结果如下：

![`img.kongyixueyuan.com/012_012_%E6%9F%A5%E8%AF%A2%E4%BD%99%E9%A2%9D.png`](img/f3cfd8af47924d68fcff53026a664185.jpg)

我们发现钱包节点的查询到的余额和主节点中的数据不一致，接下来我们启动节点进行数据同步

**NODE 3000**

切换到主节点的终端，输入以下命令，启动主节点：

```java
hanru:part10_Network ruby$ ./blockchain.sh startnode 
```

启动主节点后等待钱包节点链接，如果 有钱包节点链接，那么就会有消息传递，进行数据同步，运行效果如下：

![`img.kongyixueyuan.com/012_013_%E5%90%AF%E5%8A%A8%E4%B8%BB%E8%8A%82%E7%82%B9.png`](img/cdae2dce035214dc849cf67781240538.jpg)

这个节点会持续运行，直到本文定义的场景结束。

**NODE 3001**

接下来切换到钱包节点终端，输入以下命令，启动钱包节点，进行数据同步：

```java
hanru:day08_10_Net ruby$ ./bc startnode 
```

运行效果如下：

![`img.kongyixueyuan.com/012_014_%E5%90%AF%E5%8A%A8%E9%92%B1%E5%8C%85%E8%8A%82%E7%82%B9.png`](img/2ac9d27a7bc9c53bf1ee6af98a6b935c.jpg)

同步数据后，输入 ctrl+c 键，强制结束。

然后重新输入以下命令进行查看数据是否同步：

```java
hanru:part10_Network ruby$ ./blockchain.sh printchain
hanru:part10_Network ruby$ ./blockchain.sh getbalance -address 1AqJriaBRLHMatNSmCvSfWkjtAYKPWJssK
hanru:part10_Network ruby$ ./blockchain.sh getbalance -address 1MJ82eQi56X9rPdjpbeTnitee2kzPV1KgG 
```

运行结果如下，打印区块，我们发现 blockchain_3001.db 中已经有了 2 个 block：

![`img.kongyixueyuan.com/012_015_%E6%89%93%E5%8D%B0%E5%8C%BA%E5%9D%97.png`](img/6a1be927a6f1dd8ccb77cd6d104e1bec.jpg)

查询余额，效果如下：

![`img.kongyixueyuan.com/012_016_%E6%9F%A5%E8%AF%A2%E4%BD%99%E9%A2%9D.png`](img/8dc2d8618abf25446df4a3f21ced4dcb.jpg)

**NODE 3002**

打开一个新的终端窗口，将它的 `NODE_ID` 设置为 3002，然后生成一个钱包。这会是一个矿工节点。

在终端输入以下命令，初始化区块链：

```java
hanru:part10_Network ruby$ export NODE_ID=3002
hanru:part10_Network ruby$ cp -r blockchain_genesis.db blockchain_3002.db
hanru:part10_Network ruby$ ./blockchain.sh h
hanru:part10_Network ruby$ ./blockchain.sh printchain 
```

运行效果如下：

![`img.kongyixueyuan.com/012_017_%E7%9F%BF%E5%B7%A5%E8%8A%82%E7%82%B9.png`](img/28c785725009c471b560643f03a1d82c.jpg)

创建地址，启动节点指定奖励地址：

```java
hanru:part10_Network ruby$ ./blockchain.sh createwallet
hanru:part10_Network ruby$ ./blockchain.sh startnode -address 1BDv3qpjQBHUZsczu5rLabrmggp1ZDMEBi 
```

启动矿工节点后，会先同步主节点的数据，效果如下:

![`img.kongyixueyuan.com/012_018_%E7%9F%BF%E5%B7%A5%E5%90%8C%E6%AD%A5.png`](img/af21b0f21c13089e5fc6322cb74bd11e.jpg)

**NODE 3001**

保证主节点和矿工节点启动，然后切换到钱包节点，进行转账，在终端输入命令如下：

```java
hanru:part10_Network ruby$ ./blockchain.sh send -from 1MJ82eQi56X9rPdjpbeTnitee2kzPV1KgG -to 1D1PiWvfKoFqTxbGmyDfWLsdnek5hMpDcu -amount 3 -mine false 
```

本次转账没有理解挖矿，所以会交由矿工节点进行挖矿，效果如下：

![`img.kongyixueyuan.com/012_018_%E7%9F%BF%E5%B7%A5%E5%90%8C%E6%AD%A5.png`](img/af21b0f21c13089e5fc6322cb74bd11e.jpg)

**NODE 3002**

迅速切换到矿工节点，你会看到挖出了一个新块！同时查询余额是最新的数据。

```java
# 钱包节点的两个地址
hanru:part10_Network ruby$ ./blockchain.sh getbalance -address 1MJ82eQi56X9rPdjpbeTnitee2kzPV1KgG
hanru:part10_Network ruby$ ./blockchain.sh getbalance -address 1D1PiWvfKoFqTxbGmyDfWLsdnek5hMpDcu
# 矿工节点指定的奖励地址
hanru:part10_Network ruby$ ./blockchain.sh getbalance -address 1BDv3qpjQBHUZsczu5rLabrmggp1ZDMEBi 
```

运行效果如下：

![`img.kongyixueyuan.com/012_019_%E7%9F%BF%E5%B7%A5%E6%8C%96%E7%9F%BF.png`](img/21434c818133a305d419a211a0d31e3f.jpg)

查询余额：

![`img.kongyixueyuan.com/012_020_%E6%9F%A5%E8%AF%A2%E4%BD%99%E9%A2%9D.png`](img/ccfaab198cdb01ed1ef80dc1644a5411.jpg)

**NODE 3001**

切换到钱包节点并启动：

```java
hanru:part10_Network ruby$ ./blockchain.sh startnode
hanru:part10_Network ruby$ ./blockchain.sgetbalance -address 1MJ82eQi56X9rPdjpbeTnitee2kzPV1KgG
hanru:part10_Network ruby$ ./blockchain.sh getbalance -address 1D1PiWvfKoFqTxbGmyDfWLsdnek5hMpDcu 
```

它会下载最近挖出来的块，暂停节点并检查余额：

运行效果如下：

![`img.kongyixueyuan.com/012_021_%E9%92%B1%E5%8C%85%E5%90%AF%E5%8A%A8.png`](img/34c9da9066bb85050f6e23a3b2bf5c4e.jpg)

钱包节点查询余额：

![`img.kongyixueyuan.com/012_022_%E6%9F%A5%E8%AF%A2%E4%BD%99%E9%A2%9D.png`](img/7746bae3c4cef26b9684df5a0a0162c2.jpg)

就是这么多了！

## 5\. 总结

本章节的学习，我们实现了简易版的网络，能够同步节点之间的数据消息，尽管这个过程比较复杂。我们已经尽可能的简化了，仅仅是通过端口来模拟不同的节点。实现了主节点，钱包节点和矿工节点之间的数据传递。

最后，这是本系列的最后一篇文章了，希望本文已经回答了关于比特币技术的一些问题，也给读者提出了一些问题，这些问题你可以自行寻找答案。在比特币技术中还有隐藏着很多有趣的事情！好运！

[项目源代码](https://github.com/rubyhan1314/BitcoinForJava)