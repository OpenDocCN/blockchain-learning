# 第六章 CLI (Command Line Interface) 命令行界面

# 第六章 CLI(Command Line Interface)

到目前为止，我们的实现还没有提供一个与程序交互的接口：目前只是在 `main` 函数中简单执行了 `CreateBlockChainWithGenesisBlock()` 和 `AddBlockToBlockChain()` 。是时候改变了！

## 1\. 课程目标

1.  了解什么是 CLI

2.  学会使用 flag 包的语法

3.  学会在项目中添加 cli 命令

## 2\. 项目代码及效果展示

### 2.1 项目代码结构

![`img.kongyixueyuan.com/006001_%E9%A1%B9%E7%9B%AE%E7%BB%93%E6%9E%84.png`](img/5bc3e12f0b9d597833a2be5e86425435.jpg)

### 2.2 项目运行结果

项目打包：

![`img.kongyixueyuan.com/006010_%E9%A1%B9%E7%9B%AE%E6%89%93%E5%8C%85.gif`](img/5c488e2ffc616edc2f6ca000f995bf0a.jpg)

通过 jar 包直接运行：

![`img.kongyixueyuan.com/006008_jar%E8%BF%90%E8%A1%8C.gif`](img/d9caede3e635464aa68ae7a0be0b2d58.jpg)

也可以编写一个 sh 脚本文件，运行程序：

![`img.kongyixueyuan.com/006009_sh%E8%84%9A%E6%9C%AC%E8%BF%90%E8%A1%8C.gif`](img/fdba08f8c7097cacf724b155452e904b.jpg)

## 3\. 创建项目

### 3.1 创建工程

打开 IntelliJ IDEA 的工作空间，将上一个项目代码目录`part3_Persistence`，复制为`part4_CLI`。

然后打开 IntelliJ IDEA 开发工具。

打开工程：`part4_CLI`，并删除 target 目录。然后进行以下修改：

```java
step1：先将项目重新命名为：part4_CLI。
step2：修改 pom.xml 配置文件。
    改为：<artifactId>part4_CLI</artifactId>标签
    改为：<name>part4_CLI Maven Webapp</name> 
```

> 说明：我们每一章节的项目代码，都是在上一个章节上进行添加。所以拷贝上一次的项目代码，然后进行新内容的添加或修改。

### 3.2 代码实现

#### 3.2.1 创建 java 文件：`CLI.java`

新建 cldy.hanru.blockchain.cli 包，并新建 CLI.java 文件，编写代码如下:

```java
package cldy.hanru.blockchain.cli;

import cldy.hanru.blockchain.block.Block;
import cldy.hanru.blockchain.block.Blockchain;
import cldy.hanru.blockchain.pow.ProofOfWork;
import cldy.hanru.blockchain.store.RocksDBUtils;
import org.apache.commons.cli.*;

import java.text.SimpleDateFormat;
import java.util.Date;

public class CLI {
    private String[] args;
    private Options options = new Options();

    public CLI(String[] args) {
        this.args = args;

        Option helpCmd = Option.builder("h").desc("show help").build();
        options.addOption(helpCmd);

        Option data = Option.builder("data").hasArg(true).desc("add block").build();

        options.addOption(data);

    }

    /**
     * 打印帮助信息
     */
    private void help() {
        System.out.println("Usage:");
        System.out.println("  createblockchain -address ADDRESS - Create a blockchain and send genesis block reward to ADDRESS");
        System.out.println("  addblock -data DATA - Get balance of ADDRESS");
        System.out.println("  printchain - Print all the blocks of the blockchain");
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
                    createBlockchainWithGenesisBlock();
                    break;
                case "addblock":
                    String data = cmd.getOptionValue("data");
                    addBlock(data);
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
    private void createBlockchainWithGenesisBlock(){
        Blockchain.newBlockchain();
    }

    /**
     * 添加区块
     *
     * @param data
     */
    private void addBlock(String data) throws Exception {
        Blockchain blockchain = Blockchain.newBlockchain();
        blockchain.addBlock(data);
    }

    /**
     * 打印出区块链中的所有区块
     */
    private void printChain() {
        Blockchain blockchain = Blockchain.newBlockchain();
        Blockchain.BlockchainIterator iterator = blockchain.getBlockchainIterator();
        long index = 0;
        while (iterator.hashNext()) {
            Block block = iterator.next();
            System.out.println("第" + block.getHeight() + "个区块信息：");

            if (block != null){
                boolean validate = ProofOfWork.newProofOfWork(block).validate();
                System.out.println("validate = " + validate);
                System.out.println("\tprevBlockHash: " + block.getPrevBlockHash());
                System.out.println("\tData: " + block.getData());
                System.out.println("\tHash: " + block.getHash());
                SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
                String date = sdf.format(new Date(block.getTimeStamp() * 1000L));
                System.out.println("\ttimeStamp:" + date);
                System.out.println();
            }
        }
    }
} 
```

