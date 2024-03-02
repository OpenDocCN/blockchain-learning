# 第四章 Hypereldger Fabric（V1.2）源码深度解析－BCCSP 的 SW 实现方式

### SW 实现

SW（SoftWare）是 BCCSP 的默认实现方式，我们可以从 hyperledger/fabric/bccsp/factory/opts.go 文件中的 GetDefaultOpts 函数中看出，该函数源码如下：

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

SW 所有实现被定义在 hyperledger/fabric/bccsp/sw 包中，目录结构如下：

```go
# 解析者：Hanxiaodong
# QQ 群（专业 Fabric 交流群）：862733552
hyperledger/fabric/bccsp/sw
├── aes.go    # AES 方式加密与解密的实现
├── aeskey.go    # AES 类型的 Key 接口实现
├── conf.go    # BCCSP 的 SW 实现的配置定义，主要设置 SHA2 或 SHA3 对应的 SecurityLevel
├── dummyks.go    # dummy 类型的 KeyStore 接口实现；临时性的 Key，保存在内存中
├── ecdsa.go    # ECDSA 类型的签名与验签实现
├── ecdsakey.go    # ECDSA 类型的 Key 接口实现
├── fileks.go    # file 类型的 KeyStore 接口实现，即 fileBasedKeyStore；非临时性的 Key，保存在文件中
├── hash.go    #Hash 接口实现，即 hasher
├── impl.go    # 提供基于 BCCSP 接口的通用实现    
├── internals.go    # 加密、解密，签名、验签的接口定义
├── keyderiv.go    # KeyDeriver 接口实现
├── keygen.go    # KeyGenerator 接口实现
├── keyimport.go    # KeyImporter 接口实现
├── new.go    # 创建一个安全级别为 256 的基于软件的 BCCSP 的新实例，哈希族 SHA2，并使用指定的 KeyStore 作为密钥库
├── rsa.go    # RSA 类型的签名、验签实现
├── rsakey.go    # RSA 类型的 Key 接口实现 
```

### SW 接口实现

#### CSP 实例创建

BCCSP 的接口实现被定义在 hyperledger/fabric/bccsp/sw/impl.go 源文件中，结构体定义如下：

```go
type CSP struct {
    ks bccsp.KeyStore

    keyGenerators map[reflect.Type]KeyGenerator
    keyDerivers   map[reflect.Type]KeyDeriver
    keyImporters  map[reflect.Type]KeyImporter
    encryptors    map[reflect.Type]Encryptor
    decryptors    map[reflect.Type]Decryptor
    signers       map[reflect.Type]Signer
    verifiers     map[reflect.Type]Verifier
    hashers       map[reflect.Type]Hasher
} 
```

CSP 提供了基于包装器的 BCCSP 接口的通用实现。可以通过为以下基于算法的包装器提供实现来定制：KeyGenerator、KeyDeriver、KeyImporter、Encryptor、Decryptor、Signer、Verifier、Hasher。每个包装器都绑定到表示选项或 key 的 goland 类型。在 impl.go 源文件中实现了一个 New 函数，用于创建 CSP 实例，源码如下：

```go
func New(keyStore bccsp.KeyStore) (*CSP, error) {
    if keyStore == nil {
        return nil, errors.Errorf("Invalid bccsp.KeyStore instance. It must be different from nil.")
    }

    encryptors := make(map[reflect.Type]Encryptor)
    decryptors := make(map[reflect.Type]Decryptor)
    signers := make(map[reflect.Type]Signer)
    verifiers := make(map[reflect.Type]Verifier)
    hashers := make(map[reflect.Type]Hasher)
    keyGenerators := make(map[reflect.Type]KeyGenerator)
    keyDerivers := make(map[reflect.Type]KeyDeriver)
    keyImporters := make(map[reflect.Type]KeyImporter)

    csp := &CSP{keyStore,
        keyGenerators, keyDerivers, keyImporters, encryptors,
        decryptors, signers, verifiers, hashers}

    return csp, nil
} 
```

#### 接口实现

