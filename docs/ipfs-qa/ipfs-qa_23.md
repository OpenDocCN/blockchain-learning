# 第二十三章 【IPFS 一问一答】IPFS 底层架构解析之文件层

# 23 IPFS 底层架构解析之文件层

IPFS 在 Merkle DAG 上对版本化的文件系统进行了建模。这个模型与 Git 类似：

1.  block：一个可变大小的数据块
2.  list：一个块或其它列表的集合
3.  tree：块、列表或其它树的集合
4.  commit：树在版本历史记录中的一个快照

## 23.1 文件对象：blob

文件对象被存储时一般都要进行数据分割，每块大小为 256kb，但如果是小文件（< 256kb）就不需要做分割处理，直接将其作为一个数据单元进行存储，存储的类型就是 blob 对象。大文件切割后的数据单元也是 blob 对象。

每个 blob 对象包含的是一个可寻址的数据单元，如下所示：：

```go
{
    "data":"some data here", 
    //blobs 无 links
}
```

## 23.2 文件对象：list

文件大于 256kb 的大文件在 ipfs 存储时，需要进行数据分割。list 对象由几个 blob 对象组成，其中里面的 blob 可能有重复。lists 对象包含有序的 blob 和 list 对象。这些有序的 blob 和 list 会有自己的 link，link 对应了他们的数据信息。

如下结构标示了一个 lists 结构：

```go
{
    "data":["blob","list","blob"],  //lists 有一个对象类型的数组作为数据
    "links":[
        {
            "hash":"XLYkgq61DYaq8Nhkcqy7LcnSA7dSHQ78x",
            "size":189458
        },
        {
            "hash":"XLHBNsgoepUDKL8dkd9Hesa5io9sdxi7n",
            "size":19442
        },
        {
            "hash":"XLWVQKJII8v7dggkfdhHSFlkaw9yjs7dj",
            "size":5286
        } // 在 links 中 lists 是没有名字的
    ]
}
```

## 23.3 文件对象：tree

在 IPFS 中，Tree 对象与 Git 的 tree 类似，代表一个目录，或者一个名字到哈希值的映射表，哈希值表示 blob，list，其他的 tree，或 commit，结构如下：

```go
{
    "data":["blob","list","blob"], // trees 有一个对象类型的数组作为数据
    "links":[
        {
            "hash":"XLYkgq61DYaq8Nhkcqy7LcnSA7dSHQ78x",
            "name":"less",
            "size":189458
        },
        {
            "hash":"XLHBNsgoepUDKL8dkd9Hesa5io9sdxi7n",
            "name":"script",
            "size":19442
        },
        {
            "hash":"XLWVQKJII8v7dggkfdhHSFlkaw9yjs7dj",
            "name":"template",
            "size":5286
        } // trees 是没有名字的
    ]
}
```

## 23.4 文件对象：commit

在 IPFS 中，commit 对象代表任何对象在版本历史记录中的一个快照，它与 Git 的 commit 也非常类似，但它可以指向任何类型的对象。

```go
{
"data": {
    "type": "tree",
    "date": "2014-09-20 12:44:06Z",
    "message": "This is a commit message."
}, "links": [
{ "hash": "XLa1qMBKiSEEDhojb9FFZ4tEvLf7FEQdhdU",
  "name": "parent", "size": 25309 },
{ "hash": "XLGw74KAy9junbh28x7ccWov9inu1Vo7pnX",
  "name": "object", "size": 5198 },
{ "hash": "XLF2ipQ4jD3UdeX5xp1KBgeHRhemUtaA8Vm",
  "name": "author", "size": 109 }
  ]
}
> ipfs file-cat <ccc111-hash> --json
{
  "data": {
    "type": "tree",
    "date": "2014-09-20 12:44:06Z",
    "message": "This is a commit message."
}, "links": [
    { "hash": "<ccc000-hash>",
      "name": "parent", "size": 25309 },
    { "hash": "<ttt111-hash>",
      "name": "object", "size": 5198 },
    { "hash": "<aaa111-hash>",
      "name": "author", "size": 109 }
  ]
}

> ipfs file-cat <ttt111-hash> --json
{
  "data": ["tree", "tree", "blob"],
  "links": [
    { "hash": "<ttt222-hash>",
      "name": "ttt222-name", "size": 1234 },
    { "hash": "<ttt333-hash>",
      "name": "ttt333-name", "size": 3456 },
    { "hash": "<bbb222-hash>",
      "name": "bbb222-name", "size": 22 }
 ] 
}

> ipfs file-cat <bbb222-hash> --json
{
  "data": "blob222 data",
"links": [] } 
```

## 23.5 版本控制

IPFS 的版本控制是在 Git 的基础上实现的，Git 版本控制在之前的章节有讲。commit 对象代表着一个对象在历史版本中的一个特定快照。两个不同的 commit 之间相互比较对象数据，可以揭露出两个不同版本文件系统的区别。IPFS 可以实现 Git 版本控制工具的所有功能，同时也可以兼容 Git。

兼容的原因可能是：

*   使用 Git 工具版本改造成的 IPFS
*   IPFS 的挂载 FUSE 文件系统，挂载一个 IPFS 的 tree 作为 Git 的版本库，把 Git 文件系统的读写转换成 IPFS 的格式。

## 23.6 文件系统路径

在 Merkle DAG 中，我们可以看到，IPFS 对象可以使用字符串路径 API 来遍历。IPFS 文件对象是特意设计的，为的是让挂载 IPFS 到 UNIX 文件系统更加简单。

## 23.7 将文件分割成 list 和 blob

版本控制和分发大文件最主要的挑战：找到一个正确的方法来将它们分隔成独立的块。与其认为 IPFS 可以为每个不同类型的文件提供正确的分隔方法，不如说 IPFS 提供了以下的几个可选选择：

使用 Rabin Fingerprints 指纹算法来定义比较合适的块边界。 使用 rsync 和 rolling-checksum 算法，来检测块在版本之间的改变。 允许用户设定文件大小而调整数据块的分割策略。

## 23.8 路径查找性能

基于路径的访问需要遍历整个对象图，检索每个对象需要在 DHT 中查找它的 Key 值，连接到对等点并检索对应的数据块。这是一笔相当大的性能开销，特别是在查找的路径具有多个路径时。IPFS 充分考虑了这一点，并设计了如下的方式来缓解：

*   树缓存（tree cache）：由于所有的对象都是哈希寻址的，可以被无限的缓存，另外，tree 一般比较小，所以比起 blob，IPFS 会优先缓存 tree。
*   扁平树（flattened trees）：对于任何给定的 tree，一个特殊的扁平树可以构建一个链表，所有对象都可以从这个 tree 中访问得到。在扁平树中 name 就是一个从原始 tree 分离的路径，用斜线分隔。

![](img/64e5c7e7c2a22f2890ff6aeac8c58ccb.jpg)

例如，对于上面的 ttt111 的扁平树如下：

```go
{
    "data":["tree","blob","tree","list","blob","blob"],
    "links":[
        {"hash":"<ttt222-hash>","size":1234,"name":"ttt222-name"},
        {"hash":"<bbb111-hash>","size":123,"name":"ttt222-name/bbb111-name"},
        {"hash":"<ttt333-hash>","size":3456,"name":"ttt333-name"},
        {"hash":"<lll111-hash>","size":578,"name":"ttt333-name/lll111-name"},
        {"hash":"<bbb222-hash>","size":22,"name":"ttt333-name/lll111-name/bbb222-name"},
        {"hash":"<bbb222-hash>","size":22,"name":"bbb222-name"},
    ]
}
```