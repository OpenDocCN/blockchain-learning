# 第十章 Hyperledger Fabric（V1.2）源码深度解析－Orderer 服务初始化

### orderer 包目录结构：

在命令提示符中使用 docker-compose 启动网络时，会查找一个名为 docker-compose.yaml 的配置文件（默认网络启动配置文件），在该配置文件中指定了相关的节点容器，而且特别说明的是 Orderer 容器是优先启动的，所以我们可以直接打开 hyperledger/fabric/orderer 文件夹，发现有一个 main.go 的文件，打开该文件，可以发现有一个 main 函数，函数中调用了 server 包中的另一个 Main 函数用来启动服务。下面我们进入到 server.Main()函数中详细解释网络启动时都做了些什么事情。

```go
fabric/orderer
├── common
│   ├── blockcutter
│   ├── bootstrap
│   ├── broadcast
│   ├── localconfig
│   ├── metadata
│   ├── msgprocessor
│   ├── multichannel
│   ├── performance
│   └── server
├── consensus
│   ├── consensus.go
│   ├── kafka
│   └── solo
├── main.go
├── mocks
│   ├── common
│   └── util
├── README.md
└── sample_clients
    ├── broadcast_config
    ├── broadcast_msg
    └── deliver_stdout 
```

目录结构说明：

*   common：Orderer 的公共目录，包括区块切割实现、引导块、广播服务提供、配置结构信息等等。

*   consensus：Orderer 的共识实现。

*   main.go：服务启动主函数所在文件。

### Orderer 容器启动

进入 hyperledger/fabric/orderer/main.go 文件中看到如下内容：

```go
func main() {
    server.Main()    // 调用 server 包下的 Main 函数
} 
```

然后点击 Main()进入 hyperledger/fabric/orderer/common/server/main.go 中，找到 Main 函数，即进入到 Orderer 的入口点：

```go
// 流程入口点
func Main() {
    fullCmd := kingpin.MustParse(app.Parse(os.Args[1:]))

    // 如果是 version
    if fullCmd == version.FullCommand() {
        fmt.Println(metadata.GetVersionInfo())
        return
    }

    // 加载配置信息
    conf, err := localconfig.Load()
    if err != nil {
        logger.Error("failed to parse config: ", err)
        os.Exit(1)
    }
    initializeLoggingLevel(conf)
    initializeLocalMsp(conf)

    prettyPrintStruct(conf)
    Start(fullCmd, conf)
} 
```

首先使用一个第三方库 kingpin 接收命令行中的命令，对于 Orderer 而言，只有两个子命令：start 与 version，如果是 version 则输出相应的版本信息后结束。反之执行后继操作。如 conf, err := localconfig.Load()进行加载配置信息。

### 读取配置文件

在 Hyperledger Fabric 中读取配置文件信息是由 Viper 来负责的。我们可以从 hyperledger/fabric/vendor/github.com/spf13/viper/viper.go 源码中的头部注释中看到对 viper 相应的解释：Viper 是一个应用程序配置系统。它可以用多种方式配置应用程序，可以从文件系统，或远程 K/V 存储中读取并进行设置。

接下来，我们将目光移到 localconfig.Load()函数中

```go
// 解析者：Hanxiaodong
// QQ 群（专业 Fabric 交流群）：862733552
// 解析 orderer YAML 文件和环境，生成一个适合配置使用的结构，失败时返回错误。
func Load() (*TopLevel, error) {
    config := viper.New()
    coreconfig.InitViper(config, "orderer")
    config.SetEnvPrefix(Prefix)    // const Prefix = "ORDERER"
    config.AutomaticEnv()
    replacer := strings.NewReplacer(".", "_")
    config.SetEnvKeyReplacer(replacer)

    if err := config.ReadInConfig(); err != nil {
        return nil, fmt.Errorf("Error reading configuration: %s", err)
    }

    var uconf TopLevel
    // 将配置信息输出到结构体中
    if err := viperutil.EnhancedExactUnmarshal(config, &uconf); err != nil {
        return nil, fmt.Errorf("Error unmarshaling config into struct: %s", err)
    }

    uconf.completeInitialization(filepath.Dir(config.ConfigFileUsed()))
    return &uconf, nil
} 
```

在 Load 函数中，通过调用 coreconfig.InitViper(config, "orderer")函数，指定了配置文件名为 orderer.yaml，如果设置了 FABRIC_CFG_PATH 环境变量，则使用该环境变量的值；反之使用当前路径（./），判断目录/etc/hyperledger/fabric/是否存在，如果存在，则将/etc/hyperledger/fabric/设定为配置文件所在目录，InitViper 实现源如下：

```go
func InitViper(v *viper.Viper, configName string) error {
    var altPath = os.Getenv("FABRIC_CFG_PATH")
    if altPath != "" {
        // 如果用户通过 FABRIC_CFG_PATH 指定了路径，则使用此路径
        if !dirExists(altPath) {
            return fmt.Errorf("FABRIC_CFG_PATH %s does not exist", altPath)
        }

        AddConfigPath(v, altPath)
    } else {
        // 如果用户没有指定 FABRIC_CFG_PATH 环境变量，则使用 /etc/hyperledger/fabric 作为配置文件所在的目录路径
        AddConfigPath(v, "./")

        if dirExists(OfficialPath) {
            AddConfigPath(v, OfficialPath)
        }
    }

    // 设定配置文件名称为 orderer.yaml
    if v != nil {
        v.SetConfigName(configName)
    } else {
        viper.SetConfigName(configName)
    }

    return nil
} 
```

