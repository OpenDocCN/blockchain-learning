# 第五章 Hyperledger Fabric（V1.2）源码深度解析－pkcs11 实现方式及 BCCSP 工厂

### pkcs11 实现方式

pkcs11（PKCS，Public-Key Cryptography Standards）的实现形式与 SW 的实现大体相同，只不过 pkcs11 为 fabric 支持热插拔和个人安全硬件模块提供了服务。关于 pkcs11 的内容，可以参考如下的的两份文档：

*   [`www.ibm.com/developerworks/cn/security/s-pkcs/`](https://www.ibm.com/developerworks/cn/security/s-pkcs/)
*   [`docs.oracle.com/cd/E19253-01/819-7056/6n91eac56/index.html#chapter2-9`](https://docs.oracle.com/cd/E19253-01/819-7056/6n91eac56/index.html#chapter2-9)

pkcs11 目录结构：

```go
hyperledger/fabric/bccsp/pkcs11/
├── conf.go
├── ecdsa.go
├── ecdsakey.go
├── impl.go
├── pkcs11.go 
```

目录结构解释如下：

conf.go：pkcs11 的配置定义
ecdsa.go：ECDSA 算法实现的签名与验签
ecdsakey.go：ECDSA 类型的 Key 接口，实现了 ecdsaPrivateKey、ecdsaPublicKey
impl.go：pkcs11 的实现
pkcs11.go：以 pkcs11 为基础，实现了各种功能

### pkcs11 结构

在 hyperledger/fabric/bccsp/pkcs11/impl.go 源码文件中定义了 pkcs11 的结构

```go
// 解析者：Hanxiaodong
// QQ 群（专业 Fabric 交流群）：862733552
type impl struct {
    bccsp.BCCSP    // 内嵌 BCCSP 接口

    conf *config    // conf 配置
    ks   bccsp.KeyStore    // key 存储对象，用于存储及获取 key

    ctx      *pkcs11.Ctx    // pkcs11 的 context
    sessions chan pkcs11.SessionHandle    // 会话标识符通道，默认 10（sessionCacheSize = 10）
    slot     uint    // 安全硬件外设连接插槽标识号

    lib          string    // 库所在路径
    noPrivImport bool    // 是否禁止导入私钥
    softVerify   bool    // 是否以软件方式验证签名
} 
```

impl.go 文件中有一个 New 函数，返回基于软件的 BCCSP 的新实例，该实例设置为已通过的安全级别、散列族和密钥存储，实现源码如下：

```go
func New(opts PKCS11Opts, keyStore bccsp.KeyStore) (bccsp.BCCSP, error) {
    // 初始化配置
    conf := &config{}
    // 设置 SecurityLevel，只支持 SHA2(256,384)/SHA3(256,384)
    err := conf.setSecurityLevel(opts.SecLevel, opts.HashFamily)
    if err != nil {
        return nil, errors.Wrapf(err, "Failed initializing configuration")
    }
    // 返回 BCCSP 的实例
    swCSP, err := sw.NewWithParams(opts.SecLevel, opts.HashFamily, keyStore)
    if err != nil {
        return nil, errors.Wrapf(err, "Failed initializing fallback SW BCCSP")
    }

    // 检查 KeyStore
    if keyStore == nil {
        return nil, errors.New("Invalid bccsp.KeyStore instance. It must be different from nil.")
    }

    lib := opts.Library
    pin := opts.Pin
    label := opts.Label
    // 调用 loadLib 函数
    ctx, slot, session, err := loadLib(lib, pin, label)
    if err != nil {
        return nil, errors.Wrapf(err, "Failed initializing PKCS11 library %s %s",
            lib, label)
    }

    sessions := make(chan pkcs11.SessionHandle, sessionCacheSize)
    csp := &impl{swCSP, conf, keyStore, ctx, sessions, slot, lib, opts.Sensitive, opts.SoftVerify}
    csp.returnSession(*session)
    return csp, nil
} 
```

在 New 函数中，其核心就是调用了 hyperledger/fabric/bccsp/pkcs11/pkcs11.go 中的 loadLib 函数建立与硬件安全模块的通信。loadLib 函数的实现源码如下：

```go
func loadLib(lib, pin, label string) (*pkcs11.Ctx, uint, *pkcs11.SessionHandle, error) {
    var slot uint = 0
    logger.Debugf("Loading pkcs11 library [%s]\n", lib)
    if lib == "" {
        return nil, slot, nil, fmt.Errorf("No PKCS11 library default")
    }

    ctx := pkcs11.New(lib)    // 创建一个新的 ctx 并初始化要使用的模块/库。
    if ctx == nil {
        return nil, slot, nil, fmt.Errorf("Instantiate failed [%s]", lib)
    }

    ctx.Initialize()    // 初始化
    slots, err := ctx.GetSlotList(true)        // 返回可用的插槽列表
    if err != nil {
        return nil, slot, nil, fmt.Errorf("Could not get Slot List [%s]", err)
    }
    found := false
    for _, s := range slots {
        info, err := ctx.GetTokenInfo(s)    // 获取令牌的信息
        if err != nil {
            continue
        }
        logger.Debugf("Looking for %s, found label %s\n", label, info.Label)
        if label == info.Label {
            found = true
            slot = s
            break
        }
    }
    if !found {
        return nil, slot, nil, fmt.Errorf("Could not find token with label %s", label)
    }

    var session pkcs11.SessionHandle
    for i := 0; i < 10; i++ {    // 尝试 10 次调用 ctx.OpenSession 打开一个会话 session
        session, err = ctx.OpenSession(slot, pkcs11.CKF_SERIAL_SESSION|pkcs11.CKF_RW_SESSION)
        if err != nil {
            logger.Warningf("OpenSession failed, retrying [%s]\n", err)
        } else {
            break
        }
    }
    if err != nil {
        logger.Fatalf("OpenSession [%s]\n", err)
    }
    logger.Debugf("Created new pkcs11 session %+v on slot %d\n", session, slot)

    if pin == "" {
        return nil, slot, nil, fmt.Errorf("No PIN set\n")
    }
    err = ctx.Login(session, pkcs11.CKU_USER, pin)    // 登录会话
    if err != nil {
        if err != pkcs11.Error(pkcs11.CKR_USER_ALREADY_LOGGED_IN) {
            return nil, slot, nil, fmt.Errorf("Login failed [%s]\n", err)
        }
    }

    return ctx, slot, &session, nil        // 返回 ctx, slot, session 会话
} 
```

### BCCSP 工厂

BCCSP 实例通过 Factory 来创建并提供，实现了 SW 与 pkcs11 两种实例的提供，SW 由 swfactory.go 提供，pkcs11 由 pkcs11factory.go 提供；pluginfactory.go 似乎有些问题，所以我们不需要去关注。

```go
hyperledger/fabric/bccsp/factory/
├── factory.go
├── nopkcs11.go
├── opts.go
├── pkcs11factory.go
├── pkcs11.go
├── pkcs11_test.go
├── pluginfactory.go
├── race_test.go
├── swfactory.go 
```

目录结构解释如下：

factory.go：定义了工厂接口，指定实现了默认的 BCCSP
nopkcs11.go：定义了工厂选项 FactoryOpts，初始化和获取 bccsp 实例
opts.go：定义了默认的工厂 opts
pkcs11factory.go：pkcs11 类型的 bccsp 工厂实现 PKCS11Factory
pkcs11.go：定义了 FactoryOpts
swfactory.go：sw 类型的 bccsp 工厂实现 SWFactory 及 SwOpts

### BCCSP 工厂接口

在 hyperledger/fabric/bccsp/factory/factory.go 源文件中定义了 BCCSP 工厂的接口，如下所示：

```go
type BCCSPFactory interface {

    // 返回此工厂的名称
    Name() string

    // 使用 opts 返回 BCCSP 的一个实例
    Get(opts *FactoryOpts) (bccsp.BCCSP, error)
} 
```

在该 factory.go 源文件中，还定义了两个函数：

*   GetDefault() bccsp.BCCSP ：返回一个非短暂(长期)的 BCCSP，默认为 SW。源码如下：

    ```go
    func GetDefault() bccsp.BCCSP {
        if defaultBCCSP == nil {    // 如果 defultBCCSP 为空
            logger.Warning("Before using BCCSP, please call InitFactories(). Falling back to bootBCCSP.")
            bootBCCSPInitOnce.Do(func() {
                var err error
                // 创建一个 SWFactory 对象
                f := &SWFactory{}
                // 根据给定的默认的 DefaultOpts 函数返回的 FactoryOpts 作为参数调用 hyperledger/fabric/bccsp/factory/swfactory.go 源码文件中的 Get 函数获取 BCCSP 对象
                bootBCCSP, err = f.Get(GetDefaultOpts())
                if err != nil {
                    panic("BCCSP Internal error, failed initialization with GetDefaultOpts!")
                }
            })
            return bootBCCSP
        }
        return defaultBCCSP
    } 
    ```

*   GetBCCSP(name string) (bccsp.BCCSP, error)：返回根据传入的选项创建的 BCCSP。

上面的源码中提到过在 hyperledger/fabric/bccsp/factory/swfactory.go 的 GetDefault 函数中调用了 hyperledger/fabric/bccsp/factory/opts.go 源文件中提供的一个 GetDefaultOpts 函数，返回了一个默认的工厂选项，具体源代码如下：

```go
func GetDefaultOpts() *FactoryOpts {
    return &FactoryOpts{
        ProviderName: "SW",
        SwOpts: &SwOpts{
            HashFamily: "SHA2",
            SecLevel:   256,

            Ephemeral: true,
        },
    }
} 
```

其它源文件中的代码大同小异，在此就不再赘述。