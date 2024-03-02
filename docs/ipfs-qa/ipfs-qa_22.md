# 第二十二章 【IPFS 一问一答】IPFS 底层架构解析之对象层

# 22 IPFS 底层架构解析之对象层

IPFS 对象层即存储层。借鉴 Git 使用的存储数据结构，IPFS 同样使用了 Merkle DAG 技术，构建了一个有向无环图数据结构，用于存储对象数据，通常由 Base58 编码的散列引用。该数据结构的特性是：

*   内容可寻址：所有内容由多重哈希校验并唯一标识。
*   防止篡改：所有内容都通过哈希校验，如果数据被篡改，IPFS 网络将能检测到。
*   重复数据删除：相同内容有相同哈希，只存储一次，对索引对象特别有用。

IPFS 对象的结构是：

```go
type IPFSLink struct {
    Name string     //此 link 的别名
    Hash Multihash  //目标的加密 hash
    Size int        //目标总大小
}

type IPFSObject struct {
    links []IPFSLink    //links 数组
    data []byte         //不透明内容数据
}
```

在执行`ipfs object links <对象 hash>`，会列出该对象下的所有 IPFSLink 项。也就是 IPFSLink 组成了某个对象。

```go
$ ipfs object links QmZV47ZKbJ1DCLoHWHGhXZ4EMqsyTsZRJUsZuRwEAoiesn

QmUEHn7UnFyFMpJwkvotwuejnMX8XEkqxNsXMeDXJtiJyf 8019571 WeChatSight1.mp4
QmaFN5uLVBujrsQ6HFzToxUAbtV9z7MfvCtmLrscS9npf3 52173   WechatIMG1.jpeg
QmXd18A2gF1rDbNntsDTX48jexiytU7Wi3dT1rWcqNKjeV 71      ixde 
```

输出的结构类型就是 IPFSLink 的结构内容，表现为`<object multihash> <object size> <link name>`

## 22.1 路径

IPFS 对象可以遍历一个字符串的路径，路径与传统 UNIX 文件系统中的路径一样，Merkle DAG 的 links 使遍历变得简单，完整路径如下所示：

```go
# 格式
/ipfs/<hash-of-object>/<name-path-to-object>
# 事例
/ipfs/XLYkgq61DyaQ8Nhkcqyu7rcnSa7dSHQ16x/foo.txt
```

IPFS 对象也支持多重路径：

```go
/ipfs/<hash-of-foo>/bar/baz
/ipfs/<hash-of-bar>/baz
/ipfs/<hash-of-baz>
```

## 22.2 本地对象

IPFS 节点本地有一个本地存储库，用于硬盘存储或缓存存储数据的块。比如上传数据时，块数据会存储在该存储库的硬盘空间中，如果是从网络中下载某些数据，则块数据会缓存在存储库中。

## 22.3 对象锁定

某些对象的数据存储在存储库中是缓存的，IPFS 可以 pin 锁定，将数据永久保存到本地，同时也可以递归地锁定所有相关的派生对象，这对长期存储完整的对象文件特别有用。

## 22.4 发布对象

IPFS 是全球分布的文件系统，DHT 使用内容哈希寻址技术，使发布对象是公平的，安全的，完全分布式的。任何人都可以发布对象，只需要将对象的 Key 加入到 DHT 中，并且对象是通过 P2P 传输的方式加入进去，然后把访问路径给其他的用户。

## 22.5 对象级别的加密

IPFS 的对象可以先进行加密，形成一个新的对象，然后将该新对象进行存储，加密对象的结构如下：

```go
type EncryptedObject struct {
    Object []bytes              //已加密的原始对象数据
    Tag []bytes                 //可选择的加密标识
    type SignedObject struct {
        Object []bytes          //已签名的原始对象数据
        Signature []bytes       //HMAC 签名
        PublicKey   []multihash //多重哈希身份键值
    }
} 
```

这也是运用到区块链存储的原因。区块链先用公钥对对象进行加密，得到新的对象，然后将该对象存储到 IPFS 网络中，得到 IPFS hash 值，区块链将只存储 hash 值。获取数据时，先用 IPFS hash 值获取到加密的对象，然后用自己的私钥对对象进行解密，获取到真正的数据对象。