设定配置文件所在目录之后调用 config.AutomaticEnv()加载环境变量，将环境变量中的 _ 替换为 . ，之后通过调用 config.ReadInConfig()对配置文件内容进行读取。

通过 viperutil.EnhancedExactUnmarshal(config, &uconf)将配置信息解码到一个结构中检查其正确性。

最后通过调用 uconf.completeInitialization(filepath.Dir(config.ConfigFileUsed()))函数，检查各主要配置信息是否进行了设置（TopLevel 结构体对象中的各个成员），如果没有设置，则将其设定为相应的默认值，如指定分类账本的类型、账本前缀，监听地址、端口号，日志级别，本地 MSP 的 ID 及目录所在路径等等相关信息。

TopLevel 结构体定义如下：

```go
// 直接对应于 orderer 的 YAML 配置文件
type TopLevel struct {
    General    General
    FileLedger FileLedger
    RAMLedger  RAMLedger
    Kafka      Kafka
    Debug      Debug
}

// 包含在所有 Orderer 类型中的通用配置
type General struct {
    LedgerType     string
    ListenAddress  string
    ListenPort     uint16
    TLS            TLS
    Keepalive      Keepalive
    GenesisMethod  string
    GenesisProfile string
    SystemChannel  string
    GenesisFile    string
    Profile        Profile
    LogLevel       string
    LogFormat      string
    LocalMSPDir    string
    LocalMSPID     string
    BCCSP          *bccsp.FactoryOpts
    Authentication Authentication
}

// 包含 gRPC 服务器的配置
type Keepalive struct {
    ServerMinInterval time.Duration
    ServerInterval    time.Duration
    ServerTimeout     time.Duration
}

// 包含 TLS 连接的配置
type TLS struct {
    Enabled            bool
    PrivateKey         string
    Certificate        string
    RootCAs            []string
    ClientAuthRequired bool
    ClientRootCAs      []string
}

// 包含与验证客户端消息相关的配置参数
type Authentication struct {
    TimeWindow time.Duration
}

// 包含用于 Go pprof 分析的配置
type Profile struct {
    Enabled bool
    Address string
}

// 包含基于文件的分类账本的配置
type FileLedger struct {
    Location string
    Prefix   string
}

// 包含 RAM 分类帐本的配置
type RAMLedger struct {
    HistorySize uint
}

// 包含基于 kafka 的 Orderer 的配置
type Kafka struct {
    Retry   Retry
    Verbose bool
    Version sarama.KafkaVersion // TODO Move this to global config
    TLS     TLS
}

// 包含 Orderer 调试参数的配置
type Debug struct {
    BroadcastTraceDir string
    DeliverTraceDir   string
} 
```

结构体中相关各项项定义的默认值如下：

```go
// 指定 orderer 的默认配置值
var Defaults = TopLevel{
    General: General{
        LedgerType:     "file",
        ListenAddress:  "127.0.0.1",
        ListenPort:     7050,
        GenesisMethod:  "provisional",
        GenesisProfile: "SampleSingleMSPSolo",
        SystemChannel:  "test-system-channel-name",
        GenesisFile:    "genesisblock",
        Profile: Profile{
            Enabled: false,
            Address: "0.0.0.0:6060",
        },
        LogLevel:    "INFO",
        LogFormat:   "%{color}%{time:2006-01-02 15:04:05.000 MST} [%{module}] %{shortfunc} -> %{level:.4s} %{id:03x}%{color:reset} %{message}",
        LocalMSPDir: "msp",
        LocalMSPID:  "SampleOrg",
        BCCSP:       bccsp.GetDefaultOpts(),
        Authentication: Authentication{
            TimeWindow: time.Duration(15 * time.Minute),
        },
    },
    RAMLedger: RAMLedger{
        HistorySize: 10000,
    },
    FileLedger: FileLedger{
        Location: "/var/hyperledger/production/orderer",
        Prefix:   "hyperledger-fabric-ordererledger",
    },
    Kafka: Kafka{
        Retry: Retry{
            ShortInterval: 1 * time.Minute,
            ShortTotal:    10 * time.Minute,
            LongInterval:  10 * time.Minute,
            LongTotal:     12 * time.Hour,
            NetworkTimeouts: NetworkTimeouts{
                DialTimeout:  30 * time.Second,
                ReadTimeout:  30 * time.Second,
                WriteTimeout: 30 * time.Second,
            },
            Metadata: Metadata{
                RetryBackoff: 250 * time.Millisecond,
                RetryMax:     3,
            },
            Producer: Producer{
                RetryBackoff: 100 * time.Millisecond,
                RetryMax:     3,
            },
            Consumer: Consumer{
                RetryBackoff: 2 * time.Second,
            },
        },
        Verbose: false,
        Version: sarama.V0_10_2_0,
        TLS: TLS{
            Enabled: false,
        },
    },
    Debug: Debug{
        BroadcastTraceDir: "",
        DeliverTraceDir:   "",
    },
} 
```

随后使用 initializeLoggingLevel(conf)初始化日志系统。

```go
// 设置日志级别
func initializeLoggingLevel(conf *localconfig.TopLevel) {
    flogging.InitBackend(flogging.SetFormat(conf.General.LogFormat), os.Stderr)    // 初始化日志对象及输出格式
    flogging.InitFromSpec(conf.General.LogLevel)    // 初始化日志级别
} 
```