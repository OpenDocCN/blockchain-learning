# 第十九章 【分布式存储系统 etcd】etcd-manage 项目——创建日志

# etcd-manage 项目——创建日志

接下来我们加入日志的操作。分为生成日志，查询日志等。

首先我们先实现日志的创建和存储。然后再进行查询。其实就是根据用户的操作，生成 json 数据，存入到本地日志文件中。

## 生成日志

我们先来创建日志的目录，以及日志文件，然后根据用户的操作，生成日志数据，并且进行存储。

我们现在 common.go 文件中，添加一个判断路径是否存在的函数：

```go
 // PathExists 判断文件或目录是否存在
func PathExists(path string) (bool, error) {
    _, err := os.Stat(path)
    if err == nil {
        return true, nil
    }
    if os.IsNotExist(err) {
        return false, nil
    }
    return false, err
} 
```

在 program 目录下新建 logger 子目录，里面创建一个 logger.go 文件：

首先定义一个 log 对象变量：

```go
// 日志对象
var (
    Log *zap.SugaredLogger
)
```

然后我们设计一个初始化日志的函数：

```go
 // InitLogger 日志初始化，用于记录操作日志
func InitLogger(logPath string, isDebug bool) (*zap.SugaredLogger, error) {
    infoLogPath := ""
    // errorLogPath := ""
    if logPath == "" {
        logRoot := common.GetRootDir() + "logs" + string(os.PathSeparator)
        if isExt, _ := common.PathExists(logRoot); isExt == false {
            os.MkdirAll(logRoot, os.ModePerm)
        }
        infoLogPath = logRoot + time.Now().Format("20060102") + ".log"
        // errorLogPath = logRoot + time.Now().Format("20060102_error") + ".log"
    } else {
        logPath = strings.TrimRight(logPath, string(os.PathSeparator))
        infoLogPath = logPath + string(os.PathSeparator) + time.Now().Format("20060102") + ".log"
        // errorLogPath = logPath + string(os.PathSeparator) + time.Now().Format("20060102_error") + ".log"
    }

    // 兼容 win 根完整路径问题
    zap.RegisterSink("winfile", func(u *url.URL) (zap.Sink, error) {
        return os.OpenFile(u.Path[1:], os.O_WRONLY|os.O_APPEND|os.O_CREATE, 0644)
    })

    cfg := &zap.Config{
        Encoding: "json",
    }
    cfg.EncoderConfig = zap.NewProductionEncoderConfig()
    atom := zap.NewAtomicLevel()
    if isDebug == true {
        atom.SetLevel(zapcore.DebugLevel)
        cfg.OutputPaths = []string{"stdout"}
        // cfg.ErrorOutputPaths = []string{"stdout"}
    } else {
        atom.SetLevel(zapcore.InfoLevel)
        if runtime.GOOS == "windows" {
            cfg.OutputPaths = []string{"winfile:///" + infoLogPath}
        } else {
            cfg.OutputPaths = []string{infoLogPath}
        }
        // cfg.ErrorOutputPaths = []string{errorLogPath}
    }
    cfg.Level = atom
    logger, err := cfg.Build()
    if err != nil {
        return nil, err
    }
    Log = logger.Sugar()
    return Log, nil
} 
```

这里首先要创建日志文件所存储的目录，如果目录存在就无需在创建了，我们想存储到项目的 logs 目录下。然后就要创建日志文件了，以产生日志的时间来命名文件。此处还兼容了 windows 系统。

首先我们在 Program.go 文件中，新建程序的时候，要日志进行初始化，先修改 New()函数：

```go
// New 创建主程序
func New() (*Program, error) {
    // 配置文件
    cfg, err := config.LoadConfig("")
    if err != nil {
        return nil, err
    }

    // 日志对象
    _, err = logger.InitLogger(cfg.LogPath, cfg.Debug)
    if err != nil {
        return nil, err
    }

    return &Program{
        cfg: cfg,
    }, nil
}
```

我们在读取配置文件后，就初始化日志对象。

