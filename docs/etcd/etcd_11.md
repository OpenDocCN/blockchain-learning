# 第十一章 【分布式存储系统 etcd】etcd-manage 项目——客户端实现和 http 服务

# etcd-manage 项目——客户端实现和 http 服务

在本文档中，我们主要实现客户端的创建和 http 服务。

## 一、静态文件处理

我们这里提供了写好的前端，如果你不需要改动界面，可以直接拿来使用，我们将 static 整体打包到了 tpls/tpls.go，我们可以直接使用。

这里使用 go-bindata 打包。

go-bindata 很简单，设计理念也不难理解。它的任务就是将静态文件封装在一个 Go 语言的 Source Code 里面，然后提供一个统一的接口，你通过这个接口传入文件路径，它将给你返回对应路径的文件数据。这也就是说它不在乎你的文件是字符型的文件还是字节型的，你自己处理，它只管包装。我们只要记住路径，然后访问 Asset()函数即可。

在 program 目录下新建 go 文件，http_ui.go：

```go
package program

import (
    "github.com/gin-gonic/gin"
    "strings"
    "myetcd-manage/tpls"
    "mime"
    "path"
    "net/http"
)

// ui 界面
// 处理静态文件
func (p *Program) handlerStatic(c *gin.Context) {
    uri := strings.TrimLeft(c.Request.RequestURI, "/")
    if uri == "ui/" || uri == "ui" {
        uri = "dist/index.html"
    } else {
        uri = strings.Replace(uri, "ui", "dist", 1)
    }
    // log.Println(uri)
    // 读取模版内容
    body, err := tpls.Asset(uri)
    if err != nil {
        c.Status(http.StatusNotFound)
        return
    }
    mimetype := mime.TypeByExtension(path.Ext(uri))
    if mimetype != "" {
        c.Header("Content-Type", mimetype)
    }

    c.Writer.Write(body)
}
```

这里我们提供了一个方法 handlerStatic()，用来处理静态文件，里面最重要的就是 tpls.Asset()。

## 二、etcd 的客户端实现

我们现在 program 目录下新建子目录 etcdv3 目录，新建一个 go 文件，etcdv3.go：

首先创建一个 Etcd3Client 结构体：

```go
// Etcd3Client etcd v3 客户端
type Etcd3Client struct {
    *clientv3.Client
}
```

再提供 etcd 连接对象：

```go
var (
    // EtcdClis etcd 连接对象
    etcdClis *sync.Map
)

func init() {
    etcdClis = new(sync.Map)

}
```

接下来我们提供一个函数，用来创建 Etcd3Client 对象：

```go
 // NewEtcdCli 创建一个 etcd 客户端
func NewEtcdCli(etcdCfg *config.EtcdServer) (*Etcd3Client, error) {
    if etcdCfg == nil {
        return nil, errors.New("etcdCfg is nil")
    }
    if etcdCfg.TLSEnable == true && etcdCfg.TLSConfig == nil {
        return nil, errors.New("TLSConfig is nil")
    }
    if len(etcdCfg.Address) == 0 {
        return nil, errors.New("Etcd connection address cannot be empty")
    }

    var cli *clientv3.Client
    var err error

    if etcdCfg.TLSEnable == true {
        // tls 配置
        tlsInfo := transport.TLSInfo{
            CertFile:      etcdCfg.TLSConfig.CertFile,
            KeyFile:       etcdCfg.TLSConfig.KeyFile,
            TrustedCAFile: etcdCfg.TLSConfig.CAFile,
        }
        tlsConfig, err := tlsInfo.ClientConfig()
        if err != nil {
            return nil, err
        }

        cli, err = clientv3.New(clientv3.Config{
            Endpoints:   etcdCfg.Address,
            DialTimeout: 10 * time.Second,
            TLS:         tlsConfig,
            Username:    etcdCfg.Username,
            Password:    etcdCfg.Password,
        })
    } else {
        cli, err = clientv3.New(clientv3.Config{
            Endpoints:   etcdCfg.Address,
            DialTimeout: 10 * time.Second,
            Username:    etcdCfg.Username,
            Password:    etcdCfg.Password,
        })
    }

    if err != nil {
        return nil, err
    }
    etcdClis.Store(etcdCfg.Name, cli)
    return &Etcd3Client{
        Client: cli,
    }, nil
} 
```

获取一个 etcd cli 对象：

