# 第三十五章 【IPFS 一问一答】解析 IPFS 应用层 IPLD 之 merkle-path

# 35\. 【IPFS 一问一答】解析 IPFS 应用层 IPLD 之 merkle-path

## 35.1 merkle-path 是什么?

一个 merkle-path 是一个 unix 风格的路径(e.g. /a/b/c/d)，它可以实现通过 merkle-link 遍历，并且获得所有的对象。

通用的文件系统可以被设计成在 IPFS 之上的对象模型，设计特定的算法来实现数据对象的操作和查询。

## 35.2 merkle-paths 的工作原理是什么?

一个 merkle-path 是一种 unix 风格的路径，它依据路径遍历，同时也逐步解析循内容。解析内容意味着获得 merkle-link 的内容，再进一步解析。

例如，假设我们有如下 merkle-path:

`/ipfs/QmUmg7BZC1YP1ca66rRtWKxpXp77WgVHrnv263JtDuvs2k/a/b/c/d`

其中:

*   `ipfs`是协议的命名空间，让电脑区别该做什么。
*   `QmUmg7BZC1YP1ca66rRtWKxpXp77WgVHrnv263JtDuvs2k`是一个加密哈希值
*   `a/b/c/d`是一个可遍历的 unix 路径

由`/`表示的可遍历的路径，可以表示两种链接：

*   在同一对象内部遍历数据
*   依据 merkle-link 实现对象间的信息遍历

例如 假设有如下数据集：

```go
> ipfs object cat --fmt=yaml QmUmg7BZC1YP1ca66rRtWKxpXp77WgVHrnv263JtDuvs2k
---
a:
  b:
    link:
      /: QmV76pUdAAukxEHt9Wp2xwyTpiCmzJCvjnMxyQBreaUeKT
    c: "d"
    foo:
      /: QmQmkZPNPoRkPd7wj2xUJe5v5DsY6MX33MFaGhZKB2pRSE
> ipfs object cat --fmt=yaml QmV76pUdAAukxEHt9Wp2xwyTpiCmzJCvjnMxyQBreaUeKT
---
c: "e"
d:
  e: "f"
foo:
  name: "second foo"
> ipfs object cat --fmt=yaml QmQmkZPNPoRkPd7wj2xUJe5v5DsY6MX33MFaGhZKB2pRSE
---
name: "third foo"
```

假设有如下 paths:

*   `/ipfs/QmUmg7BZC1YP1ca66rRtWKxpXp77WgVHrnv263JtDuvs2k/a/b/c`只会遍历第一个对象，得到字符串`d`.
*   `/ipfs/QmUmg7BZC1YP1ca66rRtWKxpXp77WgVHrnv263JtDuvs2k/a/b/link/c` 会遍历两个对象，得到字符串`e`
*   `/ipfs/QmUmg7BZC1YP1ca66rRtWKxpXp77WgVHrnv263JtDuvs2k/a/b/link/d/e` 会遍历两个对象，得到字符串`f`
*   `/ipfs/QmUmg7BZC1YP1ca66rRtWKxpXp77WgVHrnv263JtDuvs2k/a/b/link/foo/name`会遍历第一个，第二个对象，得到字符串`second foo`
*   `/ipfs/QmUmg7BZC1YP1ca66rRtWKxpXp77WgVHrnv263JtDuvs2k/a/b/foo/name`会遍历第一个，第二个对象，得到字符串`third foo`