到此日志目录和日志文件可以创建了，我们需要在用户的一些操作上添加保存日志数据的操作。大多数的用户操作的实现都在 v1.go 文件中， 所以我们需要逐个方法进行日志的添加：

先来实现添加日志的函数：

```go
 // 记录访问日志
func saveLog(c *gin.Context, msg string) {
    user := c.MustGet(gin.AuthUserKey).(string) // 用户
    userRole := ""                              // 角色
    userRoleIn, exists := c.Get("userRole")
    if exists == true && userRoleIn != nil {
        userRole = userRoleIn.(string)
    }
    // 存储日志
    logger.Log.Infow(msg, "user", user, "role", userRole)
}
```

首先修改 getEtcdKeyList()函数：

```go
 // 获取 etcd key 列表
func getEtcdKeyList(c *gin.Context) {
    go saveLog(c, "获取列表")

    key := c.Query("key")

    var err error
    defer func() {
        if err != nil {
            logger.Log.Errorw("获取 key 列表错误", "err", err)
            c.JSON(http.StatusBadRequest, gin.H{
                "msg": err.Error(),
            })
        }
    }()
    ...
} 
```

保存日志的功能可以通过并发同步进行，在有错误的时候显示错误信息。

然后修改 getEtcdKeyValue()函数，同上面的修改方式一样：

```go
 // 获取 key 的值
func getEtcdKeyValue(c *gin.Context) {
    go saveLog(c, "获取 key 的值")

    key := c.Query("key")
    var err error
    defer func() {
        if err != nil {
            logger.Log.Errorw("获取 key 值的值错误", "err", err)
            c.JSON(http.StatusBadRequest, gin.H{
                "msg": err.Error(),
            })
        }
    }()

    ...
}
```

修改 getEtcdMembers()函数：

```go
 // 获取服务节点
func getEtcdMembers(c *gin.Context) {
    go saveLog(c, "获取 etcd 集群信息")

    var err error
    defer func() {
        if err != nil {
            logger.Log.Errorw("获取服务节点错误", "err", err)
            c.JSON(http.StatusBadRequest, gin.H{
                "msg": err.Error(),
            })
        }
    }()

    ...
}
```

修改 delEtcdKey()函数：

```go
// 删除 key
func delEtcdKey(c *gin.Context) {
    go saveLog(c, "删除 key")

    key := c.Query("key")

    var err error
    defer func() {
        if err != nil {
            logger.Log.Errorw("删除 key 错误", "err", err)
            c.JSON(http.StatusBadRequest, gin.H{
                "msg": err.Error(),
            })
        }
    }()
    ...
} 
```

修改 saveEtcdKey()函数：

```go
 // 保存 key
func saveEtcdKey(c *gin.Context, isPut bool) {
    go saveLog(c, "保存 key")

    var err error
    defer func() {
        if err != nil {
            logger.Log.Errorw("保存 key 错误", "err", err)
            c.JSON(http.StatusBadRequest, gin.H{
                "msg": err.Error(),
            })
        }
    }()
    ...
}
```

修改 getValueToFormat()函数：

```go
 // 获取 key 前缀，下的值为指定格式 josn toml
func getValueToFormat(c *gin.Context) {
    go saveLog(c, "格式化显示 key")

    format := c.Query("format")
    key := c.Query("key")

    var err error
    defer func() {
        if err != nil {
            logger.Log.Errorw("保存 key 错误", "err", err)
            c.JSON(http.StatusBadRequest, gin.H{
                "msg": err.Error(),
            })
        }
    }()
    ...
} 
```

修改 getEtcdServerList()函数：

```go
 // 获取 etcd 服务列表
func getEtcdServerList(c *gin.Context) {
    go saveLog(c, "获取 etcd 服务列表")

    cfg := config.GetCfg()

     ...
} 
```

到此我们修改完了之前的功能，当用户操作的时候，都会存储到日志文件中。

[源代码](https://github.com/rubyhan1314/myetcd-manage)