```go
 // GetEtcdCli 获取一个 etcd cli 对象
func GetEtcdCli(etcdCfg *config.EtcdServer) (*Etcd3Client, error) {
    if etcdCfg == nil {
        return nil, errors.New("etcdCfg is nil")
    }
    val, ok := etcdClis.Load(etcdCfg.Name)
    if ok == false {
        if len(etcdCfg.Address) > 0 {
            cli, err := NewEtcdCli(etcdCfg)
            if err != nil {
                return nil, err
            }
            return cli, nil
        }
        return nil, errors.New("Getting etcd client error")
    }
    return &Etcd3Client{
        Client: val.(*clientv3.Client),
    }, nil
} 
```

然后我们在 config.go 中添加一个方法，用于根据 name 获取 EtcdServer 服务：

```go
// GetEtcdServer 根据服务标识查找服务
func GetEtcdServer(name string) *EtcdServer {
    if cfg == nil {
        return nil
    }
    for _, v := range cfg.Server {
        fmt.Println("name-->",v,"Name,,,",v.Name,",name:",name)
        if v.Name == name {
            fmt.Println("v-->",v)
            return v
        }
    }
    return nil
}
```

## 三、http 服务

我们现在 program 目录下新建一个子目录 v1，新建一个 go 文件 v1.go：

```go
// V1 v1 版接口
func V1(v1 *gin.RouterGroup){

}
```

我们就可以在这里注册路由，然后添加对应的功能实现方法了。

然后我们创建一个 go 文件，http.go：

```go
 // 启动 http 服务
func (p *Program) startAPI() {
    router := gin.Default()

    // 跨域问题
    router.Use(p.middleware())

    // 设置静态文件目录
    router.GET("/ui/*w", p.handlerStatic)
    router.GET("/", func(c *gin.Context) {
        c.Redirect(301, "/ui")
    })

    // 读取认证列表
    accounts := make(gin.Accounts, 0)
    if p.cfg.Users != nil {
        for _, u := range p.cfg.Users {
            accounts[u.Username] = u.Password
        }
    }

    // v1 api
    apiV1 := router.Group("/v1", gin.BasicAuth(accounts))
    apiV1.Use(p.middlewareEtcd()) // 注入 etcd 客户端
    v1.V1(apiV1)

    addr := fmt.Sprintf("%s:%d", p.cfg.HTTP.Address, p.cfg.HTTP.Port)
    // 监听
    s := &http.Server{
        Addr:         addr,
        Handler:      router,
        ReadTimeout:  10 * time.Second,
        WriteTimeout: 10 * time.Second,
    }

    log.Println("启动 HTTP 服务:", addr)
    var err error
    if p.cfg.HTTP.TLSEnable == true {
        if p.cfg.HTTP.TLSConfig == nil || p.cfg.HTTP.TLSConfig.CertFile == "" || p.cfg.HTTP.TLSConfig.KeyFile == "" {
            log.Fatalln("启用 tls 必须配置证书文件路径")
        }
        err = s.ListenAndServeTLS(p.cfg.HTTP.TLSConfig.CertFile, p.cfg.HTTP.TLSConfig.KeyFile)
    } else if p.cfg.HTTP.TLSEncryptEnable == true {
        if len(p.cfg.HTTP.TLSEncryptDomainNames) == 0 {
            log.Fatalln("域名列表不能为空")
        }
        err = autotls.Run(router, p.cfg.HTTP.TLSEncryptDomainNames...)
    } else {
        err = s.ListenAndServe()
    }
    if err != nil {
        log.Fatalln(err)
    }
} 
```

首先我们来解决一下跨域问题，这个通过中间件来实现就可以了，添加一个方法：

```go
// 跨域中间件
func (p *Program) middleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        // gin 设置响应头，设置跨域
        c.Header("Access-Control-Allow-Origin", "*")
        c.Header("Access-Control-Allow-Methods", "POST, GET, OPTIONS, PUT, DELETE")
        c.Header("Access-Control-Allow-Headers", "Origin, Content-Type, Authorization, Access-Control-Allow-Origin")
        c.Header("Access-Control-Expose-Headers", "Content-Length, Access-Control-Allow-Origin, Access-Control-Allow-Headers, Content-Type")

        //放行所有 OPTIONS 方法
        if c.Request.Method == "OPTIONS" {
            c.Status(http.StatusOK)
        }
    }
}
```

还要注册 etcd 的客户端，先提供获取 etcd 连接的函数：

```go
 // 获取 etcd 连接
func getEtcdCli(name, role string) (ctl *etcdv3.Etcd3Client, s *config.EtcdServer, err error) {
    s = config.GetEtcdServer(name)
    if s == nil {
        return nil, nil, errors.New("etcd 服务不存在")
    }
    if len(s.Roles) > 0 {
        isRole := false
        for _, r := range s.Roles {
            if role == r {
                isRole = true
                break
            }
        }
        if isRole == false {
            return nil, nil, errors.New("无权限访问")
        }
    }
    ctl, err = etcdv3.GetEtcdCli(s)
    return
}
```