#### 3.2.2 修改`main.java`

在`main.java`中修改测试代码

```java
package cldy.hanru.blockchain;

import cldy.hanru.blockchain.block.Block;
import cldy.hanru.blockchain.block.Blockchain;
import cldy.hanru.blockchain.cli.CLI;
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

/*
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
*/
        CLI cli = new CLI(args);
        cli.run();
    }

} 
```

#### 3.2.3 修改`pom.xml`配置文件

打开`pom.xml`配置文件，并修改如下：

```java
<?xml version="1.0" encoding="UTF-8"?>

<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>cldy.hanru.blockchain</groupId>
    <artifactId>part4_CLI</artifactId>
    <version>1.0-SNAPSHOT</version>
    <packaging>jar</packaging>

    <name>part4_CLI Maven Webapp</name>
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

        <!--CLI 命令行-->
        <dependency>
            <groupId>commons-cli</groupId>
            <artifactId>commons-cli</artifactId>
            <version>1.4</version>
        </dependency>

    </dependencies>

    <build>
        <finalName>part4_CLI</finalName>
            <plugins>
                <!--
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

-->
                <plugin>
                    <groupId>org.apache.maven.plugins</groupId>
                    <artifactId>maven-assembly-plugin</artifactId>
                    <version>3.1.0</version>
                    <configuration>
                        <archive>
                            <manifest>
                                <addClasspath>true</addClasspath>
                                <classpathPrefix>lib/</classpathPrefix>
                                <mainClass>cldy.hanru.blockchain.Main</mainClass>
                            </manifest>
                        </archive>
                        <descriptorRefs>
                            <descriptorRef>jar-with-dependencies</descriptorRef>
                        </descriptorRefs>
                    </configuration>
                    <executions>
                        <execution>
                            <id>make-assembly</id>
                            <!-- this is used for inheritance merges -->
                            <phase>package</phase>
                            <!-- 指定在打包节点执行 jar 包合并操作 -->
                            <goals>
                                <goal>single</goal>
                            </goals>
                        </execution>
                    </executions>
                </plugin>

            </plugins>
    </build>

</project> 
```

#### 3.2.4 添加`blockchain.sh`脚本文件

在项目下新建一个 sh 脚本文件，命名为：`blockchain.sh`，并编写内容如下：

```java
#!/bin/bash

set -e

# Check if the jar has been built.
if [ ! -e target/part4_CLI-jar-with-dependencies.jar ]; then
  echo "Compiling blockchain project to a JAR"
  mvn package -DskipTests
fi

java -jar target/part4_CLI-jar-with-dependencies.jar "$@" 
```

## 4\. CLI 讲解

### 4.1 什么是 CLI

目前只是在 `main` 函数中简单执行了 `newBlockchain` 和 `addBlock` 。是时候改变了！现在我们想要拥有这些命令,就需要通过 cli 实现。

```java
hanru:part4_CLI ruby$ ./blockchain.sh h
hanru:part4_CLI ruby$ ./blockchain.sh createblockchain
hanru:part4_CLI ruby$ ./blockchain.sh addblock -data "send 1 BTC to hanru"
hanru:part4_CLI ruby$ ./blockchain.sh printchain 
```

**命令行界面**（英语：**command-line interface**，缩写：**CLI**）是在图形用户界面得到普及之前使用最为广泛的用户界面，它通常不支持鼠标，用户通过键盘输入指令，计算机接收到指令后，予以执行。

### 4.2 使用 CLI

要想使用 CLI 命令行工具，需要导入 org.apache.commons.cli 包，所以我们最先要做的就是修改 pom.xml 配置文件，添加依赖包

```java
 <!--CLI 命令行-->
        <dependency>
            <groupId>commons-cli</groupId>
            <artifactId>commons-cli</artifactId>
            <version>1.4</version>
        </dependency> 
```

接下来，我们创建一个包：cldy.hanru.blockchain.cli，并新建 CLI.java 文件，创建 CLI 类。所有命令行相关的操作都会通过 `CLI` 类的对象进行处理：

```java
public class CLI {
    private String[] args;
    private Options options = new Options();
} 
```

它的 “入口” 是 `run()` 函数：

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
                    createBlockchainWithGenesisBlock();
                    break;
                case "addblock":
                    String data = cmd.getOptionValue("data");
                    addBlock(data);
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

