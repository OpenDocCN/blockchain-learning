# 2.2 Fabric 核心配置文件的理解

## 目标

1.  了解 Hyperledger Fabric 对 Peer 节点的核心配置信息
2.  了解 Hyperledger Fabric 对 orderer 节点的核心配置信息

## 任务实现

> 在 Hyperledger Fabric 中， 有两个示例配置文件，一个为 Peer 节点的示例配置文件，一个为 Orderer 节点的示例配置文件，理解这两个配置文件的内容对于我们而言，会更进一步的理解 Hyperledger Fabric 运行状况。

### 2.2.1 core.yaml 详解

core.yaml 配置文件是 Peer 节点的示例配置文件，具体路径在 fabric-samples/config 目录下；该 core.yaml 示例配置文件中共指定了六大部分内容，详见解释如下。

> 在 Fabirc 源码中的路径为：`$GOPATH/src/github.com/hyperledger/fabric/sampleconfig/core.yaml`

#### 2.2.1.1 日志部分：

日志记录级别有六种：`CRITICAL | ERROR | WARNING | NOTICE | INFO | DEBUG`

使用 level 指定默认所有模块为 `info` 级别，然后单独指定 cauthdsl、gossip、grpc、ledger、msp、policies、peer 的 gossip 模块的日志级别，以覆盖默认的日志级别。

format 属性指定了日志的输出格式。

```go
logging:
    level:       info    # 全局的日志级别

    # 单独模块的日志级别，覆盖全局日志级别
    cauthdsl:   warning
    gossip:     warning
    grpc:       error
    ledger:     info
    msp:        warning
    policies:   warning

    peer:
        gossip: warning

    format: '%{color}%{time:2006-01-02 15:04:05.000 MST} [%{module}] %{shortfunc} -> %{level:.4s} %{id:03x}%{color:reset} %{message}' 
```

#### 2.2.1.2 peer 部分：

peer 部分是 Peer 服务的核心配置内容，包括 Peer 的基础服务部分、gossip 部分、event、tls、BCCSP 等相关配置信息

