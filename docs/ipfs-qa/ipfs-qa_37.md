# 第三十七章 【IPFS 一问一答】解析 IPFS 应用层 IPLD 之 CID

# 37\. 【IPFS 一问一答】解析 IPFS 应用层 IPLD 之 CID

## 37.1 为什么需要 CID？

CID 是类似 IPFS 分布式文件系统中标准的文件寻址格式。它集合了内容寻址、加密散列算法和自我描述的格式。它是 IPFS 和 IPLD 的内部重要的识别符。

![](img/8473008eb63db180ddfff992c0538934.jpg)

CID 相关讨论可参照：[`github.com/ipfs/specs/issues/130(第一个 post 见这里`](https://github.com/ipfs/specs/issues/130(第一个 post 见这里))

## 37.2 协议描述

CID 是一种自描述式的内容寻址的识别符。它必须使用加密散列函数来得到内容的地址。它使用了很多 multiformats 来实现灵活的自描述，即使用 multihash 得到哈希值，multicodec-packed 用于描述内容类型，通过 multibase 将 CID 本身编码成字符串。

![](img/dc981ed7612c92eebf71b93f8b5eea65.jpg)

当前版本: CIDv1

一个 CIDv1 由四部分组成:

```go
<cidv1> ::= <mb><version><mcp><mh>
# or, expanded:
<cidv1> ::= <multibase-prefix><cid-version><multicodec-packed-content-type><multihash-content-address>
```

其中：

*   `<multibase-prefix>` 是一个 multibase 编码 (1 到 2 个字节), 便于将 CID 编码成不同的格式。
*   `<cid-version>` 是一个表示 CID 版本的变量，为了便于今后升级。
*   `<multicodec-packed-content-type>` 是一种用 multicodec-packed 编码表示内容的类型或者数据的格式。
*   `<multihash-content-address>` 是一个 multihash 值, 表示了内容的加密哈希散列值。Multihash 让 CID 可以使用不同的加密哈希散列函数，便于今后的升级和改造。

## 37.3 设计理念

CIDs 在设计的时候考虑到了构建 IPFS 时遇到的各种权衡方案。这与众多支持多格式的项目有关。

压缩性：CID 二进制的特性让其压缩效率非常高，这也让 CID 可以作为 URL 的一部分。 传输友好性：即“易复制性，copy-pastability”。CID 以 multibase 编码来方便传输，例如，以 base58btc 编码的 CID 的长度将更短，而且便于哈希值的复制黏贴。 多变性：CID 可以表示任意格式、任意哈希函数的结果。 避免内容锁：CID 要防止受限于历史内容。 可升级性：CID 的编码版本必须要可以升级。

## 37.4 可读的 CID 值

为了更好的调试和解释，我们需要 CID 的内容是有意义的，可读的。按照以下方法可以将普通 CID 转化为“用户可读 CID”：

`<hr-cid> ::= <hr-mbc> "-" <hr-cid-version> "-" <hr-mcp> "-" <hr-mh>`

每一部分都表示了各自的可读内容：

*   `<hr-mbc>` 是一种用户可读的 multibase 编码 (如`base58btc`)
*   `<hr-cid-version>` 是一个版本`cidv#` (如 `cidv1` 或 `cidv2`)
*   `<hr-mcp>` 是一种用户可读的 multicodec-packed 编码 (如`cbor`)
*   `<hr-mh>` 是一种用户可读的 multihash (如`sha2-256-256-abcdef0123456789...`)

例如:

```go
# TODO example
# example CID
# corresponding human readable CID
```

## 37.5 版本

### 37.5.1 CIDv0

CIDv0 是一个向后兼容的版本，其中:

CIDv0 是一个向后兼容的版本，其中:

*   `multibase` 一直为 `base58btc`
*   `multicodec` 一直为 `protobuf-mdag`
*   `cid-version` 一直为 `cidv0`
*   `multihash` 表示为： `cidv0 ::= <multihash-content-address>`

### 37.5.2 CIDv1

描述见: [`github.com/ipld/cid#how-does-it-work-protocol-description`](https://github.com/ipld/cid#how-does-it-work-protocol-description)

```go
<cidv1> ::= <multibase-prefix><cid-version><multicodec-packed-content-type><multihash-content-address> 
```

## 37.6 已有的实现方式

*   go-cid
*   java-cid
*   js-cid
*   rust-cid
*   py-cid