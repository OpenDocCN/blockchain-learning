# 三、.3 教程 3: 从 ipfs 网络中下载数据到本地

ipfs 的使用和 http 模式一样。你可以在线访问数据，也可以下载数据。

如果你不想成为 IPFS 节点，却又想在 ipfs 网络中浏览数据，没问题。你可以直接在网页中使用 hash 访问数据，就像 http 一样，在线观看电影一样。访问的入口是 https://ipfs.io/ipfs/hash（你想检索的数据 hash 值）。

如果你想在 ipfs 网络中下载某个数据，那你必须是 ipfs 网络节点，否则不能下载数据。 下载数据很简单：

1.  确定你的 ipfs 节点已经启动；
2.  使用`ipfs get` 命令下载你想要的数据。这里我提供一个 ipfs 网络中的 hash 值`QmbrevseVQKf1vsYMsxCscRf6D7S2dftYpHwxkYf94pc7T`。 首先确定好你想要将数据下载到哪里。 新开启一个终端窗口，进入到指定目录下，执行： `ipfs get QmbrevseVQKf1vsYMsxCscRf6D7S2dftYpHwxkYf94pc7T`，显示如下：

```go
$ ipfs get QmbrevseVQKf1vsYMsxCscRf6D7S2dftYpHwxkYf94pc7T
Saving file(s) to QmbrevseVQKf1vsYMsxCscRf6D7S2dftYpHwxkYf94pc7T
```

ok，此时这个数据就会被下载到你的当前目录下了。