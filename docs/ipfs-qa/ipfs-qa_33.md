# 第三十三章 【IPFS 一问一答】解析 IPFS 应用层 IPLD 之 merkle-link

# 33\. 【IPFS 一问一答】解析 IPFS 应用层 IPLD 之 merkle-link

一个 merkle-link 是链接两个对象的方式。目标对象和源对象都使用加密 Hash 的内容寻址。同时，目标对象的 hash 也会嵌入到源对象中。包含 merkle-links 的内容寻址可以做到：

*   加密完整性检查：解析链接的值可以通过 Hash 来测试。 这样一来可以实现广泛的，安全的，不受信任的数据交换（例如 git 或 bittorrent），因为其他人不能给你任何不通过 Hash 链接到的数据。
*   不可变数据结构：带有 merkle 链接的数据结构不能改变，这对于分布式系统来说是一个重要的属性。 这对于版本控制，表示分布式可变状态（例如 CRDT）和长期归档很有用。

一个 merkle-link 通过如下的 IPLD 对象模型表示：一个包含 / 映射到一个 “映射值”（“link value”），例如：

一个链接，在 json 中可以表示为一个“链接对象”（“link object”）。

```go
{ 
   "/" : "/ipfs/QmUmg7BZC1YP1ca66rRtWKxpXp77WgVHrnv263JtDuvs2k"
}
// "/" 是一个链接键
// "/ipfs/QmUmg7BZC1YP1ca66rRtWKxpXp77WgVHrnv263JtDuvs2k" 是链接值
```

一个在 foo/baz 有链接的对象：

```go
{
  "foo": {
       "bar": "/ipfs/QmUmg7BZC1YP1ca66rRtWKxp77WgVHrnv263JtDuvs2k", // 不是一个链接
       "baz":
      {"/": "/ipfs/QmUmg7BZC11ca66rRtWKxpXp77WgVHrnv263JtDuvs2k"} 
// 一个链接
         }
} 
```

一下结构中又一个 有一个伪”链接对象” 在 files/cat.jpg ，而实际的链接在 `files/cat.jpg/link`

```go
{
  "files": {
    "cat.jpg": { 
// 链接的属性包含在其他对象中
      "link": {
      "/": "/ipfs/QmUmg7BZC1YP1ca66rRtWKxpXp77WgVHrnv263JtDuvs2k"}, // 链接
      "mode": 0755,
      "owner": "jbenet"
    }
  }
}
```

当链接被修改时，映射本身将被其指向的对象替换，除非链接路径无效。

这个链接可以是 multihash, 也就是说它假设这个链接是在 /ipfs 层级下的，或者是对象的绝对路径。但目前只有 /ipfs 层级路径可以使用。

如果应用需要使用 /表示其他内容，那么应用自身需要保证解析的时候不冲突。