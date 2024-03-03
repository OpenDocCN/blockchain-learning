# 3.4 教程 4: Pinning-锁定文件永久保存

## 教程目标

通过学习本小节，使你：

1.  学会使用`pin`命令锁定你喜欢的数据
2.  明白你上传的数据具体存到哪了，是以什么方式存储的，如实际硬盘存储还是缓存存储。
3.  手动清理 ipfs 缓存

## 学习步骤

### 第一步：查看本地上传和检索到的数据存哪了

我们本地上传到 ipfs 的数据，就是使用`ipfs add`命令添加到本地 ipfs 存储库了。ipfs 的存储库有些人翻译成 ipfs 资源库，一个东西。这个存储库在我们`ipfs init`时，在本机用户名下创建的，不过是个隐藏文件夹，显示隐藏文件后，你会看见一个./ipfs 的文件夹，这就是本机的 ipfs 存储库。打开./ipfs 文件夹之后，你会看到： ![](img/541858577df70510802e54011aa47c6f.jpg)

我们添加的数据就存储在上图中 blocks 中了，不过是以块数据的类型存储的。 从 ipfs 网络下载的数据，即使用`ipfs get`得到的数据同样也存储到 blocks 中了。

### 第二步：使用 pin 锁定你喜欢的数据

存储在 blocks 中的数据，有两种方式，一个是硬盘存储（永久保存），另一个是缓存存储（系统定期清理）。用户使用`ipfs add`添加的数据都是硬盘存储的，而`ipfs get`到的数据都是以缓存存储的。这里将提到`pin`的作用。

实际上 blocks 中的数据存储方式是由`pin`决定的。`pin`的作用是锁定数据，保证数据永不被系统清理掉。上面说的两种存储方式，现在可以说成一个是数据被`pin`处理过的，另一个是没有被处理过的。

ipfs 系统会默认将`ipfs init`后的数据以及`ipfs add`的数据进行`pin`处理后存储到 blocks 中的。我们可以通过`ipfs pin ls`查看`pin`过的数据，如下：

```go
$ ipfs pin ls
QmZTR5bcpQD7cFgTorqxZDYaew1Wqgfbd2ud9QqGPAkK2V indirect
QmQ5vhrL7uv6tuoN9KeVBwd4PwfQkXdVVmDLUZuTNxqgvm indirect
QmS4ustL54uo8FzR9455qaxZwuMiUhyvMcX9Ba8nUH4uVv recursive
QmUNLLsPACCz1vLxQVkXqqLX5R1X345qqfHbsf67hvA3Nn recursive
QmWj7uThVH9a2HBiewA53U6g1uwzj1N4r4LNDSV7V3BgGi indirect
QmXgqKTbzdh83pQtKFb19SpMCpDDcKR2ujqk3pKph9aCNF indirect
QmaTbcssmxUTB7na2ggMDTBXaUbSJzJsT5xQEnj3KJx9VL recursive
QmY5heUM5qgRubMDD1og9fhCPA6QdkMp3QCwd4s7gJsyE7 indirect
QmbrevseVQKf1vsYMsxCscRf6D7S2dftYpHwxkYf94pc7T recursive
QmPZ9gcCEpqKTo6aq61g2nXGUhM4iCL3ewB6LDXZCtioEB indirect
QmREJwRsxCuENyEhH2cfHbA7Q3g43MBAkUFFaV36TZ3YwH recursive
QmYCvbfNbCwFR45HiNP45rwJgvatpiW38D961L5qAhUM5Y indirect
QmZmr5ECfGMJLJGM1yx8LNQznYraxCXqPzN6S6V3R7NxFP indirect
QmejvEPop4D7YUadeGqYWmZxHhLc4JBUCzJJHWMzdcMe2y indirect
```

你可以 add 一个小数据，查看 pin 列表中，有没有刚刚产生的 hash 值？

而`ipfs get`获取到的数据，并没有被`pin`处理过，将属于缓存存储。但如果你对这份数据很感兴趣，想永久保存它，ok，没问题，我们可以手动`pin`处理该数据，将其转换为硬盘存储，命令很简单 `ipfs pin add <hash>` 在终端的操作如下：

```go
localhost:ipfs-tutorial zhanghengxing$ ipfs pin add QmbrevseVQKf1vsYMsxCscRf6D7S2dftYpHwxkYf94pc7T
pinned QmbrevseVQKf1vsYMsxCscRf6D7S2dftYpHwxkYf94pc7T recursively
```

现在你再打开`pin`列表，该 hash 将会出现在里面。

如果你对这个数据不感兴趣了，我们可以解除它的`pin`锁定，命令`ipfs pin rm -r <foo hash>`。 操作如下：

```go
$ ipfs pin rm -r QmbrevseVQKf1vsYMsxCscRf6D7S2dftYpHwxkYf94pc7T
unpinned QmbrevseVQKf1vsYMsxCscRf6D7S2dftYpHwxkYf94pc7T
```

这样这个数据就被解除`pin`了。`pin`列表中将不会存在这个 hash 值。

小提示：`ipfs cat`的数据都是在 blocks 中存储的数据，如果 blocks 中没有该数据，将会从 ipfs 网络中请求数据。

⚠️：接下来的操作需要关掉本地 ipfs 节点。

我获取下解除`pin`的数据还在不在？

```go
$ ipfs cat QmbrevseVQKf1vsYMsxCscRf6D7S2dftYpHwxkYf94pc7T
liyc1215 
```

数据还存在。

缓存的有效期是多长时间呢？我不确定，

我们可以手动清理缓存，命令`ipfs repo gc`，操作如下：

```go
$ ipfs repo gc
removed QmVvCAgjEgTVMpm9ng3Re9ByiCSZDaXiotmv3DnfywDq4t
removed QmXgnwQn6xZUYXtq8gtKejDffU2cnigqiHQ6CrKWK94PUj
removed Qmc7cgRoGHfvepuBEk1TY3coByY9bwH3AsXnoUvtnQjPSr
removed QmbrevseVQKf1vsYMsxCscRf6D7S2dftYpHwxkYf94pc7T
removed QmbUNyTXGMXj1N4symorb9ZuN5vFWxw6M7NFFox8AeeuDk
removed QmPymDr8PPpyBXtFPDncrybF9V1G6XmGuamhc6r3BVFRXn
removed QmP4z1hBdsGsLEHGDCx15VBo8E7XqouUs3tZrhKWBfsYGx
```

被清理的就有`QmbrevseVQKf1vsYMsxCscRf6D7S2dftYpHwxkYf94pc7T`。 现在我们再 cat 这个数据，还能吗？

```go
$ ipfs cat QmbrevseVQKf1vsYMsxCscRf6D7S2dftYpHwxkYf94pc7T
Error: merkledag: not found 
```

数据已经找不到了。

明白了`pin`的使用，赶快锁定你喜欢的数据吧。