有了 CSP 实例，就可以进行密钥的生成、派生、导入、获取、签名、验签、加密、解密等一系列的操作。具体分别由下面的对应函数实现：

```go
// 解析者：Hanxiaodong
// QQ 群（专业 Fabric 交流群）：862733552
// KeyGen 函数主要是负责生成 Key
func (csp *CSP) KeyGen(opts bccsp.KeyGenOpts) (k bccsp.Key, err error) {
    // 验证 opts
    if opts == nil {
        return nil, errors.New("Invalid Opts parameter. It must not be nil.")
    }
    // 根据指定的 opes 获取 KeyGenerator
    keyGenerator, found := csp.keyGenerators[reflect.TypeOf(opts)]
    if !found {
        return nil, errors.Errorf("Unsupported 'KeyGenOpts' provided [%v]", opts)
    }
    // 调用 KeyGen 生成 Key
    k, err = keyGenerator.KeyGen(opts)
    if err != nil {
        return nil, errors.Wrapf(err, "Failed generating key with opts [%v]", opts)
    }

    // 如果不是临时的，则调用 StoreKey 进行存储
    if !opts.Ephemeral() {
        // Store the key
        err = csp.ks.StoreKey(k)
        if err != nil {
            return nil, errors.Wrapf(err, "Failed storing key [%s]", opts.Algorithm())
        }
    }

    return k, nil

// 根据指定的 opts 派生 key
func (csp *CSP) KeyDeriv(k bccsp.Key, opts bccsp.KeyDerivOpts) (dk bccsp.Key, err error)
// 使用 opts 从其原始表示中导入 key
func (csp *CSP) KeyImport(raw interface{}, opts bccsp.KeyImportOpts) (k bccsp.Key, err error)
// 根据指定的 ski 获取对应的 key
func (csp *CSP) GetKey(ski []byte) (k bccsp.Key, err error)
// 根据选项 opts 对一个 msg 消息进行哈希
func (csp *CSP) Hash(msg []byte, opts bccsp.HashOpts) (digest []byte, err error)
// 根据 opts 选项获取一个 Hash
func (csp *CSP) GetHash(opts bccsp.HashOpts) (h hash.Hash, err error)
// 根据指定的 opts 使用 k 对 digest 进行签名
func (csp *CSP) Sign(k bccsp.Key, digest []byte, opts bccsp.SignerOpts) 
(signature []byte, err error)
// 根据指定的 opts 使用 k 与 digest 对 signature 验签
func (csp *CSP) Verify(k bccsp.Key, signature, digest []byte, opts bccsp.SignerOpts) (valid bool, err error)
// 根据指定的 opts 使用 k 对 plaintext 进行加密
func (csp *CSP) Encrypt(k bccsp.Key, plaintext []byte, opts bccsp.EncrypterOpts) (ciphertext []byte, err error)
// 根据指定的 opts 使用 k 对 ciphertext 进行解密
func (csp *CSP) Decrypt(k bccsp.Key, ciphertext []byte, opts bccsp.DecrypterOpts) (plaintext []byte, err error)
// 将传递的类型绑定到传递的包装器中
func (csp *CSP) AddWrapper(t reflect.Type, w interface{}) error 
```

生成 Key 主要依赖于 opts 参数，可选使用到三种技术，分别为：ECDSA、RSA、AES。主要在 hyperledger/fabric/bccsp/sw/new.go 源文件中调用 NewWithParams 函数实现：

