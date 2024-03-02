# 第八章 比特币地址生成解析

# 第八章 地址(Address)

在上一篇文章中，我们已经初步实现了交易。相信你应该了解了交易中的一些天然属性，这些属性没有丝毫“个人”色彩的存在：在比特币中，没有用户账户，不需要也不会在任何地方存储个人数据（比如姓名，护照号码或者 SSN）。但是，我们总要有某种途径识别出你是交易输出的所有者（也就是说，你拥有在这些输出上锁定的币）。这就是比特币地址（address）需要完成的使命。在上一篇中，我们把一个由用户定义的任意字符串当成是地址，现在我们将要实现一个跟比特币一样的真实地址。

## 1\. 课程目标

1.  学会什么是 Base58 编码和解码

2.  学会创建私钥和公钥

3.  学会使用公钥生成钱包地址

4.  学会将钱包地址进行持久化存储

## 2\. 项目代码及效果展示

### 2.1 项目代码结构

![`img.kongyixueyuan.com/008_001_%E9%A1%B9%E7%9B%AE%E7%BB%93%E6%9E%84.png`](img/edde8f6341f3702cbef53388d45b448a.jpg)

### 2.2 项目运行结果

![`img.kongyixueyuan.com/008_004_%E8%BF%90%E8%A1%8C%E6%95%88%E6%9E%9C.gif`](img/c59931dbd5a039705651313c90abe677.jpg)

## 3\. 创建项目

### 3.1 创建工程

打开 IntelliJ IDEA 的工作空间，将上一个项目代码目录`part5_Transaction`，复制为`part6_Wallet`。

然后打开 IntelliJ IDEA 开发工具。

打开工程：`part6_Wallet`，并删除 target 目录。然后进行以下修改：

```java
step1：先将项目重新命名为：part6_Wallet。
step2：修改 pom.xml 配置文件。
    改为：<artifactId>part6_Wallet</artifactId>标签
    改为：<name>part6_Wallet Maven Webapp</name> 
```

> 说明：我们每一章节的项目代码，都是在上一个章节上进行添加。所以拷贝上一次的项目代码，然后进行新内容的添加或修改。

### 3.2 代码实现

#### 3.2.1 新建 java 文件：`Base58Check.java`

打开`cldy.hanru.blockchain.util`包，新建`Base58Check.java`文件。添加 Base58 的编码和解码方法，代码如下:

```java
 package cldy.hanru.blockchain.util;

import java.io.ByteArrayOutputStream;
import java.io.IOException;
import java.math.BigInteger;
import java.util.Arrays;

/**
 * Base58 转化工具
 *
 * @author hanru
 *
 */
public final class Base58Check {

    /*---- Class constants ----*/

    private static final String ALPHABET = "123456789ABCDEFGHJKLMNPQRSTUVWXYZabcdefghijkmnopqrstuvwxyz";
    private static final BigInteger ALPHABET_SIZE = BigInteger.valueOf(ALPHABET.length());

    /**
     * 添加校验码并转化为 Base58 字符串
     *
     * @param data
     * @return
     */
    public static String bytesToBase58(byte[] data) {
        return rawBytesToBase58(addCheckHash(data));
    }

    /**
     * 转化为 Base58 字符串
     *
     * @param data
     * @return
     */
    public static String rawBytesToBase58(byte[] data) {
        // Convert to base-58 string
        StringBuilder sb = new StringBuilder();
        BigInteger num = new BigInteger(1, data);
        while (num.signum() != 0) {
            BigInteger[] quotrem = num.divideAndRemainder(ALPHABET_SIZE);
            sb.append(ALPHABET.charAt(quotrem[1].intValue()));
            num = quotrem[0];
        }

        // Add '1' characters for leading 0-value bytes
        for (int i = 0; i < data.length && data[i] == 0; i++) {
            sb.append(ALPHABET.charAt(0));
        }
        return sb.reverse().toString();
    }

    /**
     * 添加校验码并返回待有校验码的原生数据
     *
     * @param data
     * @return
     */
    static byte[] addCheckHash(byte[] data) {
        try {
            byte[] hash = Arrays.copyOf(AddressUtils.doubleHash(data), 4);
            ByteArrayOutputStream buf = new ByteArrayOutputStream();
            buf.write(data);
            buf.write(hash);
            return buf.toByteArray();
        } catch (IOException e) {
            throw new AssertionError(e);
        }
    }

    /**
     * 将 Base58Check 字符串转化为 byte 数组，并校验其校验码
     * 返回的 byte 数组带有版本号，但不带有校验码
     *
     * @param s
     * @return
     */
    public static byte[] base58ToBytes(String s) {
        byte[] concat = base58ToRawBytes(s);
        byte[] data = Arrays.copyOf(concat, concat.length - 4);
        byte[] hash = Arrays.copyOfRange(concat, concat.length - 4, concat.length);
        byte[] rehash = Arrays.copyOf(AddressUtils.doubleHash(data), 4);
        if (!Arrays.equals(rehash, hash)) {
            throw new IllegalArgumentException("Checksum mismatch");
        }
        return data;
    }

    /**
     * 将 Base58Check 字符串反转为 byte 数组
     *
     * @param s
     * @return
     */
    static byte[] base58ToRawBytes(String s) {
        // Parse base-58 string
        BigInteger num = BigInteger.ZERO;
        for (int i = 0; i < s.length(); i++) {
            num = num.multiply(ALPHABET_SIZE);
            int digit = ALPHABET.indexOf(s.charAt(i));
            if (digit == -1) {
                throw new IllegalArgumentException("Invalid character for Base58Check");
            }
            num = num.add(BigInteger.valueOf(digit));
        }
        // Strip possible leading zero due to mandatory sign bit
        byte[] b = num.toByteArray();
        if (b[0] == 0) {
            b = Arrays.copyOfRange(b, 1, b.length);
        }
        try {
            // Convert leading '1' characters to leading 0-value bytes
            ByteArrayOutputStream buf = new ByteArrayOutputStream();
            for (int i = 0; i < s.length() && s.charAt(i) == ALPHABET.charAt(0); i++) {
                buf.write(0);
            }
            buf.write(b);
            return buf.toByteArray();
        } catch (IOException e) {
            throw new AssertionError(e);
        }
    }
    /*---- Miscellaneous ----*/

    private Base58Check() {
    }  // Not instantiable

} 
```