```go
peer:
    id: jdoe        # 指定节点 ID
    networkId: dev     # 指定网络 ID
    listenAddress: 0.0.0.0:7051        #侦听本地网络接口上的地址。默认监听所有网络接口

    #侦听入站链码连接的端点。如果被注释，则选择侦听地址端口 7052 的对等点地址
    # chaincodeListenAddress: 0.0.0.0:7052    
    # 此 peer 的链码端点用于连接到 peer。如果没有指定，则选择 chaincodeListenAddress 地址。
    # 如果没有指定 chaincodeListenAddress，则从其中选择 address 
    # chaincodeAddress: 0.0.0.0:705

    address: 0.0.0.0:7051    # 节点对外的服务地址
    addressAutoDetect: false    # 是否自动探测对外服务地址    
    gomaxprocs: -1    # 进程数限制，－1 代表无限制

    # Peer 服务与 Client 的设置
    keepalive:
        # 指定客户机 ping 的最小间隔，如果客户端频繁发送 ping，Peer 服务器会自动断开
        minInterval: 60s    

        client:     # 客户端与 Peer 的通信设置 
            # 指定 ping Peer 节点的间隔时间，必须大于或等于 minInterval 的值        
            interval: 60s           
            timeout: 20s    # 在断开 peer 节点连接之前等待的响应时间

        deliveryClient:    # 客户端与 Orderer 节点的通信设置
            # 指定 ping orderer 节点的间隔时间，必须大于或等于 minInterval 的值
            interval: 60s             
            timeout: 20s    # 在断开 Orderer 节点连接之前等待的响应时间

    gossip:   # gossip 相关配置    
        bootstrap: 127.0.0.1:7051    # 启动后的初始节点
        useLeaderElection: true     # 是否指定使用选举方式产式 Leader
        orgLeader: false    # 是否指定当前节点为 Leader
        endpoint:      

        maxBlockCountToStore: 100  # 保存在内存中的最大区块     
        maxPropagationBurstLatency: 10ms    #消息连续推送之间的最大时间(超过则触发，转发给其它节点)
        maxPropagationBurstSize: 10    # 消息的最大存储数量，直到推送被触发 
        propagateIterations: 1    # 将消息推送到远程 Peer 节点的次数   
        propagatePeerNum: 3  # 选择推送消息到 Peer 节点的数量     
        pullInterval: 4s    # 拉取消息的时间间隔  
        pullPeerNum: 3      # 从指定数量的 Peer 节点拉取 
        requestStateInfoInterval: 4s # 确定从 Peer 节点提取状态信息消息的频率(单位:秒)             
        publishStateInfoInterval: 4s # 确定将状态信息消息推送到 Peer 节点的频率     
        stateInfoRetentionInterval:  # 状态信息的最长保存时间     
        publishCertPeriod: 10s      #  启动后包括证书的等待时间
        skipBlockVerification: false     # 是否应该跳过区块消息的验证   
        dialTimeout: 3s     # 拨号的超时时间     
        connTimeout: 2s     # 连接超时时间    
        recvBuffSize: 20    # 接收到消息的缓存区大小    
        sendBuffSize: 200    # 发送消息的缓冲区大小
        digestWaitTime: 1s  # 处理摘要数据的等待时间     
        requestWaitTime: 1500ms      # 处理 nonce 之前等待的时间   
        responseWaitTime: 2s   # 终止拉取数据处理的等待时间    
        aliveTimeInterval: 5s      # 心跳检查间隔时间  
        aliveExpirationTimeout: 25s    # 心跳消息的超时时间    
        reconnectInterval: 25s       # 重新连接的间隔时间 
        externalEndpoint:    # 组织外的端点

        election:   # 选举 Leader 配置     
            startupGracePeriod: 15s       # 最长等待时间 
            membershipSampleInterval: 1s  # 检查稳定性的间隔时间     
            leaderAliveThreshold: 10s     # 进行选举的间隔时间
            leaderElectionDuration: 5s    # 声明自己为 Leader 的等待时间

        pvtData:    # 私有数据配置
            # 尝试从 peer 节点中提取给定块对应的私有数据的最大持续时间
            pullRetryThreshold: 60s    
            # 当前分类帐在提交时的高度之间的最大差异
            transientstoreMaxBlockRetention: 1000            
            pushAckTimeout: 3s   # 等待每个对等方确认的最大时间         
            # 用作缓冲器；防止 peer 试图获取私有数据来自即将在接下来的 N 个块中被清除的对等节点
            btlPullMargin: 10    

    events:    
        address: 0.0.0.0:7053    # 指定事件服务的地址
        buffersize: 100    # 可以在不阻塞发送的情况下缓冲的事件总数
        # 将事件添加到一个完整的缓冲区时要阻塞多长时间
        # 如果小于 0，直接丢弃
        # 如果等于 0，事件被添加至缓冲区并发出
        # 如果大于 0，超时还未发出则丢弃
        timeout: 10ms    
        # 在注册事件中指定的时间和客户端时间之间的差异
        timewindow: 15m    

        keepalive: # peer 服务器与客户端的实时设置           
            minInterval: 60s    # 允许客户端向 peer 服务器发送 ping 的最小间隔时间

        sendTimeout: 60s    # GRPC 向客户端发送事件的超时时间

    tls:    # TLS 设置 
        enabled:  false   # 是否开启服务器端 TLS    
        # 是否需要客户端证书（没有配置使用证书的客户端不能连接到对等点）
        clientAuthRequired: false    

        cert:    # TLS 服务器的 X.509 证书
            file: tls/server.crt

        key:    # TLS 服务器(需启用 clientAuthEnabled 的客户端)的签名私钥
            file: tls/server.key

        rootcert:    # 可信任的根 CA 证书
            file: tls/ca.crt

        clientRootCAs:    # 用于验证客户端证书的根证书
            files:
              - tls/ca.crt

        clientKey:    # 建立客户端连接时用于 TLS 的私钥。如果没有设置将使用 peer.tls.key
            file:

        clientCert:    # 建立客户端连接时用于 TLS 的证书。如果没有设置将使用 peer.tls.cert
            file:

    authentication:    # 与身份验证相关的配置
        timewindow: 15m    # 当前服务器时间与客户端请求消息中指定的客户端时间差异

    fileSystemPath: /var/hyperledger/production    # 文件存储路径

    BCCSP:    # 区块链加密实现
        Default: SW        # 设置 SW 为默认加密程序  
        SW:      # SW 加密配置（如果默认为 SW）       
            Hash: SHA2        # 默认的哈希算法和安全级别
            Security: 256    # 

            FileKeyStore:    # 密钥存储位置
                # 如果为空，默认为'mspConfigPath/keystore'
                KeyStore:

        PKCS11:  # PKCS11 加密配置（如果默认为 PKCS11）
            Library:    # PKCS11 模块库位置           
            Label:    # 令牌 Label

            Pin:
            Hash:
            Security:
            FileKeyStore:
                KeyStore:

    # MSP 配置路径，peer 根据此路径找到 MSP 本地配置
    mspConfigPath: msp
    localMspId: SampleOrg    #本地 MSP 的标识符

    client:    # CLI 客户端配置选项
        connTimeout: 3s    # 连接超时

    deliveryclient:    # 订购服务相关的配置        
        reconnectTotalTimeThreshold: 3600s    # 尝试重新连接的总时间
        connTimeout: 3s    # 订购服务节点连接超时
        reConnectBackoffThreshold: 3600s    # 最大延迟时间

    localMspType: bccsp    # 本地 MSP 类型（默认情况下，是 bccsp 类型）

    # 仅在非生产环境中与 Go 分析工具一起使用。在生产中，它应该被禁用
    profile:
        enabled:     false
        listenAddress: 0.0.0.0:6060

    # 用于管理操作，如控制日志模块的严重程度等。只有对等管理员才能使用该服务
    adminService:

    # 定义处理程序可以过滤和自定义处理程序在对等点内传递的对象
    handlers:
        authFilters:
          -
            name: DefaultAuth
          -
            name: ExpirationCheck   
        decorators:
          -
            name: DefaultDecorator
        endorsers:
          escc:
            name: DefaultEndorsement
            library:
        validators:
          vscc:
            name: DefaultValidation
            library:

    # 并行执行事务验证的 goroutines 的数量（注意重写此值可能会对性能产生负面影响）
    validatorPoolSize:

    # 客户端使用发现服务查询关于对等点的信息
    # 例如——哪些同行加入了某个频道，最新消息是什么通道配置
    # 最重要的是——给定一个链码和通道，什么可能的对等点集满足背书政策
    discovery:
        enabled: true        
        authCacheEnabled: true        
        authCacheMaxSize: 1000       
        authCachePurgeRetentionRatio: 0.75       
        orgMembersAllowedAccess: false 
```