```go
// 返回基于软件的 BCCSP 的新实例，该实例设置为已通过的安全级别、散列族和密钥存储
func NewWithParams(securityLevel int, hashFamily string, keyStore bccsp.KeyStore) (bccsp.BCCSP, error) {
    // 初始化配置
    conf := &config{}
    err := conf.setSecurityLevel(securityLevel, hashFamily)
    if err != nil {
        return nil, errors.Wrapf(err, "Failed initializing configuration at [%v,%v]", securityLevel, hashFamily)
    }
    // 创建一个 CSP 实例
    swbccsp, err := New(keyStore)
    if err != nil {
        return nil, err
    }

    // 设置加密
    swbccsp.AddWrapper(reflect.TypeOf(&aesPrivateKey{}), &aescbcpkcs7Encryptor{})

    // 设置解密
    swbccsp.AddWrapper(reflect.TypeOf(&aesPrivateKey{}), &aescbcpkcs7Decryptor{})

    // 设置签名者
    swbccsp.AddWrapper(reflect.TypeOf(&ecdsaPrivateKey{}), &ecdsaSigner{})
    swbccsp.AddWrapper(reflect.TypeOf(&rsaPrivateKey{}), &rsaSigner{})

    // 设置验证
    swbccsp.AddWrapper(reflect.TypeOf(&ecdsaPrivateKey{}), &ecdsaPrivateKeyVerifier{})
    swbccsp.AddWrapper(reflect.TypeOf(&ecdsaPublicKey{}), &ecdsaPublicKeyKeyVerifier{})
    swbccsp.AddWrapper(reflect.TypeOf(&rsaPrivateKey{}), &rsaPrivateKeyVerifier{})
    swbccsp.AddWrapper(reflect.TypeOf(&rsaPublicKey{}), &rsaPublicKeyKeyVerifier{})

    // 设置 hashers
    swbccsp.AddWrapper(reflect.TypeOf(&bccsp.SHAOpts{}), &hasher{hash: conf.hashFunction})
    swbccsp.AddWrapper(reflect.TypeOf(&bccsp.SHA256Opts{}), &hasher{hash: sha256.New})
    swbccsp.AddWrapper(reflect.TypeOf(&bccsp.SHA384Opts{}), &hasher{hash: sha512.New384})
    swbccsp.AddWrapper(reflect.TypeOf(&bccsp.SHA3_256Opts{}), &hasher{hash: sha3.New256})
    swbccsp.AddWrapper(reflect.TypeOf(&bccsp.SHA3_384Opts{}), &hasher{hash: sha3.New384})

    // 设置 Key 生成器
    swbccsp.AddWrapper(reflect.TypeOf(&bccsp.ECDSAKeyGenOpts{}), &ecdsaKeyGenerator{curve: conf.ellipticCurve})
    swbccsp.AddWrapper(reflect.TypeOf(&bccsp.ECDSAP256KeyGenOpts{}), &ecdsaKeyGenerator{curve: elliptic.P256()})
    swbccsp.AddWrapper(reflect.TypeOf(&bccsp.ECDSAP384KeyGenOpts{}), &ecdsaKeyGenerator{curve: elliptic.P384()})
    swbccsp.AddWrapper(reflect.TypeOf(&bccsp.AESKeyGenOpts{}), &aesKeyGenerator{length: conf.aesBitLength})
    swbccsp.AddWrapper(reflect.TypeOf(&bccsp.AES256KeyGenOpts{}), &aesKeyGenerator{length: 32})
    swbccsp.AddWrapper(reflect.TypeOf(&bccsp.AES192KeyGenOpts{}), &aesKeyGenerator{length: 24})
    swbccsp.AddWrapper(reflect.TypeOf(&bccsp.AES128KeyGenOpts{}), &aesKeyGenerator{length: 16})
    swbccsp.AddWrapper(reflect.TypeOf(&bccsp.RSAKeyGenOpts{}), &rsaKeyGenerator{length: conf.rsaBitLength})
    swbccsp.AddWrapper(reflect.TypeOf(&bccsp.RSA1024KeyGenOpts{}), &rsaKeyGenerator{length: 1024})
    swbccsp.AddWrapper(reflect.TypeOf(&bccsp.RSA2048KeyGenOpts{}), &rsaKeyGenerator{length: 2048})
    swbccsp.AddWrapper(reflect.TypeOf(&bccsp.RSA3072KeyGenOpts{}), &rsaKeyGenerator{length: 3072})
    swbccsp.AddWrapper(reflect.TypeOf(&bccsp.RSA4096KeyGenOpts{}), &rsaKeyGenerator{length: 4096})

    // 设置 Key 生成器
    swbccsp.AddWrapper(reflect.TypeOf(&ecdsaPrivateKey{}), &ecdsaPrivateKeyKeyDeriver{})
    swbccsp.AddWrapper(reflect.TypeOf(&ecdsaPublicKey{}), &ecdsaPublicKeyKeyDeriver{})
    swbccsp.AddWrapper(reflect.TypeOf(&aesPrivateKey{}), &aesPrivateKeyKeyDeriver{conf: conf})

    // 设置 Key 导入器
    swbccsp.AddWrapper(reflect.TypeOf(&bccsp.AES256ImportKeyOpts{}), &aes256ImportKeyOptsKeyImporter{})
    swbccsp.AddWrapper(reflect.TypeOf(&bccsp.HMACImportKeyOpts{}), &hmacImportKeyOptsKeyImporter{})
    swbccsp.AddWrapper(reflect.TypeOf(&bccsp.ECDSAPKIXPublicKeyImportOpts{}), &ecdsaPKIXPublicKeyImportOptsKeyImporter{})
    swbccsp.AddWrapper(reflect.TypeOf(&bccsp.ECDSAPrivateKeyImportOpts{}), &ecdsaPrivateKeyImportOptsKeyImporter{})
    swbccsp.AddWrapper(reflect.TypeOf(&bccsp.ECDSAGoPublicKeyImportOpts{}), &ecdsaGoPublicKeyImportOptsKeyImporter{})
    swbccsp.AddWrapper(reflect.TypeOf(&bccsp.RSAGoPublicKeyImportOpts{}), &rsaGoPublicKeyImportOptsKeyImporter{})
    swbccsp.AddWrapper(reflect.TypeOf(&bccsp.X509PublicKeyImportOpts{}), &x509PublicKeyImportOptsKeyImporter{bccsp: swbccsp})

    return swbccsp, nil
} 
```

