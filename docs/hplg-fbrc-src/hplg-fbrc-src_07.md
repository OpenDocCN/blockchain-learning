# 第七章 Hyperledger Fabric（V1.2）源码深度解析－configtxgen 命令解析

### 命令解析

生成组织结构及身份证书之后，需要创建初始区块配置文件、应用通道交易配置文件、锚节点更新配置文件。这些功能的实现源码被封装在 hyperledger/fabric/common/tools/configtxgen/main.go 文件中，我们先来分析 main 函数，在 main 函数中首先定义了一些带有指定名称、默认值和使用字符串的字符串标志， 作为 configtxgen 工具的可附带参数。如下所示：

```go
 var outputBlock, outputChannelCreateTx, profile, configPath, channelID, inspectBlock, inspectChannelCreateTx, outputAnchorPeersUpdate, asOrg, printOrg string

    flag.StringVar(&outputBlock, "outputBlock", "", "The path to write the genesis block to (if set)")
    flag.StringVar(&channelID, "channelID", "", "The channel ID to use in the configtx")
    flag.StringVar(&outputChannelCreateTx, "outputCreateChannelTx", "", "The path to write a channel creation configtx to (if set)")
    flag.StringVar(&profile, "profile", genesisconfig.SampleInsecureSoloProfile, "The profile from configtx.yaml to use for generation.")
    flag.StringVar(&configPath, "configPath", "", "The path containing the configuration to use (if set)")
    flag.StringVar(&inspectBlock, "inspectBlock", "", "Prints the configuration contained in the block at the specified path")
    flag.StringVar(&inspectChannelCreateTx, "inspectChannelCreateTx", "", "Prints the configuration contained in the transaction at the specified path")
    flag.StringVar(&outputAnchorPeersUpdate, "outputAnchorPeersUpdate", "", "Creates an config update to update an anchor peer (works only with the default channel creation, and only for the first update)")
    flag.StringVar(&asOrg, "asOrg", "", "Performs the config generation as a particular organization (by name), only including values in the write set that org (likely) has privilege to set")
    flag.StringVar(&printOrg, "printOrg", "", "Prints the definition of an organization as JSON. (useful for adding an org to a channel manually)")

    version := flag.Bool("version", false, "Show version information")

    flag.Parse() 
```

之后，检查有无 channelID，如果没有设置，则使用默认的“testchainid”作为 channelID 的值。

```go
if channelID == "" {
    channelID = genesisconfig.TestChainID
    logger.Warningf("Omitting the channel ID for configtxgen is deprecated.  Explicitly passing the channel ID will be required in the future, defaulting to '%s'.", channelID)
} 
```

判断是否为查看版本信息的参数，如果是则调用 printVersion()函数输出版本信息，源代码如下：

```go
// 显示版本
if *version {
    printVersion()
    os.Exit(exitCode)
} 
```

> 示例：如果在命令提示符中输入如下命令：
> 
> ```go
> $ ../bin/configtxgen -version 
> ```
> 
> 则命令执行后会在终端中输出如下的类似信息：
> 
> ```go
> configtxgen:
>  Version: 1.1.0
>  Go version: go1.9.2
>  OS/Arch: linux/amd64 
> ```

如果提供的参数不是显示版本信息，则通过调用 logging.SetLevel 函数设置日志级别为 INFO。然后调用 factory.InitFactories(nil)，指定默认的 BCCSP，默认使用“SW”

如果指定了生成的配置文件存储目录，则调用 genesisconfig.Load(profile, configPath)获取概要配置信息，并将其解封成为相应的结构体，以便于在输出文件时使用。源码如下：

```go
var profileConfig *genesisconfig.Profile
if outputBlock != "" || outputChannelCreateTx != "" || outputAnchorPeersUpdate != "" {
    if configPath != "" {
        profileConfig = genesisconfig.Load(profile, configPath)
    } else {
        profileConfig = genesisconfig.Load(profile)
    }
} 
```

> 注意：profile 的值来自于参数中实际给定的值，如：TwoOrgsOrdererGenesis 或 TwoOrgsChannel。configPath 的值同样取自于参数中实际给定的值，如果没有指定 configPath 参数，则默认为当前目录下的名为 configtx.yaml 的文件

之后通过判断 configPath 是否为空，选择加载 configtx，将 yaml 配置文件中的内容保存在定义的对应的结构体中，完成初始化工作，以便于后期将组织的定义输出为 JSON 串。如下：

```go
var topLevelConfig *genesisconfig.TopLevel
if configPath != "" {
    topLevelConfig = genesisconfig.LoadTopLevel(configPath)
} else {
    topLevelConfig = genesisconfig.LoadTopLevel()
} 
```

获取到相应的配置信息之后，根据指定的参数调用不同的函数生成不同的配置文件：

```go
// 解析者：Hanxiaodong
// QQ 群（专业 Fabric 交流群）：862733552
if outputBlock != "" {    // 生成初始区块文件
        if err := doOutputBlock(profileConfig, channelID, outputBlock); err != nil {
            logger.Fatalf("Error on outputBlock: %s", err)
        }
    }

if outputChannelCreateTx != "" {    // 生成应用通道交易配置文件
    if err := doOutputChannelCreateTx(profileConfig, channelID, outputChannelCreateTx); err != nil {
        logger.Fatalf("Error on outputChannelCreateTx: %s", err)
    }
}

if inspectBlock != "" {    // 检查初始区块文件内容
    if err := doInspectBlock(inspectBlock); err != nil {
        logger.Fatalf("Error on inspectBlock: %s", err)
    }
}

if inspectChannelCreateTx != "" {    // 检查应用通道交易配置文件内容
    if err := doInspectChannelCreateTx(inspectChannelCreateTx); err != nil {
        logger.Fatalf("Error on inspectChannelCreateTx: %s", err)
    }
}

if outputAnchorPeersUpdate != "" {    // 生成锚节点更新配置文件
    if err := doOutputAnchorPeersUpdate(profileConfig, channelID, outputAnchorPeersUpdate, asOrg); err != nil {
        logger.Fatalf("Error on inspectChannelCreateTx: %s", err)
    }
}

if printOrg != "" {
    if err := doPrintOrg(topLevelConfig, printOrg); err != nil {
        logger.Fatalf("Error on printOrg: %s", err)
    }
} 
```