#### 2.2.1.3 vm 部分：

对链码运行环境的配置，目前主要支持 `Docker` 容器

```go
vm:
    endpoint: unix:///var/run/docker.sock    # vm 管理系统的端点

    docker:    # 设置 docker
        tls:
            enabled: false
            ca:
                file: docker/ca.crt
            cert:
                file: docker/tls.crt
            key:
                file: docker/tls.key

        attachStdout: false    # 启用/禁用链码容器中的标准 out/err

        # 创建 docker 容器的参数
        # 使用用于集群的 ipam 和 dns-server 可以有效地创建容器设置容器的网络模式
        # 支持标准值是：“host”(默认)、“bridge”、“ipvlan”、“none”
        # Dns -供容器使用的 Dns 服务器列表
        #注:'Privileged'、'Binds'、'Links'和'PortBindings'属性不支持 Docker 主机配置，设置后将不使用
        hostConfig:
            NetworkMode: host
            Dns:
               # - 192.168.0.1
            LogConfig:
                Type: json-file
                Config:
                    max-size: "50m"
                    max-file: "5"
            Memory: 2147483648 
```

#### 2.2.1.4 chaincode 部分：

与链码相关的配置

```go
chaincode:
    id:    
        path:
        name:
    # 通用构建器环境，适用于大多数链代码类型
    builder: $(DOCKER_NS)/fabric-ccenv:latest

    # 在用户链码实例化过程中启用/禁用基本 docker 镜像的拉取
    pull: false

    golang:    # golang 的 baseos
        runtime: $(BASE_DOCKER_NS)/fabric-baseos:$(ARCH)-$(BASE_VERSION)
        dynamicLink: false    # 是否动态链接 golang 链码

    car:
        #　平台可能需要更多的扩展工具(JVM 等)。目前，只能使用 baseos
        runtime: $(BASE_DOCKER_NS)/fabric-baseos:$(ARCH)-$(BASE_VERSION)

    java:
        # 用于 Java 链代码运行时，基于 java:openjdk-8 和加法编译器的镜像
        Dockerfile:  |
            from $(DOCKER_NS)/fabric-javaenv:$(ARCH)-1.1.0

    node:  
        # js 引擎在运行时，指定的 baseimage（不是 baseos）
        runtime: $(BASE_DOCKER_NS)/fabric-baseimage:$(ARCH)-$(BASE_VERSION)

    startuptimeout: 300s    # 启动超时时间
    executetimeout: 30s        # 调用和 Init 调用的超时持续时间
    mode: net    # 指定模式（dev、net 两种）
    keepalive: 0    # Peer 和链码之间的心跳超时，值小于或等于 0 会关闭

    system:    # 系统链码白名单
        cscc: enable
        lscc: enable
        escc: enable
        vscc: enable
        qscc: enable

    systemPlugins:    # 系统链码插件:

    logging:   # 链码容器的日志部分  
      level:  info
      shim:   warning
      format: '%{color}%{time:2006-01-02 15:04:05.000 MST} [%{module}] %{shortfunc} -> %{level:.4s} %{id:03x}%{color:reset} %{message}' 
```