### Generators Key

生成 Key 调用了 hyperledger/fabric/bccsp/sw/keygen.go 源码文件中相应的函数来实现具体的功能。keygen.go 文件源码如下所示：

```go
type ecdsaKeyGenerator struct {
    curve elliptic.Curve
}
// 使用 ECDSA 技术生成 Key，通过调用 Go 源码中 crypto/ecdsa/ecdsa.go 库中的 GenerateKey 函数来实现，返回 hyperledger/fabric/bccsp/sw/ecdsakey.go/ecdsaPrivateKey 对象
func (kg *ecdsaKeyGenerator) KeyGen(opts bccsp.KeyGenOpts) (k bccsp.Key, err error) {
    privKey, err := ecdsa.GenerateKey(kg.curve, rand.Reader)
    if err != nil {
        return nil, fmt.Errorf("Failed generating ECDSA key for [%v]: [%s]", kg.curve, err)
    }

    return &ecdsaPrivateKey{privKey}, nil
}

type aesKeyGenerator struct {
    length int
}

// 使用 AES 技术生成 Key，通过调用 hyperledger/fabric/bccsp/sw/aes.go 源文件中的 GetRandomBytes 函数来实现，返回 hyperledger/fabric/bccsp/sw/aeskey.go/aesPrivateKey 对象
func (kg *aesKeyGenerator) KeyGen(opts bccsp.KeyGenOpts) (k bccsp.Key, err error) {
    lowLevelKey, err := GetRandomBytes(int(kg.length))
    if err != nil {
        return nil, fmt.Errorf("Failed generating AES %d key [%s]", kg.length, err)
    }

    return &aesPrivateKey{lowLevelKey, false}, nil
}

type rsaKeyGenerator struct {
    length int
}

// 使用 RSA 技术生成 Key，通过调用 Go 源码中 crypto/rsa/rsa.go 库中的 GenerateKey 函数来实现，返回 hyperledger/fabric/bccsp/sw/rsakey.go/rsaPrivateKey 对象
func (kg *rsaKeyGenerator) KeyGen(opts bccsp.KeyGenOpts) (k bccsp.Key, err error) {
    lowLevelKey, err := rsa.GenerateKey(rand.Reader, int(kg.length))

    if err != nil {
        return nil, fmt.Errorf("Failed generating RSA %d key [%s]", kg.length, err)
    }

    return &rsaPrivateKey{lowLevelKey}, nil
} 
```