#### 3.2.2 创建 java 文件：`AddressUtils.java`

打开`cldy.hanru.blockchain.util`包，新建`AddressUtils.java`文件。在`AddressUtils.java`文件中编写代码如下：

```java
package cldy.hanru.blockchain.util;

import org.apache.commons.codec.digest.DigestUtils;
import org.bouncycastle.crypto.digests.RIPEMD160Digest;
import org.bouncycastle.util.Arrays;

/**
 * 地址工具类
 *
 * @author hanru
 *
 */
public class AddressUtils {
    /**
     * 双重 Hash
     *
     * @param data
     * @return
     */
    public static byte[] doubleHash(byte[] data) {
        return DigestUtils.sha256(DigestUtils.sha256(data));
    }

    /**
     * 计算公钥的 RIPEMD160 Hash 值
     *
     * @param pubKey 公钥
     * @return ipeMD160Hash(sha256 ( pubkey))
     */
    public static byte[] ripeMD160Hash(byte[] pubKey) {
        //1\. 先对公钥做 sha256 处理
        byte[] shaHashedKey = DigestUtils.sha256(pubKey);
        RIPEMD160Digest ripemd160 = new RIPEMD160Digest();
        ripemd160.update(shaHashedKey, 0, shaHashedKey.length);
        byte[] output = new byte[ripemd160.getDigestSize()];
        ripemd160.doFinal(output, 0);
        return output;
    }
    /**
     * 生成公钥的校验码
     *
     * @param payload
     * @return
     */
    public static byte[] checksum(byte[] payload) {
        return Arrays.copyOfRange(doubleHash(payload), 0, 4);
    }
} 
```

####

#### 3.2.2 创建 java 文件：`Wallet.java`

新建`wallet`包，并创建`Wallet.java`文件。在`Wallet.java`文件中编写代码如下：

```java
package cldy.hanru.blockchain.wallet;

import cldy.hanru.blockchain.util.AddressUtils;
import cldy.hanru.blockchain.util.Base58Check;
import lombok.AllArgsConstructor;
import lombok.Data;
import org.bouncycastle.jcajce.provider.asymmetric.ec.BCECPrivateKey;
import org.bouncycastle.jcajce.provider.asymmetric.ec.BCECPublicKey;
import org.bouncycastle.jce.ECNamedCurveTable;
import org.bouncycastle.jce.provider.BouncyCastleProvider;
import org.bouncycastle.jce.spec.ECParameterSpec;

import java.io.ByteArrayOutputStream;
import java.io.IOException;
import java.io.Serializable;
import java.security.KeyPair;
import java.security.KeyPairGenerator;
import java.security.SecureRandom;
import java.security.Security;

/**
 * @author hanru
 */
@AllArgsConstructor
@Data
public class Wallet implements Serializable {

    private static final long serialVersionUID = 6796631943459965436L;

    /**
     * 校验码长度
     */
    private static final int ADDRESS_CHECKSUM_LEN = 4;
    /**
     * 私钥
     */
    private BCECPrivateKey privateKey;
    /**
     * 公钥
     */
    private byte[] publicKey;

    /**
     * 初始化钱包
     */
    private void initWallet() {
        try {
            KeyPair keyPair = newECKeyPair();
            BCECPrivateKey privateKey = (BCECPrivateKey) keyPair.getPrivate();
            BCECPublicKey publicKey = (BCECPublicKey) keyPair.getPublic();

            byte[] publicKeyBytes = publicKey.getQ().getEncoded(false);

            this.setPrivateKey(privateKey);
            this.setPublicKey(publicKeyBytes);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    /**
     * 创建新的密钥对
     *
     * @return
     * @throws Exception
     */
    private KeyPair newECKeyPair() throws Exception {
        // 注册 BC Provider
        Security.addProvider(new BouncyCastleProvider());
        // 创建椭圆曲线算法的密钥对生成器，算法为 ECDSA
        KeyPairGenerator keyPairGenerator = KeyPairGenerator.getInstance("ECDSA", BouncyCastleProvider.PROVIDER_NAME);
        // 椭圆曲线（EC）域参数设定
        // bitcoin 为什么会选择 secp256k1，详见：https://bitcointalk.org/index.php?topic=151120.0
        ECParameterSpec ecSpec = ECNamedCurveTable.getParameterSpec("secp256k1");
        keyPairGenerator.initialize(ecSpec, new SecureRandom());
        return keyPairGenerator.generateKeyPair();
    }

    public Wallet() {
        initWallet();
    }

    /**
     * 获取钱包地址
     *
     * @return
     */
    public String getAddress() {
        try {
            // 1\. 获取 ripemdHashedKey
            byte[] ripemdHashedKey = AddressUtils.ripeMD160Hash(this.getPublicKey());

            // 2\. 添加版本 0x00
            ByteArrayOutputStream addrStream = new ByteArrayOutputStream();
            addrStream.write((byte) 0);
            addrStream.write(ripemdHashedKey);
            byte[] versionedPayload = addrStream.toByteArray();

            // 3\. 计算校验码
            byte[] checksum = AddressUtils.checksum(versionedPayload);

            // 4\. 得到 version + paylod + checksum 的组合
            addrStream.write(checksum);
            byte[] binaryAddress = addrStream.toByteArray();

            // 5\. 执行 Base58 转换处理
            return Base58Check.rawBytesToBase58(binaryAddress);
        } catch (IOException e) {
            e.printStackTrace();
        }
        throw new RuntimeException("Fail to get wallet address ! ");
    }

} 
```

