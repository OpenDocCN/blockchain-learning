# 第三章 Hyperledger Fabric（V1.2）源码深度解析－BCCSP

### BCCSP 介绍

**BCCSP（blockchain cryptographic service provider）：**称之为**区域链加密服务提供者**，在 Hyperledger Fabric 中，BCCSP 的主要作用是为 Hyperledger Fabric 提供加密、签名服务，包括如下：

*   AES：一种分组加密实现算法，Hyperledger Fabric 中主要用来对数据做加密解密处理。主要包含：AES128、AES192、AES256。
*   ECDSA：一种椭圆曲线加密算法，Hyperledger Fabric 中主要用于签名与验签。主要包含：ECDSAP256、ECDSAP384。
*   RSA：一种非对称加密算法，是目前使用最广泛的公钥密码体制之一，Hyperledger Fabric 中主要用于签名与验签。主要包含：RSA1024、RSA2048、RSA3072、RSA4096。
*   HASH：哈希算法。主要包含：SHA256、SHA384、SHA3_256、SHA3_384。
*   PKCS11：一套基于硬件的标准安全接口。
*   HMAC： 密匙相关的哈希运算消息认证码

如上所有的支持的技术都被 Hyperledger Fabric 作为常量定义在 hyperledger/fabric/bccsp/opts.go 文件中：

```go
package bccsp

const (
    ECDSA = "ECDSA"
    ECDSAP256 = "ECDSAP256"
    ECDSAP384 = "ECDSAP384"
    ECDSAReRand = "ECDSA_RERAND"

    RSA = "RSA"
    RSA1024 = "RSA1024"
    RSA2048 = "RSA2048"
    RSA3072 = "RSA3072"
    RSA4096 = "RSA4096"

    AES = "AES"
    AES128 = "AES128"
    AES192 = "AES192"
    AES256 = "AES256"

    HMAC = "HMAC"
    HMACTruncated256 = "HMAC_TRUNCATED_256"

    SHA = "SHA"
    SHA2 = "SHA2"
    SHA3 = "SHA3"

    SHA256 = "SHA256"
    SHA384 = "SHA384"
    SHA3_256 = "SHA3_256"
    SHA3_384 = "SHA3_384"

    X509Certificate = "X509Certificate"
) 
```

### BCCSP 目录结构

BCCSP 服务目录结构如下：

```go
hyperledger/fabric/bccsp
├── aesopts.go
├── bccsp.go
├── bccsp_test.go
├── ecdsaopts.go
├── factory
├── hashopts.go
├── idemixopts.go
├── keystore.go
├── mocks
├── opts.go
├── pkcs11
├── rsaopts.go
├── signer
├── sw
└── utils 
```

**目录解释：**

*   factory 目录：为 BCCSP 的工厂目录，创建一个 BCCSP 实例（如 sw 或 psck11）

*   pkcs11 目录：pkcs11 的实现包

*   signer 目录：基于 BCCSP 的加密签名实现

*   sw 目录：基于软件的 BCCSP 实现包。

*   util 目录：BCCSP 工具包。

*   bccsp.go：定义了 BCCSP 及 Key 的接口，以及相应的 xxxOpts 接口。

*   keystore.go：定义了 key 的管理存储接口。

*   opts.go：定义了 BCCSP 所支持的各种技术选项。

*   xxxopts.go：BCCSP 服务所使用到的各种技术选项的实现。

### BCCSP 接口

首先来看一下 hyperledger/fabric/bccsp.go 中定义的接口及对应解释：

```go
// 解析者：Hanxiaodong
// QQ 群（专业 Fabric 交流群）：862733552
type Key interface {

    Bytes() ([]byte, error)    // 将 Key 转换为字节形式

    SKI() []byte    // 全称为 subject key identifier，此函数返回主题密钥标识符

    Symmetric() bool    // 如果为对称密钥，返回 true，否则返回 false

    Private() bool    // 如果此 Key 为私钥，返回 true，否则返回 false

    PublicKey() (Key, error)    // 返回非对称公钥/私钥对的对应公钥部分。如果为对称密钥返回错误
}

// 包含使用 CSP 生成 Key 的选项
type KeyGenOpts interface {

    Algorithm() string    // 返回要使用的密钥生成算法标识符

    Ephemeral() bool    // 如果要生成的键是临时性的，则返回 true
}

// 包含使用 CSP 派生键的选项
type KeyDerivOpts interface {

    Algorithm() string    // 返回要使用的密钥派生算法标识符

    Ephemeral() bool    // 如果派生的键是临时性的，则返回 true
}

// 包含使用 CSP 导入密钥的原材料的选项
type KeyImportOpts interface {

    Algorithm() string    // 返回密钥导入算法标识符

    Ephemeral() bool    // 如果生成的键是临时的，则返回 true
}

// 包含使用 CSP 进行哈希的选项
type HashOpts interface {

    Algorithm() string    // 返回哈希算法标识符
}

// 包含与 CSP 签名的选项
type SignerOpts interface {
    crypto.SignerOpts
}

type EncrypterOpts interface{}    // 包含使用 CSP 加密的选项

type DecrypterOpts interface{}    // 包含使用 CSP 解密的选项

// BCCSP 接口提供加密标准和算法的实现
type BCCSP interface {
    // 使用 opts 生成 Key
    KeyGen(opts KeyGenOpts) (k Key, err error)
    // 使用 opts 从派生一个 Key
    KeyDeriv(k Key, opts KeyDerivOpts) (dk Key, err error)
    // 使用 opts 从其原始表示中导入一个 Key
    KeyImport(raw interface{}, opts KeyImportOpts) (k Key, err error)
    // 返回此 CSP 关联到的 Key
    GetKey(ski []byte) (k Key, err error)
    // 根据哈希选项 opts 对一个 msg 消息进行哈希，如果 opts 为空，则使用默认选项
    Hash(msg []byte, opts HashOpts) (hash []byte, err error)
    // 根据选项 opts 获取一个 Hash 实例，如果 opts 为空，则使用默认选项
    GetHash(opts HashOpts) (h hash.Hash, err error)
    // 签名（根据 opts 选项使用 k 对 digest 进行签名）
    Sign(k Key, digest []byte, opts SignerOpts) (signature []byte, err error)
    // 验证签名（使用 opts 选项根据 k 和 digest 验证签名）
    Verify(k Key, signature, digest []byte, opts SignerOpts) (valid bool, err error)
    // 根据 opts 选项使用 k 对 plaintext 进行加密
    Encrypt(k Key, plaintext []byte, opts EncrypterOpts) (ciphertext []byte, err error)
    // 根据 opts 选项使用 k 对 ciphertext 进行解密
    Decrypt(k Key, ciphertext []byte, opts DecrypterOpts) (plaintext []byte, err error)
} 
```