#### 2.2.1.5 ledger 部分：

分类帐的配置信息。

```go
ledger:
  blockchain:

  state: 
    stateDatabase: goleveldb    # 指定默认的状态数据库
    couchDBConfig:    # 配置 couchDB 信息
       couchDBAddress: 127.0.0.1:5984    # 监听地址
       username:
       password:
       maxRetries: 3    # 重新尝试 CouchDB 错误的次数
       maxRetriesOnStartup: 10      #　对等启动期间对 CouchDB 错误的重试次数 
       requestTimeout: 35s      #　对等启动期间对 CouchDB 错误的重试次数 
       queryLimit: 10000     #　限制每个查询返回的记录数量  
       maxBatchUpdateSize: 1000      #　限制每个 CouchDB 批量更新的记录数量

       #　值为１时将在每次提交块后对进行索引
       # 增加值可以提高 peer 和 CouchDB 的写效率，但是可能会降低查询响应时间
       warmIndexesAfterNBlocks: 1    

  history:    
    enableHistoryDatabase: true        # 是否开启历史数据库 
```

#### 2.2.1.6 metrics 部分：

```go
metrics:
    enabled: false    # 启用或禁用 metrics 服务器
    # 当启用 metrics 服务器时
    # 必须使用特定的 metrics 报告程序类型当前支持的类型:“statsd”、“prom”
    reporter: statsd
    interval: 1s    # 确定报告度量的频率

    statsdReporter:
          address: 0.0.0.0:8125    # 要连接的 statsd 服务器地址
          flushInterval: 2s    # 确定向 statsd 服务器推送指标的频率
          # 每个 push metrics 请求的最大字节数 #内部网推荐 1432，互联网推荐 512
          flushBytes: 1432    

    promReporter:
          listenAddress: 0.0.0.0:8080    # http 服务器监听地址 
```

### 2.2.2 orderer.yaml 详解

`orderer.yaml` 配置文件是 Orderer 节点的示例配置文件，具体路径在 `fabric-samples/config` 目录下；该 orderer.yaml 示例配置文件中共指定了五大部分内容，详细解释见如下内容。

> 在 Fabirc 源码中的路径为：`$GOPATH/src/github.com/hyperledger/fabric/sampleconfig/orderer.yaml`

该 `orderer.yaml` 示例配置文件中共指定了五大部分内容：

#### 2.2.2.1 General 部分：