Apache Commons CLI 是开源的命令行解析工具，它可以帮助开发者快速构建启动命令，并且帮助你组织命令的参数、以及输出列表等。

CLI 分为三个过程：

*   定义阶段：在 Java 代码中定义 Optin 参数，定义参数、是否需要输入值、简单的描述等
*   解析阶段：应用程序传入参数后，CLI 进行解析
*   询问阶段：通过查询 CommandLine 询问进入到哪个程序分支中

**定义阶段**

调用的方法：

```java
public Options addOption(String opt, String longOpt, boolean hasArg, String description)
    {
        addOption(new Option(opt, longOpt, hasArg, description));
        return this;
    } 
```

> 其中 Option 的参数：
> 
> *   第一个参数：参数的简单形式
> *   第二个参数：参数的复杂形式
> *   第三个参数：是否需要额外的输入
> *   第四个参数：对参数的描述信息

我们在 CLI 的构造函数中，通过向 options 中添加 Option 对象，用于表示不同的命令行参数：

```java
public CLI(String[] args) {
        this.args = args;

        Option helpCmd = Option.builder("h").desc("show help").build();
        options.addOption(helpCmd);

        Option data = Option.builder("data").hasArg(true).desc("add block").build();

        options.addOption(data);

    } 
```

首先，我们利用终端的参数来表示 4 个命令：：`h`,`createblockchain`,`addblock`,`printchain`。 其中`addblock` ，需要接收额外的输入，所以定义 Option 接接收：

我们希望当程序运行时，命令行提示信息如下:

![`img.kongyixueyuan.com/006002_%E7%BB%88%E7%AB%AF%E5%91%BD%E4%BB%A4.png`](img/9bb4a5f825600caed0f63ab202cb18f2.jpg)

**解析阶段**

通过解析器解析参数

首先，创建一个 CommandLineParser 解析器对象，然后调用 parse()方法解析 options 和 args：

```java
 try {
    CommandLineParser parser = new DefaultParser();
     CommandLine cmd = parser.parse(options, args);
}catch(Exception e){
    //TODO xxx
} 
```

**询问阶段**

根据 commandLine 查询参数，提供服务，此处配合分支语句：

```java
 switch (args[0]) {
                case "createblockchain":
                    createBlockchainWithGenesisBlock();
                    break;
                case "addblock":
                    String data = cmd.getOptionValue("data");
                    addBlock(data);
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
```

如何终端输入的命令是 h，那么现实帮助信息：

```java
 /**
     * 打印帮助信息
     */
    private void help() {
        System.out.println("Usage:");
        System.out.println("  createblockchain -address ADDRESS - Create a blockchain and send genesis block reward to ADDRESS");
        System.out.println("  addblock -data DATA - Get balance of ADDRESS");
        System.out.println("  printchain - Print all the blocks of the blockchain");
        System.exit(0);
    } 
```

如果终端输入的命令是 createblockchain，表示创建创世区块：

```java
 /**
     * 创建创世块
     */
    private void createBlockchainWithGenesisBlock(){
        Blockchain.newBlockchain();
    } 
```

如果终端输入的命令是 addblock，表示挖掘新的区块，并添加到区块链中：

```java
 /**
     * 添加区块
     *
     * @param data
     */
    private void addBlock(String data) throws Exception {
        Blockchain blockchain = Blockchain.newBlockchain();
        blockchain.addBlock(data);
    } 
```

如果终端输入的命令是 print，表示整个区块链中的所有的区块：

```java
 /**
     * 打印出区块链中的所有区块
     */
    private void printChain() {
        Blockchain blockchain = Blockchain.newBlockchain();
        Blockchain.BlockchainIterator iterator = blockchain.getBlockchainIterator();
        long index = 0;
        while (iterator.hashNext()) {
            Block block = iterator.next();
            System.out.println("第" + block.getHeight() + "个区块信息：");

            if (block != null){
                boolean validate = ProofOfWork.newProofOfWork(block).validate();
                System.out.println("validate = " + validate);
                System.out.println("\tprevBlockHash: " + block.getPrevBlockHash());
                System.out.println("\tData: " + block.getData());
                System.out.println("\tHash: " + block.getHash());
                SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
                String date = sdf.format(new Date(block.getTimeStamp() * 1000L));
                System.out.println("\ttimeStamp:" + date);
                System.out.println();
            }
        } 
```

最后，在`main.go`中修改代码：

```java
package cldy.hanru.blockchain;

import cldy.hanru.blockchain.block.Block;
import cldy.hanru.blockchain.block.Blockchain;
import cldy.hanru.blockchain.cli.CLI;
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

        CLI cli = new CLI(args);
        cli.run();
    }

} 
```

