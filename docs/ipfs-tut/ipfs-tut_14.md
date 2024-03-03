# 4.1 教程 1：使用 IPNS 发布你的网页

在之前的课程中，我们可以将自己的数据共享到 IPFS 网络中，但是数据在网页被访问时使用的都是数据内容的 hash 值。所以当内容发生变化后，我们是不能访问到最新的数据的。

## 教程目标

通过本小节的学习，使你能够重新编辑发布过的数据，而不需要修改访问的 hash 值

## 学习步骤

### 第一步：上传一个文件到 ipfs 网络

根据之前的课程，我们可以将一个文件发布到 ipfs 网络。我创建了一个文件 ipns01，内容是“chaindesk”

```go
$ vi ipns01
$ ipfs add ipns01 
added QmTry1Hy6LJBz4uDTF6KwTnTmhAj8D15vnYyCsmhabvXaX ipns01
```

现在在网络中访问 ipfs.io/ipfs/QmTry1Hy6LJBz4uDTF6KwTnTmhAj8D15vnYyCsmhabvXaX 。没问题 ![](img/af7ecdacf141dc3f455d6dab2f897f5c.jpg)

### 第二步：使用 IPNS 访问文件

如果我上一个文件内容，修改了，那么你再用上边的网址是访问不到修改过的内容的。所以就引入了 IPNS。 在 IPNS 中，允许我们节点的域名空间中引用一个 IPFS hash，也就是说我们可以通过节点 ID 对项目根目录的 IPFS HASH 进行绑定，以后我们访问网站时直接通过节点 ID 访问即可，当我们更新文件时，重新发布到 IPNS`即可。

使用命令是`ipfs name publish 文件 hash`，操作如下：

```go
$ ipfs name publish QmTry1Hy6LJBz4uDTF6KwTnTmhAj8D15vnYyCsmhabvXaX
Published to QmXdSpUBx9Ut6q8LF8Wyt1Wi2wxmob6qnnT11uV3SmvUP3: /ipfs/QmTry1Hy6LJBz4uDTF6KwTnTmhAj8D15vnYyCsmhabvXaX
```

我们再查看下我们的 ipfs 节点 IDhash，是不是和上边的 hash 一样呢？

```go
$ ipfs id
{
    "ID": "QmXdSpUBx9Ut6q8LF8Wyt1Wi2wxmob6qnnT11uV3SmvUP3",
    "PublicKey": "CAAS... 
```

一样的。

通过命令`ipfs name resolve <IDhash>`验证该 hash 解析的值，操作如下：

```go
$ ipfs resolve QmXdSpUBx9Ut6q8LF8Wyt1Wi2wxmob6qnnT11uV3SmvUP3
/ipfs/QmTry1Hy6LJBz4uDTF6KwTnTmhAj8D15vnYyCsmhabvXaX
```

ok，我们使用新的网址访问 ipns01 文件，网址是：ipfs.io/ipns/QmXdSpUBx9Ut6q8LF8Wyt1Wi2wxmob6qnnT11uV3SmvUP3 。注意网址中 ipns，不是 ipfs 字样。

![](img/871922973d5a261bb6a7f9be0cd04ce1.jpg)

访问到的内容是一样的。

记住，每次修改文件内容，发布到 ipfs 网络后，都需要执行`ipfs name publish 新返回的 hash`，使最新内容发布。这样 ipns 访问到的都是最新的内容。

```go
$ vi ipns01 
localhost:IPFS zhanghengxing$ ipfs add ipns01 
added QmU8cbkwRq9V6t39NFQ1BWxjf4EksmfxFenNPdREtazc7h ipns01
 23 B / 23 B [=========================================================] 100.00%localhost:IPFS 
$ ipfs name publish QmU8cbkwRq9V6t39NFQ1BWxjf4EksmfxFenNPdREtazc7h
Published to QmXdSpUBx9Ut6q8LF8Wyt1Wi2wxmob6qnnT11uV3SmvUP3: /ipfs/QmU8cbkwRq9V6t39NFQ1BWxjf4EksmfxFenNPdREtazc7h 
```

再次访问 ![](img/a7acfb2c56c63dbea0aea944886919c4.jpg)