```go
General:
    LedgerType: file    # 指定账本类型（可选 file、RAM、json 三种）
    ListenAddress: 127.0.0.1    # 监听地址
    ListenPort: 7050    # 监听端口号

    TLS:    # GRPC 服务器的 TLS 设置
        Enabled: false    # 默认不启用
        PrivateKey: tls/server.key    # 签名的私钥文件
        Certificate: tls/server.crt    # 证书文件
        RootCAs:    # 可信任的根 CA 证书
          - tls/ca.crt
        ClientAuthRequired: false
        ClientRootCAs:

    Keepalive:    # GRPC 服务器的激活设置
        ServerMinInterval: 60s    # 客户机 ping 之间的最小允许时间
        ServerInterval: 7200s    # 连接到客户机的 ping 之间的时间
        ServerTimeout: 20s    # 服务器等待响应的超时时间

    LogLevel: info

    LogFormat: '%{color}%{time:2006-01-02 15:04:05.000 MST} [%{module}] %{shortfunc} -> %{level:.4s} %{id:03x}%{color:reset} %{message}'

    GenesisMethod: provisional    # 生成初始区块的提供方式（可选 provisional、file 两种）
    GenesisProfile: SampleInsecureSolo    # 用于动态生成初始区块的概要
    GenesisFile: genesisblock    # 生成初始区块的配置文件 

    LocalMSPDir: msp    # 本地 MSP 目录 
    LocalMSPID: SampleOrg    # MSP ID

    Profile:    # 是否为 Go“profiling”配置启用 HTTP 服务
        Enabled: false
        Address: 0.0.0.0:6060

    BCCSP:    # 区块链加密实现
        Default: SW    # 默认使用 SW
        SW:
            Hash: SHA2
            Security: 256
            FileKeyStore:
                KeyStore:

    Authentication:    # 与身份验证相关的配置
        TimeWindow: 15m    # # 当前服务器时间与客户端请求消息中指定的客户端时间差异 
```

#### 2.2.2.2 FileLedger 部分：

文件账本配置信息

```go
FileLedger:
    Location: /var/hyperledger/production/orderer    # 区块存储路径
    Prefix: hyperledger-fabric-ordererledger    # 在临时空间中生成分类目录时使用的前缀 
```

#### 2.2.2.3 RAMLedger 部分：

内存账本配置信息

```go
RAMLedger:
    HistorySize: 1000    # 如果设置为保存在内存中，分类帐最大保留的块的数量 
```

#### 2.2.2.4 Kafka 部分：

Kafka 集群的配置信息

```go
Kafka:
    Retry:    # 无法建立到 Kafka 集群的连接时的重试请求    
        ShortInterval: 5s    # 重试时间间隔
        ShortTotal: 10m        # 重试的总时间
        LongInterval: 5m    # 重试失败后再次发送重试的时间间隔
        LongTotal: 12h        # 重试的最长总时间
        NetworkTimeouts:    # 网络超时设置
            DialTimeout: 10s
            ReadTimeout: 10s
            WriteTimeout: 10s
        Metadata:    # 请求领导人选举时影响元数据的设置
            RetryBackoff: 250ms    # 指定重试的最大时间
            RetryMax: 3    # 重试的最大次数
        Producer:    # 向 Kafka 集群发送消息失败的设置
            RetryBackoff: 100ms    # 指定重试的最大时间
            RetryMax: 3    # 重试的最大次数
        Consumer:    # 向 Kafka 集群读取消息失败的设置
            RetryBackoff: 2s    # 指定重试的最大时间

    Verbose: false    # 是否为与 Kafka 集群的交互启用日志记录

    TLS:    # Orderer 连接到 Kafka 集群的 TLS 设置
      Enabled: false    # 连接到 Kafka 集群时是否使用 TLS
      PrivateKey:      
      Certificate:       
      RootCAs:

    Version:    # Kafka 版本(未指定默认值为 0.10.2.0) 
```

#### 2.2.2.5 Debug 部分：

调试配置信息

```go
Debug:
    BroadcastTraceDir:    # 对广播服务的每个请求写入此目录中的文件
    DeliverTraceDir:    # 对交付服务的每个请求写入此目录中的文件 
```

## FAQ

1.  这些配置文件的内容需要全部都记下吗？

    不需要死记硬背，重要的是理解这些配置信息都指定的什么重要内容。

    ​