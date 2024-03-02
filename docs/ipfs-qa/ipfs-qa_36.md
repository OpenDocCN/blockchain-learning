# 第三十六章 【IPFS 一问一答】解析 IPFS 应用层 IPLD 之 IPLD Tree

# 36\. 【IPFS 一问一答】解析 IPFS 应用层 IPLD 之 IPLD Tree

## 36.1 什么是 IPLD 数据模型?

IPLD 数据模型定义了一种简单的，适用于所有 merkle-dags，基于 JSON 的结构。同时也定义了一系列编码的格式结构。

## 36.2 限制和愿景

有如下限制：

*   IPLD 路径必须是明确无误的，任意给定的路径遍历的方式必须是恒定的（e.g.避免链接命名冲突）
*   IPLD 路径必须是全局的，同时也要支持其他语言(e.g. 使用 UTF-8,而不是 ASCII).
*   IPLD 路径必须是在 UNIX 和 Web 之上的层级 (使用 /, 在 ASCII 系统内转变必须是确定的 ).
*   鉴于 JSON 的成功, 很多系统都支持 JSON 接口. IPLD 必须具有支持 JSON 格式的导入导出能力
*   JSON 数据模型也是简单而且易于使用的。IPLD 也必须要易于使用。
*   可以让定义数据的操作变得很简单。在 IPLD 之上定义新的数据结构不需要具有很多背景知识
*   由于 IPLD 是基于 JSON 数据模型的， 它应该通过 JSON-LD 与 RDF 及 Linked Data standards 兼容
*   IPLD 序列化格式(在磁盘上，传输中)都需要快速和空间有效 (不能用 JSON 格式存储, 而是应该用 CBOR 或其他格式)
*   IPLD 加密哈希散列必须可升级 (使用 multihash)

如下特性是加分项:

*   IPLD 不应该包含错误的数据，e.g. 存储不完整的 JSON.
*   IPLD 应该可升级, e.g. 如果一种更好的在磁盘上存储的格式出现了，系统应该可以只要花费很小的代价就升级
*   IPLD 对象应该可以可以解析属性，而不仅仅是 merkle links.
*   IPLD 事先定义的格式应该容易实现和转化
*   IPLD 事先定义的格式应该在不获得整个对象的情况下也可以搜索(CBOR 和 Protobuf 已经可以做到).

## 36.3 格式定义

