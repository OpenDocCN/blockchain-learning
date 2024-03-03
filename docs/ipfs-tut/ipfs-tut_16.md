# 4.3 教程 3: 运用自己的域名发布网页

首先你要有自己注册的域名，我自己注册了一个`ipfsexplore.cn`。

ipns 允许为哈希地址绑定域名，很简单，只需要在域名解析里面添加一条 TXT 记录即可：

```go
dnslink=/ipfs/<your_hash>
```

而教程 1 和教程 2，我们已经有了 IPNS 值，所以这里的<your_hash>就是 IPNS 值。</your_hash>

例如我用教程 2 的 hash 值`QmX4KM1J82p895JveV3xfL4aNoEPqaiUmMEkTiaHMgj8Ru`，那么 TXT 解析的值为：

```go
dnslink=/ipfs/QmX4KM1J82p895JveV3xfL4aNoEPqaiUmMEkTiaHMgj8Ru
```

这样我在域名解析里添加这个记录，并指向自己运行 ipfs 节点的主机，域名绑定完成。

最好访问的地址就是：[`ipfs.io/ipns/ipfsexplore.cn`](https://ipfs.io/ipns/ipfsexplore.cn)