### Deriv Key

派生 Key， 也就是将原有 Key 打乱之后重新生成一个 Key。主要实现有两种方式：ecdsaPrivateKey/ecdsaPublicKey、aesPrivateKey， 源码实现在 hyperledger/fabric/bccsp/sw/keyderiv.go 文件中。

### Signer & Verifier

签名与验证主要使用两种技术来实现：

*   ecdsa
*   rsa

签名主要是根据 opts 选项使用 k 对 digest 进行签名，分别调用对应的 Sign(k bccsp.Key, digest []byte, opts bccsp.SignerOpts) (signature []byte, err error)函数进行处理。

验证根据不同的技术实现分为 PrivateKeyVerifier、PublicKeyKeyVerifier 两种形式，分别调用不同的函数来实现，以 ECDSA 方式为例，源码如下：

```go
type ecdsaPrivateKeyVerifier struct{}

func (v *ecdsaPrivateKeyVerifier) Verify(k bccsp.Key, signature, digest []byte, opts bccsp.SignerOpts) (valid bool, err error) {
    return verifyECDSA(&(k.(*ecdsaPrivateKey).privKey.PublicKey), signature, digest, opts)
}

type ecdsaPublicKeyKeyVerifier struct{}

func (v *ecdsaPublicKeyKeyVerifier) Verify(k bccsp.Key, signature, digest []byte, opts bccsp.SignerOpts) (valid bool, err error) {
    return verifyECDSA(k.(*ecdsaPublicKey).pubKey, signature, digest, opts)
} 
```

通过如上的源码可以看出，无论是 ecdsaPrivateKeyVerifier，还是 ecdsaPublicKeyKeyVerifier，都是调用了 verifyECDSA 函数并将其对应的公钥作为参数进行传递，最终在 verifyECDSA 函数中通过调用 ecdsa.Verify 函数来实现验证的功能。

### Encrypt & Decrypt

对数据的加密与解密只有通过 AES 方式来实现。在 Hyperledger Fabric 中，AES 方式的加密采用 CBC 模式。

CBC（Cipher-block chaining）：密码分组链接模式。每个明文块先与前一个密文块进行异或后，再进行加密。在这种方法中，每个密文块都依赖于它前面的所有明文块。同时，为了保证每条消息的唯一性，在第一个块中需要使用初始化向量。

加密源码：

```go
func aesCBCEncryptWithRand(prng io.Reader, key, s []byte) ([]byte, error) {
    if len(s)%aes.BlockSize != 0 {
        return nil, errors.New("Invalid plaintext. It must be a multiple of the block size")
    }

    block, err := aes.NewCipher(key)
    if err != nil {
        return nil, err
    }

    ciphertext := make([]byte, aes.BlockSize+len(s))
    iv := ciphertext[:aes.BlockSize]
    if _, err := io.ReadFull(prng, iv); err != nil {
        return nil, err
    }

    mode := cipher.NewCBCEncrypter(block, iv)    // 返回使用给定块以密码块链接模式加密的块模式，iv 的长度必须与块的块大小相同
    mode.CryptBlocks(ciphertext[aes.BlockSize:], s)    // 加密

    return ciphertext, nil
} 
```

解密源码：

```go
func aesCBCDecrypt(key, src []byte) ([]byte, error) {
    block, err := aes.NewCipher(key)    // 创建并返回一个新的加密块
    if err != nil {
        return nil, err
    }

    if len(src) < aes.BlockSize {
        return nil, errors.New("Invalid ciphertext. It must be a multiple of the block size")
    }
    iv := src[:aes.BlockSize]    // 向量
    src = src[aes.BlockSize:]    // 实际数据

    if len(src)%aes.BlockSize != 0 {
        return nil, errors.New("Invalid ciphertext. It must be a multiple of the block size")
    }

    // 返回一个块模式，该模式使用给定块以密码块链接模式解密。iv 的长度必须与块的块大小相同，并且必须与用于加密数据的 iv 匹配
    mode := cipher.NewCBCDecrypter(block, iv)

    mode.CryptBlocks(src, src)    // 解密

    return src, nil
} 
```