然后我们再添加一个注入 etcd 客户端方法：

```go
 // etcd 客户端中间件
func (p *Program) middlewareEtcd() gin.HandlerFunc {
    return func(c *gin.Context) {
        // 获取登录用户名，查询角色
        userIn := c.MustGet(gin.AuthUserKey)
        userRole := ""
        if userIn != nil {
            user := userIn.(string)
            if user == "" {
                c.Set("userRole", "")
            } else {
                u := p.cfg.GetUserByUsername(user)
                if u == nil {
                    c.Set("userRole", "")
                } else {
                    userRole = u.Role
                    // 角色和用户信息
                    c.Set("userRole", u.Role)
                    c.Set("authUser", u)
                }
            }
        }

        // 绑定 etcd 连接
        //etcdServerName := c.GetHeader("EtcdServerName")
        etcdServerName := "cluster_run"
        fmt.Println("etcdServerName,",etcdServerName)
        //if etcdServerName ==""{
        //  etcdServerName = "default"
        //}
        if etcdServerName != "" {
            cli, s, err := getEtcdCli(etcdServerName, userRole)
            if err == nil {
                c.Set("EtcdServer", cli)
                c.Set("EtcdServerCfg", s)
            }
        }

        c.Next()

    }
}
```

这里我们还需要在 config.go 文件中添加一个函数，就是根据 username 获取 User 对象：

```go
// GetUserByUsername 根据用户名获取用户信息
func (c *Config) GetUserByUsername(username string) *User {
    if c.Users != nil && len(c.Users) > 0 {
        for _, u := range c.Users {
            if u.Username == username {
                return u
            }
        }
    }
    return nil
}
```

## 四、主程序 Program

我们在 program 目录下新建 go 文件，program.go：

先提供一个结构体，主要有配置文件对象和 http 服务即可：

```go
// Program 主程序
type Program struct {
    cfg *config.Config
    s   *http.Server
}
```

再提供一个函数用来创建一个 Program 对象：

```go
// New 创建主程序
func New() (*Program, error) {
    // 配置文件
    cfg, err := config.LoadConfig("")
    if err != nil {
        return nil, err
    }

    return &Program{
        cfg: cfg,
    }, nil
}
```

打开浏览器的函数：

```go
// 打开 url
func openURL(urlAddr string) {
    var cmd *exec.Cmd
    if runtime.GOOS == "windows" {
        cmd = exec.Command("cmd", " /c start "+urlAddr)
    } else if runtime.GOOS == "darwin" {
        cmd = exec.Command("open", urlAddr)
    } else {
        return
    }
    err := cmd.Start()
    if err != nil {
        log.Error(err)
    }
}
```

然后提供一个函数，用于运行，其中主要就是启动 http 服务，以及打开浏览器显示主页：

```go
// Run 启动程序
func (p *Program) Run() error {
    // 启动 http 服务
    go p.startAPI()

    // 打开浏览器
    go func() {
        time.Sleep(100 * time.Millisecond)
        openURL(fmt.Sprintf("http://127.0.0.1:%d/ui/", p.cfg.HTTP.Port))
    }()

    return nil
}
```

最后还得添加一个停止服务的方法：

```go
// Stop 停止服务
func (p *Program) Stop() {
    if p.s != nil {
        p.s.Close()
    }
}
```

然后我们在 main 中启动：

```go
func main() {
    // 服务对象
    p, err := program.New()
    if err != nil {
        log.Println(err)
        os.Exit(1)
    }
    err = p.Run()
    if err != nil {
        log.Println(err)
        os.Exit(1)
    }

    // 监听退出信号
    c := make(chan os.Signal)
    signal.Notify(c, os.Interrupt, os.Kill) // , syscall.SIGUSR1, syscall.SIGUSR2
    <-c
    p.Stop()
    log.Println("程序退出")

}
```

这里的 Notify 函数让 signal 包将输入信号转发到 c。如果没有列出要传递的信号，会将所有输入信号传递到 c；否则只传递列出的输入信号。

signal 包不会为了向 c 发送信息而阻塞（就是说如果发送时 c 阻塞了，signal 包会直接放弃）：调用者应该保证 c 有足够的缓存空间可以跟上期望的信号频率。对使用单一信号用于通知的通道，缓存为 1 就足够了。

[源代码](https://github.com/rubyhan1314/myetcd-manage)