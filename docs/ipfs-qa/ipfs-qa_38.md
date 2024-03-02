# 第三十八章 【IPFS 一问一答】解析 IPFS 应用层 IPLD 之 Resolvers

# 38.【IPFS 一问一答】解析 IPFS 应用层 IPLD 之 Resolvers

IPLD resolver 即 IPLD 数据结构解析器，它是一个内部的 DAG API 模型：

*   put(node, options, callback)。存储 IPLD 数据结构的节点。

node：IPLD 结构的节点。 options：是一个对象，它必须要包含以下内容的其中一个： 1) cid - 节点的 CID 2) [hashAlg]、[version]和 format。它们被用来创建节点的 CID。 默认的`hashAlg` 和`version`对应默认 format。 3） callback 是一个带有签名的回调函数: `function (err, cid) {}`，`err`是执行函数时可能返回的错误； `cid`是存储对象的 CID。

*   get(cid [, path] [, options], callback)。通过给定的节点 CID 或路径检索到对应的节点。

    *   `options`是可选择的对象:

localResolve 是个 bool 类型，如果是`true`， 将只解析本地的路径

*   `callback`是一个带有签名的回调函数`function (err, result)`， `result`是包含下边内容的对象:

1）`value` - get 获取到的数据 2）`remainderPath` - 是否成功全路径被解析出来，或是否选择 localResolve。 3）`cid` - 遍历最后找到的节点。

*   getMany(cids, callback)。同时检索多个节点。

`callback` 是个带有签名的回调函数`function (err, result)`，`result` 是一个与 CID 对应的节点数组。

*   getStream(cid [, path] [, options])。和`get`一样，但是返回一个被用来传递节点（获取到的）的源`pull-stream`。

*   treeStream(cid [, path] [, options])。在一个`cid + path`下，通过`pull-stream`返回所有的路径。 `options`: `recursive`布尔类型 - 通过`lingks`遍历

*   remove(cid, callback)。通过给定的`cid`删除某个节点。

*   support.add(multicodec, formatResolver, formatUtil)。给 IPLD 数据结构添加一些支持。

*   support.rm(multicodec)。把 IPLD 的一些支持删除掉。