(住: 此处同时使用 JSON 和 YML 来展示格式的形式。我们使用两者来显示两种不同格式的对象的等价性。）

IPLD 数据模型的核心仅仅是 JSON，也就是说它 (a) 是树状结构的文件，有一些原始类型，(b) 与 JSON 实现一对一映射 © 可以在 json 中使用它

另一方面，它又不是“JSON”因为 (a)它纠正了一些错误, (b)实现了高效序列化 , 同时 ©并不局限于单一的传输格式，有待于新的技术

### 36.3.1 Basic Node 单节点

下面是一个用 JSON 表示的 IPLD 对象 :

```go
{
  "name": "Vannevar Bush"
}
```

假设它的 multihash 值为`QmAAA...AAA`。 注意到其中没有链接，只有字符串。但是用户仍可以通过 key`name`来解析它。

```go
> ipld cat --json QmAAA...AAA
{
  "name": "Vannevar Bush"
}
> ipld cat --json QmAAA...AAA/name
"Vannevar Bush"
```

同时，在 yml 结构中:

```go
> ipld cat --yml QmAAA...AAA
---
name: Vannevar Bush
> ipld cat --xml QmAAA...AAA
<!xml> <!-- todo -->
<node>
  <name>Vannevar Bush</name>
</node>
```

### 36.3.2 Linking Between Nodes 节点间链接

节点间的 Merkle-Link 是 IPLD 存在的原因。IPLD 内的 Link 只是节点内部包含的特殊格式：

```go
{
  "title": "As We May Think",
  "author": {
    "/": "QmAAA...AAA" // 链接到上个例子的数据
  }
}
```

假设以上数据的 multihash 值为`QmBBB...BBB`。 该节点有 _subpath 链接`author`到`QmAAA...AAA`，所以可以有如下操作：

```go
> ipld cat --json QmBBB...BBB
{
  "title": "As We May Think",
  "author": {
    "/": "QmAAA...AAA" // links to the node above.
  }
}
> ipld cat --json QmBBB...BBB/author
{
  "name": "Vannevar Bush"
}
> ipld cat --yml QmBBB...BBB/author
---
name: "Vannevar Bush"
> ipld cat --json QmBBB...BBB/author/name
"Vannevar Bush" 
```

### 36.3.3 Link Properties Convention 链接属性约定

通过使用链接，IPLD 的用户可以构建复杂的数据结构。这样可以方便的将信息编码，例如表示一种关系结构，或者添加辅助数据。但是它不同于接下来要讨论的“对象链接约定”。有时候，你只是想在链接上添加一些数据，而不必制作另一个对象。 IPLD 不会妨碍你。 您可以简单地通过将实际的 IPLD 链接嵌套在另一个对象中，并使用附加属性来完成。

请注意: 链接属性不允许直接在链接对象，因为 travesal 歧义。 阅读 spec 历史记录，就实现困难的讨论。

例如，假设你有一个文件系统，并且想分发权限控制的元信息,或者节点间的归属权。假设你有一个目录对象 `directory`它的哈希值为`QmCCC...CCC`:

```go
{
  "foo": { // link wrapper with more properties
    "link": {"/": "QmCCC...111"} // the link
    "mode": "0755",
    "owner": "jbenet"
  },
  "cat.jpg": {
    "link": {"/": "QmCCC...222"},
    "mode": "0644",
    "owner": "jbenet"
  },
  "doge.jpg": {
    "link": {"/": "QmCCC...333"},
    "mode": "0644",
    "owner": "jbenet"
  }
}
```

用 YML 表示：

```go
---
foo:
  link:
    /: QmCCC...111
  mode: 0755
  owner: jbenet
cat.jpg:
  link:
    /: QmCCC...222
  mode: 0644
  owner: jbenet
doge.jpg:
  link:
    /: QmCCC...333
  mode: 0644
  owner: jbenet
```

即使我们在数据结构中增加了特有属性的描述，仍然可以解析：

```go
> ipld cat --json QmCCC...CCC/cat.jpg
{
  "data": "\u0008\u0002\u0012��\u0008����\u0000\u0010JFIF\u0000\u0001\u0001\u0001\u0000H\u0000H..."
}
> ipld cat --json QmCCC...CCC/doge.jpg
{
  "subfiles": [
    {
      "/": "QmPHPs1P3JaWi53q5qqiNauPhiTqa3S1mbszcVPHKGNWRh"
    },
    {
      "/": "QmPCuqUTNb21VDqtp5b8VsNzKEMtUsZCCVsEUBrjhERRSR"
    },
    {
      "/": "QmS7zrNSHEt5GpcaKrwdbnv1nckBreUxWnLaV4qivjaNr3"
    }
  ]
}
> ipld cat --yml QmCCC...CCC/doge.jpg
---
subfiles:
  - /: QmPHPs1P3JaWi53q5qqiNauPhiTqa3S1mbszcVPHKGNWRh
  - /: QmPCuqUTNb21VDqtp5b8VsNzKEMtUsZCCVsEUBrjhERRSR
  - /: QmS7zrNSHEt5GpcaKrwdbnv1nckBreUxWnLaV4qivjaNr3
> ipld cat --json QmCCC...CCC/doge.jpg/subfiles/1/
{
  "data": "\u0008\u0002\u0012��\u0008����\u0000\u0010JFIF\u0000\u0001\u0001\u0001\u0000H\u0000H..."
}
```

但是 link 的解析仍需要通过遍历获得。

### 36.3.4 Duplicate property keys 重复属性

值得注意的是目前不允许有重名的属性，即使这样通过 parsers 也可能会出现这样的问题。所以，为了避免冲突，解析时只读取第一个出现的条目。例如，有如下对象：

```go
{
  "name": "J.C.R. Licklider",
  "name": "Hans Moravec"
}
```

以这样的顺序出现在在标准格式中 Canonical Format (不是 json, 是 cbor), 它的哈希值为 `QmDDD...DDD`。 我们只能得到:

```go
> ipld cat --json QmDDD...DDD
{
  "name": "J.C.R. Licklider",
  "name": "Hans Moravec"
}
> ipld cat --json QmDDD...DDD/name
"J.C.R. Licklider"
```

### 36.3.5 Path Restrictions 路径限制

Unix 和 Web 中的路径描述有一些重要的问题。 具体请看这个讨论 this discussion。 为了与 unix 和 web 的模型和期望兼容，IPLD 明确地禁止具有特定路径组件的路径。 请注意，数据本身可能仍然包含这些属性（有人会这样做，并有合法的用途）。 所以只有路径解析器不能通过这些路径来解析。 这些限制与典型的 unix 和 UTF-8 路径系统相同：

### 36.3.6 Integers in JSON

IPLD 与 JSON 直接兼容，可以充分利用 JSON 的成功，但是不用被 JSON 的错误所束缚。 这是我们能够遵循格式惯用选择的地方，但必须注意确保始终存在定义明确的 1：1 映射。 关于整数，在 JSON 中有多种表示整数的字符串，例如 EJSON。 这些可以被使用，并且从其他格式的转换应该自然地发生 — 也就是说，当将 JSON 转换为 CBOR 时，EJSON 整数应该自然地转换为适当的 CBOR 整数，而不是将其表示为具有字符串值的映射。

### 36.3.7 数据结构例子

PLD 是一种简单，灵活和灵活的格式，不妨碍用户定义新的或导入的旧数据文件，这一点很重要。 为此，下面我将展示一些示例数据结构。

Unix 文件系统

A small File 一个小文件

```go
{
  "data": "hello world",
  "size": "11"
} 
```

A Chunked File 文件的一部分 将文件分解

```go
{
  "size": "1424119",
  "subfiles": [
    {
      "link": {"/": "QmAAA..."},
      "size": "100324"
    },
    {
      "link": {"/": "QmAA1..."},
      "size": "120345",
      "repeat": "10"
    },
    {
      "link": {"/": "QmAA1..."},
      "size": "120345"
    },
  ]
}
```

A Directory 一个目录

```go
{
  "foo": {
    "link": {"/": "QmCCC...111"},
    "mode": "0755",
    "owner": "jbenet"
  },
  "cat.jpg": {
    "link": {"/": "QmCCC...222"},
    "mode": "0644",
    "owner": "jbenet"
  },
  "doge.jpg": {
    "link": {"/": "QmCCC...333"},
    "mode": "0644",
    "owner": "jbenet"
  }
}
```

git git blob 数据块

```go
{
  "data": "hello world"
} 
```

git tree 树状结构

```go
{
  "foo": {
    "link": {"/": "QmCCC...111"},
    "mode": "0755"
  },
  "cat.jpg": {
    "link": {"/": "QmCCC...222"},
    "mode": "0644"
  },
  "doge.jpg": {
    "link": {"/": "QmCCC...333"},
    "mode": "0644"
  }
}
```

git commit 提交信息

```go
{
  "tree": {"/": "e4647147e940e2fab134e7f3d8a40c2022cb36f3"},
  "parents": [
    {"/": "b7d3ead1d80086940409206f5bd1a7a858ab6c95"},
    {"/": "ba8fbf7bc07818fa2892bd1a302081214b452afb"}
  ],
  "author": {
    "name": "Juan Batiz-Benet",
    "email": "juan@benet.ai",
    "time": "1435398707 -0700"
  },
  "committer": {
    "name": "Juan Batiz-Benet",
    "email": "juan@benet.ai",
    "time": "1435398707 -0700"
  },
  "message": "Merge pull request #7 from ipfs/iprs\n\n(WIP) records + merkledag specs"
}
```

Bitcoin 比特币 Bitcoin Block 比特币区块

```go
{
  "parent": {"/": "Qm000000002CPGAzmfdYPghgrFtYFB6pf1BqMvqfiPDam8"},
  "transactions": {"/": "QmTgzctfxxE8ZwBNGn744rL5R826EtZWzKvv2TF2dAcd9n"},
  "nonce": "UJPTFZnR2CPGAzmfdYPghgrFtYFB6pf1BqMvqfiPDam8"
}
```

Bitcoin Transaction 比特币交易 以 YML 的格式表示.

```go
---
inputs:
  - input: {/: Qmes5e1x9YEku2Y4kDgT6pjf91TPGsE2nJAaAKgwnUqR82}
    amount: 100
outputs:
  - output: {/: Qmes5e1x9YEku2Y4kDgT6pjf91TPGsE2nJAaAKgwnUqR82}
    amount: 50
  - output: {/: QmbcfRVZqMNVRcarRN3JjEJCHhQBcUeqzZfa3zoWMaSrTW}
    amount: 30
  - output: {/: QmV9PkR2gXcmUgNH7s7zMg9dsk7Hy7bLS18S9SHK96m7zV}
    amount: 15
  - output: {/: QmP8r8fLUnEywGnRRUrHB28nnBKwmshMLiYeg8udzYg7TK}
    amount: 5
script: OP_VERIFY 
```