### 4.3 终端运行程序

现在因为我们需要通过终端命令执行程序，所以不能像以前那样，直接选择右键 Run 执行程序。

首先我们需要修改 pom.xml 文件，修改配置信息，指定打包的配置信息以及程序的入口程序：

```java
 <build>
        <finalName>part4_CLI</finalName>
            <plugins>
                <!--
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

-->
                <plugin>
                    <groupId>org.apache.maven.plugins</groupId>
                    <artifactId>maven-assembly-plugin</artifactId>
                    <version>3.1.0</version>
                    <configuration>
                        <archive>
                            <manifest>
                                <addClasspath>true</addClasspath>
                                <classpathPrefix>lib/</classpathPrefix>
                                <mainClass>cldy.hanru.blockchain.Main</mainClass>
                            </manifest>
                        </archive>
                        <descriptorRefs>
                            <descriptorRef>jar-with-dependencies</descriptorRef>
                        </descriptorRefs>
                    </configuration>
                    <executions>
                        <execution>
                            <id>make-assembly</id>
                            <!-- this is used for inheritance merges -->
                            <phase>package</phase>
                            <!-- 指定在打包节点执行 jar 包合并操作 -->
                            <goals>
                                <goal>single</goal>
                            </goals>
                        </execution>
                    </executions>
                </plugin>

            </plugins>
    </build> 
```

接下来，我们需要将程序打包：

打开终端进入到项目所在的目录，输入以下命令将项目进行打包：

```java
hanru:part4_CLI ruby$ mvn package 
```

效果如下：

![`img.kongyixueyuan.com/006006_%E6%89%93%E5%8C%85%E9%A1%B9%E7%9B%AE.gif`](img/ecce2d409577fa85e67422bf99a9e9bd.jpg)

编译打包后会在项目根目录下生成一个 target 文件，里面有打包生成的 jar 文件：part4_CLI-jar-with-dependencies.jar。

然后我们可以通过 cd 命令进入 target 目录，执行 jar 文件运行程序：

```java
hanru:target ruby$ java -jar part4_CLI-jar-with-dependencies.jar h
hanru:target ruby$ java -jar part4_CLI-jar-with-dependencies.jar createblockchain 
```

运行效果如下：

![`img.kongyixueyuan.com/006007_%E8%BF%90%E8%A1%8C%E7%BB%93%E6%9E%9C.gif`](img/84c3cafaf644f8feae219ef29a9224f5.jpg)

也可以编写一个 sh 脚本文件，简化运行命令。在项目根目录下创建 sh 文件：blockchain.sh，编写脚本内容如下：

```java
#!/bin/bash

set -e

# Check if the jar has been built.
if [ ! -e target/part4_CLI-jar-with-dependencies.jar ]; then
  echo "Compiling blockchain project to a JAR"
  mvn package -DskipTests
fi

java -jar target/part4_CLI-jar-with-dependencies.jar "$@" 
```

然后再终端执行运行命令：

```java
hanru:part4_CLI ruby$ ./blockchain.sh h
hanru:part4_CLI ruby$ ./blockchain.sh createblockchain
hanru:part4_CLI ruby$ ./blockchain.sh addblock -data "send 1.5 BTC to hanru"
hanru:part4_CLI ruby$ ./blockchain.sh addblock -data "send 3 BTC to wangergou"
hanru:part4_CLI ruby$ ./blockchain.sh printchain 
```

执行程序，运行结果如下:

创建创世区块

![`img.kongyixueyuan.com/006003_%E5%88%9B%E5%BB%BA%E5%8C%BA%E5%9D%97%E9%93%BE.png`](img/c477b597f5980468b34e7f1395485157.jpg)

添加新的区块：

![`img.kongyixueyuan.com/006004_%E6%B7%BB%E5%8A%A0%E5%8C%BA%E5%9D%97.png`](img/35655d00ff0bc89eb45e0c9107bdcbef.jpg)

遍历打印区块：

![`img.kongyixueyuan.com/006005_%E6%89%93%E5%8D%B0%E5%8C%BA%E5%9D%97.png`](img/26e127e7c6db5f24ccac20c69d92e66f.jpg)

## 5\. 总结

通过本章节的学习，我们知道了什么是 CLI，并通过 CLI 命令执行程序。通过 Apache Commons CLI 包设置终端命令，通过命令配合命令参数执行对应的功能。本章节中我们并没有新增功能，项目功能目前为止还是 3 个，创建创世区块：`creategenesis`，添加新区块：`add`，以及打印区块：`print`。

[项目源代码](https://github.com/rubyhan1314/BitcoinForJava)