#### 3.2.3 创建`WalletUtils.java`文件

打开`cldy.hanru.blockchain.wallet`包，创建`WalletUtils.java`文件。在`WalletUtils.java`文件中编写代码如下：

添加代码如下：

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
    private final static String WALLET_FILE = "wallet.dat";
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
        private Map<String, Wallet> walletMap = new HashMap();

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

    public static WalletUtils getInstance() {
        if (instance == null) {
            synchronized (WalletUtils.class) {
                if (instance == null) {
                    instance = new WalletUtils();
                }
            }
        }
        return instance;
    }

    private WalletUtils() {
        initWalletFile();
    }

    /**
     * 初始化钱包文件
     */
    private void initWalletFile() {
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

#### 3.2.4 修改`CLI.java`文件

打开`cldy.hanru.blockchain.cli`包，修改`CLI.java`文件。添加两个命令，创建钱包地址和打印钱包地址。

修改步骤：

```java
修改步骤：
step1：修改 help()方法，添加两个命令说明
step2：修改 run()方法，增加两个分支
step3：新增两个方法用于创建钱包和打印钱包
    createWallet()
    printAddresses() 
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
import cldy.hanru.blockchain.wallet.Wallet;
import cldy.hanru.blockchain.wallet.WalletUtils;
import org.apache.commons.cli.*;
import org.apache.commons.codec.binary.Hex;
import org.apache.commons.lang3.StringUtils;
import org.apache.commons.lang3.math.NumberUtils;

import java.text.SimpleDateFormat;
import java.util.*;

public class CLI {
    private String[] args;
    private Options options = new Options();

    public CLI(String[] args) {
        this.args = args;
//        options.addOption("h", "help", false, "show help");
//        options.addOption("creategenesis", "creategenesis", false, "create blockchain with genesis block");
//        options.addOption("add", "addblock", true, "add a block to the blockchain");
//        options.addOption("print", "printchain", false, "print all the blocks of the blockchain");

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
//        HelpFormatter helpFormatter = new HelpFormatter();
//        helpFormatter.printHelp("Main", options);
//        System.exit(0);

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
                    System.out.println("\t\t 输出：");
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
        List<Transaction> transactions = new ArrayList<>();
        transactions.add(transaction);
        blockchain.mineBlock(transactions);
        RocksDBUtils.getInstance().closeDB();
        System.out.println("Success!");
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

#### 3.2.5 修改`blockchain.sh`脚本文件

最后修改`blockchain.sh`脚本文件，修改后内容如下：

```java
#!/bin/bash

set -e

# Check if the jar has been built.
if [ ! -e target/part6_Wallet-jar-with-dependencies.jar ]; then
  echo "Compiling blockchain project to a JAR"
  mvn package -DskipTests
fi

java -jar target/part6_Wallet-jar-with-dependencies.jar "$@" 
```

##

## 4\. 钱包地址讲解

### 4.1 比特币地址

这就是一个真实的比特币地址：[1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa](https://blockchain.info/address/1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa)。这是史上第一个比特币地址，据说属于中本聪。比特币地址是完全公开的，如果你想要给某个人发送币，只需要知道他的地址就可以了。但是，地址（尽管地址也是独一无二的）并不是用来证明你是一个“钱包”所有者的信物。实际上，所谓的地址，只不过是将公钥表示成人类可读的形式而已，因为原生的公钥人类很难阅读。在比特币中，你的身份（identity）就是一对（或者多对）保存在你的电脑（或者你能够获取到的地方）上的公钥（public key）和私钥（private key）。比特币基于一些加密算法的组合来创建这些密钥，并且保证了在这个世界上没有其他人能够取走你的币，除非拿到你的密钥。下图是创建钱包后到产生地址的流程图。

![`img.kongyixueyuan.com/0703_%E6%B5%81%E7%A8%8B%E5%9B%BE.png`](img/a830e702f3d26da29340acd908693ec5.jpg)

> 1、对于比特币来说，钱不是支付给个人的，而是支付给某一把私钥。这就是交易匿名性的根本原因，因为没有人知道，那些私钥背后的主人是谁。所以，比特币交易的第一件事，就是你必须拥有自己的公钥和私钥。
> 
> 2、去网上那些比特币交易所开户，它们会让你首先生成一个比特币钱包（wallet）。这个钱包不是用来存放比特币，而是存放你的公钥和私钥。软件会帮你生成这两把钥匙，然后放在钱包里面。根据协议，公钥的长度是 512 位。这个长度不太方便传播，因此协议又规定，要为公钥生成一个 160 位的指纹。所谓指纹，就是一个比较短的、易于传播的哈希值。160 位是二进制，写成十六进制，大约是 26 到 35 个字符，比如 1Ko29e6QcUXzhrtHD5aizMa6ustL88RTWR。这个字符串就叫做钱包的地址，它是唯一的，即每个钱包的地址肯定都是不一样的。
> 
> 3、你向别人收钱时，只要告诉对方你的钱包地址即可，对方向这个地址付款。由于你是这个地址的拥有者，所以你会收到这笔钱。由于你是否拥有某个钱包地址，是由私钥证明的，所以一定要保护好私钥。同样的，你向他人支付比特币，千万不能写错他人的钱包地址，否则你的比特币就支付到了另一个不同的人了。

下面，让我们来讨论一下这些算法到底是什么。

### 4.2 公钥加密

公钥加密（public-key cryptography）算法使用的是成对的密钥：公钥和私钥。公钥并不是敏感信息，可以告诉其他人。但是，私钥绝对不能告诉其他人：只有所有者（owner）才能知道私钥，能够识别，鉴定和证明所有者身份的就是私钥。在加密货币的世界中，你的私钥代表的就是你，私钥就是一切。

本质上，比特币钱包也只不过是这样的密钥对而已。当你安装一个钱包应用，或是使用一个比特币客户端来生成一个新地址时，它就会为你生成一对密钥。在比特币中，谁拥有了私钥，谁就可以控制所有发送到这个公钥的币。

私钥和公钥只不过是随机的字节序列，因此它们无法在屏幕上打印，人类也无法通过肉眼去读取。这就是为什么比特币使用了一个转换算法，将公钥转化为一个人类可读的字符串（也就是我们看到的地址）。

> 如果你用过比特币钱包应用，很可能它会为你生成一个助记符。这样的助记符可以用来替代私钥，并且可以被用于生成私钥。[BIP-039](https://github.com/bitcoin/bips/blob/master/bip-0039.mediawiki) 已经实现了这个机制。

### 4.3 椭圆曲线加密

正如之前提到的，公钥和私钥是随机的字节序列。私钥能够用于证明持币人的身份，需要有一个条件：随机算法必须生成真正随机的字节。因为没有人会想要生成一个私钥，而这个私钥意外地也被别人所有。

比特币使用椭圆曲线来产生私钥。椭圆曲线是一个复杂的数学概念，我们并不打算在这里作太多解释（如果你真的十分好奇，可以查看[这篇文章](http://andrea.corbellini.name/2015/05/17/elliptic-curve-cryptography-a-gentle-introduction/)，注意：有很多数学公式！）我们只要知道这些曲线可以生成非常大的随机数就够了。在比特币中使用的曲线可以随机选取在 0 与 2 ^ 2 ^ 56（大概是 10⁷⁷, 而整个可见的宇宙中，原子数在 10⁷⁸ 到 10⁸² 之间） 的一个数。有如此高的一个上限，意味着几乎不可能发生有两次生成同一个私钥的事情。

椭圆曲线算法（ECC，Elliptic Curve Cryptography）

*   椭圆曲线算法是不可逆的，很容易向一个方向计算，但是不可以向相反方向倒推。
*   比特币使用 SECP256K1 算法（椭圆曲线算法的一种），将私钥生成公钥。
*   SECP256K1 是不可逆的，因此公钥公开暴露，也对私钥的安全性不会造成影响。

比特币使用的是 ECDSA（Elliptic Curve Digital Signature Algorithm）算法来对交易进行签名，我们也会使用该算法。

**比特币钱包随机生成私钥的安全性**

每个用户的私钥是由比特币钱包随机生成的。可能有人会有疑问，这样随机生成的私钥安不安全，会不会我随机生成私钥跟别人的私钥恰好重复了？这个你不用过于担心，因为，比特币私钥是一串很长的文本，理论上比特币私钥的总数是：1461501637330902918203684832716283019655932542976

![`img.kongyixueyuan.com/0704_%E9%9A%8F%E6%9C%BA%E6%95%B0.png`](img/58abbe7014a56e1adf3e1211b4c6087b.jpg)

这是个什么概念？地球上的沙子是 7.5 乘以 10 的 18 次方，然后你想象一下，每粒沙子又是一个地球，这时的沙子总数是 56.25 乘以 10 的 36 次方，仍然远小于比特币私钥的总数。所以在这样一个庞大的地址空间下，私钥重复的概率微乎其微。

![`img.kongyixueyuan.com/0705_%E9%9A%8F%E6%9C%BA%E6%95%B02.png`](img/b124b2b6c1e8f918ff045795992c06bd.jpg)

### 4.4 Base64 和 Base58 编码和解码

Base64 就是一种基于 64 个可打印字符来表示二进制数据的方法

*   Base64 使用了 26 个小写字母、26 个大写字母、10 个数字以及两个符号（例如“+”和“/”），用于在电子邮件这样的基于文本的媒介中传输二进制数据。

*   Base64 通常用于编码邮件中的附件。

Base64 字符集：

```java
ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/ 
```

![`img.kongyixueyuan.com/0706_base64.png`](img/f6ac2957f4f6b49771585310a128cb2b.jpg)

Base58 是一种基于文本的二进制编码格式，是用于 Bitcoin 中使用的一种独特的编码方式，主要用于产生 Bitcoin 的钱包地址。

*   相比 Base64，Base58 不使用数字"0"，大写字母"O"，大写字母"I"和小写字母"l"，以及"+"和"/"符号。目的就是去除容易混淆的字符。
*   这种编码格式不仅实现了数据压缩，保持了易读性，还具有错误诊断功能。

Base58 字符集：

```java
ABCDEFGHJKLMNPQRSTUVWXYZabcdefghijkmnopqrstuvwxyz123456789 
```

![`img.kongyixueyuan.com/0707_base58-%E8%A1%A8.jpg`](img/9c3d2c621dcdfba2fa0918c430d52e76.jpg)

**Base58Check**

Base58Check 是一种常用在比特币中的 Base58 编码格式，增加了错误校验码来检查数据在转录中出现的错误。在 Base58Check 中，对数据添加了一个称作“版本字节”的前缀，这个前缀用来明确需要编码的数据的类型。

Base58Check 的作用

*   既然有了 Base58 编码，已经不会搞错 0 和 O, 1 和 l 和 I，也把大整数转换成了可读字符串，为什么还要再有 Base58Check 这个环节呢？
*   假设一种情况，你在程序中输入一个 Base58 编码的地址，尽管你已经不会搞错 0 和 O, 1 和 l 和 I，但是万一你不小心输错一个字符，或者少写多写一个字符，会咋样？你可能会说，没啥大不了的，错个字符而已，这不是很常见嘛，重新输入不就可以了吗？但是当用户给一个比特币地址转账，如果输入错误，那么对方就不会收到资金，更关键的是该笔资金发给了一个根本不存在的比特币地址，那么这笔资金也就永远不可能被交易，也就是说比特币丢失了。
*   校验码长 4 个字节，添加到需要编码的数据之后。
*   校验码是从需要编码的数据的哈希值中得到的，所以可以用来检测并避免转录和输入中产生的错误。
*   使用 Base58check 编码格式时，程序会计算原始数据的校验码并和自带的校验码进行对比，二者若不匹配则表明有错误产生。
*   实际上，在比特币交易中，都会校验比特币地址是否合法，如果经过 Base58Check 的比特币地址被比特币钱包程序判定是无效的，当然会阻止交易继续进行，就避免了资金损失。

### 4.5 生成地址

回到上面提到的比特币地址：1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa 。现在，我们已经知道了这是公钥用人类可读的形式表示而已。如果我们对它进行解码，就会看到公钥的本来面目（16 进制表示的字节）：0062E907B15CBF27D5425399EBF6F0FB50EBB88F18C29B7D93

比特币使用 Base58 算法将公钥转换成人类可读的形式。这个算法跟著名的 Base64 很类似，区别在于它使用了更短的字母表：为了避免一些利用字母相似性的攻击，从字母表中移除了一些字母。也就是，没有这些符号：0(零)，O(大写的 o)，I(大写的 i)，l(小写的 L)，因为这几个字母看着很像。另外，也没有 + 和 / 符号。

比特币地址生成过程中的 Base58Check 的步骤如下：

```java
1\. 获取到公钥
2\. 计算公钥的 SHA-256 哈希值（对公钥进行第一次 hash256 运算）
3\. 计算 RIPEMD-160 哈希值（对第一次 hash256 的结果进行 ripeMD160 运算）
4\. 添加版本前缀（将 ripeMD160 运算的结果前增加版本编号）
5\. 计算两次 hash（对加上版本编号的 hash 值计算两次 hash）
6\. 获取校验码（获取两次 hash 后前四个字节，作为校验码）
7\. 形成比特币地址的 16 进制格式（将校验码作为比特币地址的后缀，与添加版本前缀的 hash 值拼接）
8\. 进行 Base58 编码处理（对比特币地址的 16 进制格式进行 Base58 编码，就形成了比特币地址） 
```

![`img.kongyixueyuan.com/0708_address2.jpg`](img/49c849051fdfa52926012c7766cee29d.jpg)

因此，为了使用 Base58Check 编码格式对数据（数字）进行编码，首先我们要对数据添加一个称作“版本字节”的前缀，这个前缀用来明确需要编码的数据的类型。

> 例如，比特币地址的前缀是 0（十六进制是 0x00），而对私钥编码时前缀是 128（十六进制是 0x80）。

所以公钥解码后包含三个部分：

```java
Version  Public key hash                           Checksum
00       62E907B15CBF27D5425399EBF6F0FB50EBB88F18  C29B7D93 
```

由于哈希函数是单向的（也就说无法逆转回去），所以不可能从一个哈希中提取公钥。不过通过执行哈希函数并进行哈希比较，我们可以检查一个公钥是否被用于哈希的生成。

### 4.6 Base58 代码实现

好了，所有细节都已就绪，来写代码吧。很多概念只有当写代码的时候，才能理解地更透彻。

由于 Base58 是在货币中特有的，并不是像 Base64 那样是通用编码方式，所以 golang 自带的包中没有实现，(Base64 是有的，在"encoding/base64"包下)，那么就需要我们自己写工具方法实现编码和解码。

在`util`包下，新建一个 java 文件，命名为`Base58Check.java`，并添加代码如下：

```java
 package cldy.hanru.blockchain.util;

import java.io.ByteArrayOutputStream;
import java.io.IOException;
import java.math.BigInteger;
import java.util.Arrays;

/**
 * Base58 转化工具
 *
 * @author hanru
 *
 */
public final class Base58Check {

    /*---- Class constants ----*/

    private static final String ALPHABET = "123456789ABCDEFGHJKLMNPQRSTUVWXYZabcdefghijkmnopqrstuvwxyz";
    private static final BigInteger ALPHABET_SIZE = BigInteger.valueOf(ALPHABET.length());

    /**
     * 添加校验码并转化为 Base58 字符串
     *
     * @param data
     * @return
     */
    public static String bytesToBase58(byte[] data) {
        return rawBytesToBase58(addCheckHash(data));
    }

    /**
     * 转化为 Base58 字符串
     *
     * @param data
     * @return
     */
    public static String rawBytesToBase58(byte[] data) {
        // Convert to base-58 string
        StringBuilder sb = new StringBuilder();
        BigInteger num = new BigInteger(1, data);
        while (num.signum() != 0) {
            BigInteger[] quotrem = num.divideAndRemainder(ALPHABET_SIZE);
            sb.append(ALPHABET.charAt(quotrem[1].intValue()));
            num = quotrem[0];
        }

        // Add '1' characters for leading 0-value bytes
        for (int i = 0; i < data.length && data[i] == 0; i++) {
            sb.append(ALPHABET.charAt(0));
        }
        return sb.reverse().toString();
    }

    /**
     * 添加校验码并返回待有校验码的原生数据
     *
     * @param data
     * @return
     */
    static byte[] addCheckHash(byte[] data) {
        try {
            byte[] hash = Arrays.copyOf(AddressUtils.doubleHash(data), 4);
            ByteArrayOutputStream buf = new ByteArrayOutputStream();
            buf.write(data);
            buf.write(hash);
            return buf.toByteArray();
        } catch (IOException e) {
            throw new AssertionError(e);
        }
    }

    /**
     * 将 Base58Check 字符串转化为 byte 数组，并校验其校验码
     * 返回的 byte 数组带有版本号，但不带有校验码
     *
     * @param s
     * @return
     */
    public static byte[] base58ToBytes(String s) {
        byte[] concat = base58ToRawBytes(s);
        byte[] data = Arrays.copyOf(concat, concat.length - 4);
        byte[] hash = Arrays.copyOfRange(concat, concat.length - 4, concat.length);
        byte[] rehash = Arrays.copyOf(AddressUtils.doubleHash(data), 4);
        if (!Arrays.equals(rehash, hash)) {
            throw new IllegalArgumentException("Checksum mismatch");
        }
        return data;
    }

    /**
     * 将 Base58Check 字符串反转为 byte 数组
     *
     * @param s
     * @return
     */
    static byte[] base58ToRawBytes(String s) {
        // Parse base-58 string
        BigInteger num = BigInteger.ZERO;
        for (int i = 0; i < s.length(); i++) {
            num = num.multiply(ALPHABET_SIZE);
            int digit = ALPHABET.indexOf(s.charAt(i));
            if (digit == -1) {
                throw new IllegalArgumentException("Invalid character for Base58Check");
            }
            num = num.add(BigInteger.valueOf(digit));
        }
        // Strip possible leading zero due to mandatory sign bit
        byte[] b = num.toByteArray();
        if (b[0] == 0) {
            b = Arrays.copyOfRange(b, 1, b.length);
        }
        try {
            // Convert leading '1' characters to leading 0-value bytes
            ByteArrayOutputStream buf = new ByteArrayOutputStream();
            for (int i = 0; i < s.length() && s.charAt(i) == ALPHABET.charAt(0); i++) {
                buf.write(0);
            }
            buf.write(b);
            return buf.toByteArray();
        } catch (IOException e) {
            throw new AssertionError(e);
        }
    }
    /*---- Miscellaneous ----*/

    private Base58Check() {
    }  // Not instantiable

} 
```

在这个工具类中，我们提供了几个方法，逐一说明一下：bytesToBase58()，用于将字节数组，用 Base58 编码转为字符串。base58ToBytes()，用于将地址再转为字节数组，并对比校验码，进行校验地址是否有效。验证原理就是，将地址进行 Base58 解码，得到数据：版本号+hash 数据+checksum，将版本号+hash 数据，重新生成新的 checksum，和原来的 checksum 进行比较，如果不同，说明地址无效，抛出异常打断程序执行。

还有一个点需要单独强调一下，在 rawBytesToBase58()方法中：

```java
 // Add '1' characters for leading 0-value bytes
for (int i = 0; i < data.length && data[i] == 0; i++) {
    sb.append(ALPHABET.charAt(0));
} 
```

对于 Base58 编码，最后总会拼接下标 0 对应的 Base58 字符，就是 1。所以我们所产生的地址的第一位，都是 1。比如：1NH3bAuMAyXHnCrBmykPcKciBG6W3Dc5vY。

### 4.7 创建钱包生成地址

我们先从钱包 `Wallet` 结构开始：

在`cldy.hanru.blockchain.wallet`包下，新建`Wallet.java`文件，并添加 Wallet 类。

```java
public class Wallet implements Serializable {

    private static final long serialVersionUID = 6796631943459965436L;

    /**
     * 校验码长度
     */
    private static final int ADDRESS_CHECKSUM_LEN = 4;
    /**
     * 私钥
     */
    private BCECPrivateKey privateKey;
    /**
     * 公钥
     */
    private byte[] publicKey;

    /**
     * 初始化钱包
     */
    private void initWallet() {
        try {
            KeyPair keyPair = newECKeyPair();
            BCECPrivateKey privateKey = (BCECPrivateKey) keyPair.getPrivate();
            BCECPublicKey publicKey = (BCECPublicKey) keyPair.getPublic();

            byte[] publicKeyBytes = publicKey.getQ().getEncoded(false);

            this.setPrivateKey(privateKey);
            this.setPublicKey(publicKeyBytes);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    /**
     * 创建新的密钥对
     *
     * @return
     * @throws Exception
     */
    private KeyPair newECKeyPair() throws Exception {
        // 注册 BC Provider
        Security.addProvider(new BouncyCastleProvider());
        // 创建椭圆曲线算法的密钥对生成器，算法为 ECDSA
        KeyPairGenerator keyPairGenerator = KeyPairGenerator.getInstance("ECDSA", BouncyCastleProvider.PROVIDER_NAME);
        // 椭圆曲线（EC）域参数设定
        // bitcoin 为什么会选择 secp256k1，详见：https://bitcointalk.org/index.php?topic=151120.0
        ECParameterSpec ecSpec = ECNamedCurveTable.getParameterSpec("secp256k1");
        keyPairGenerator.initialize(ecSpec, new SecureRandom());
        return keyPairGenerator.generateKeyPair();
    }

    public Wallet() {
        initWallet();
    } 
```

`Wallet` 的构造函数会生成一个新的密钥对。`newKeyPair` 函数非常直观：ECDSA 基于椭圆曲线，所以我们需要一个椭圆曲线。接下来，使用椭圆生成一个私钥，然后再从私钥生成一个公钥。有一点需要注意：在基于椭圆曲线的算法中，公钥是曲线上的点。因此，公钥是 X，Y 坐标的组合。在比特币中，这些坐标会被连接起来，然后形成一个公钥。

现在，来生成一个地址：

```java
 /**
     * 获取钱包地址
     *
     * @return
     */
public String getAddress() {
    try {
        // 1\. 获取 ripemdHashedKey
        byte[] ripemdHashedKey = AddressUtils.ripeMD160Hash(this.getPublicKey());

        // 2\. 添加版本 0x00
        ByteArrayOutputStream addrStream = new ByteArrayOutputStream();
        addrStream.write((byte) 0);
        addrStream.write(ripemdHashedKey);
        byte[] versionedPayload = addrStream.toByteArray();

        // 3\. 计算校验码
        byte[] checksum = AddressUtils.checksum(versionedPayload);

        // 4\. 得到 version + paylod + checksum 的组合
        addrStream.write(checksum);
        byte[] binaryAddress = addrStream.toByteArray();

        // 5\. 执行 Base58 转换处理
        return Base58Check.rawBytesToBase58(binaryAddress);
    } catch (IOException e) {
        e.printStackTrace();
    }
    throw new RuntimeException("Fail to get wallet address ! ");
} 
```

至此，就可以得到一个**真实的比特币地址**，你甚至可以在 [blockchain.info](https://blockchain.info/) 查看它的余额。不过我可以负责任地说，无论生成一个新的地址多少次，检查它的余额都是 0。这就是为什么选择一个合适的公钥加密算法是如此重要：考虑到私钥是随机数，生成同一个数字的概率必须是尽可能地低。理想情况下，必须是低到“永远”不会重复。

一个钱包对象，存储了一对私钥和公钥，公钥生成一个地址。现在我们可以创建一个钱包集合，可以存储多个地址。

我们需要 `Wallets` 类型来保存多个钱包的组合，将它们保存到文件中，或者从文件中进行加载。因为目前的钱包地址创建后，程序结束后内存就销毁了，那么我们需要将创建好的钱包地址，存储到本地文件中，进行持久化保存。

```java
public static class Wallets implements Serializable {

        private static final long serialVersionUID = -4824448861236743729L;
        private Map<String, Wallet> walletMap = Maps.newHashMap();

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
```

如上，我们在 WalletUtils.java 中添加一个内部类：Wallets，用于表示钱包的集合，采用 Map 实现。对于 map 集合，一个钱包地址，对应了一个钱包对象。

```java
 /**
     * 初始化钱包文件
     */
private void initWalletFile() {
    File file = new File(WALLET_FILE);
    if (!file.exists()) {
        this.saveToDisk(new Wallets());
    } else {
        this.loadFromDisk();
    }
} 
```

然后，我们需要先定义一个文件：`Wallets.dat`，用于存储钱包集合中的数据。然后创建了一个`initWalletFile()`方法，用于初始化钱包文件。我们首先判断钱包文件是否存在，如果不存在，我们就直接创建一个空的 Wallets 钱包集合对象，存入钱包文件。

```java
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
```

再创建获取一个 createWallet()方法，用于创建钱包对象，首先加载本地文件，然后将创建好的钱包文件加入到钱包集合，重新持久化存储。

```java
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
```

接下来，在 WalletUtils.java 类中，添加方法，用于将钱包数据进行持久化存储，表示将钱包的数据保存到本地文件中。

```java
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
```

现在我们又添加了两个方法，用于在获取钱包数据和获取所有的钱包地址方法中，调用加载的方法 loadFromDisk()：

```java
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
```

接下来，在`CLI.java`文件，修改 Run()，添加一个分支，用于创建钱包：

```java
func (cli *CLI) Run() {
    //判断命令行参数的长度
    this.validateArgs(args);

    CommandLineParser parser = new DefaultParser();
    CommandLine cmd = parser.parse(options, args);

    switch (args[0]) {
    ...
    case "createwallet":
        this.createWallet();
        break;
    ...
    }
} 
```

然后添加一个方法 createWallet()，并添加代码如下：

```java
/**
     * 创建钱包
     *
     * @throws Exception
     */
private void createWallet() throws Exception {
    Wallet wallet = WalletUtils.getInstance().createWallet();
    System.out.println("wallet address : " + wallet.getAddress());
} 
```

接下来，我们代码测试一下：

在终端中输入以下命令：

```java
hanru:part6_Wallet ruby$ ./blockchain.sh createwallet 
```

运行效果如下：

![`img.kongyixueyuan.com/008_002_%E5%88%9B%E5%BB%BA%E9%92%B1%E5%8C%85.png`](img/60fc87ba802d3308fb07692950ccda73.jpg)

另外，注意：你并不需要连接到一个比特币节点来获得一个地址。地址生成算法使用的多种开源算法可以通过很多编程语言和库实现。

### 4.8 打印钱包地址

因为之前的代码已经包含了获取钱包地址的方法，所以只需要在 CLI 中添加一个命令即可。

我们在`CLI.java`中，添加一个命令，用于打印出 Wallets 钱包集合中的所有的地址。

修改 run()方法代码如下：

```java
func (cli *CLI) Run() {
    //判断命令行参数的长度
    this.validateArgs(args);

    CommandLineParser parser = new DefaultParser();
    CommandLine cmd = parser.parse(options, args);

    switch (args[0]) {
    ...
    case "printaddresses":
            try {
                this.printAddresses();
            } catch (Exception e) {
                e.printStackTrace();
            }
            break;
    ...
    }
} 
```

接下来添加一个打印钱包地址的方法 printAddresses()，代码如下：

```java
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
```

最后，我们测试一下程序，在终端输入以下命令：

```java
hanru:part6_Wallet ruby$ ./blockchain.sh printaddresses 
```

运行效果如下:

![0http://img.kongyixueyuan.com/008_003_%E6%89%93%E5%8D%B0%E5%9C%B0%E5%9D%80.png](img/8e42b25d9ea8b9a6f994e0c96d03f8dc.jpg)

## 5\. 总结

通过本章节的学习，我们知道了如何创建一个钱包，钱包中如何创建一对秘钥。我们可以根据公钥生成钱包地址，这个过程虽然有点繁琐，但是不难理解。首先将公钥，进行一次 sha256，一次 ripemd160，进行 hash 散列，生成公钥 hash(也叫指纹)。再用公钥 hash 前加 1 个 byte 的版本号，一般都是 0x00，然后进行两次 sha256，获取前 4 位，作为 checksum，然后就得到了版本号+公钥 hash+checksum 的数据。最后进行一次 Base58 编码，就得到了钱包地址。

程序中的钱包地址都是存储在 map 中，为了进行持久化，我们还需要将创建好的钱包数据，保存到本地文件中。

[项目源代码](https://github.com/rubyhan1314/BitcoinForJava)