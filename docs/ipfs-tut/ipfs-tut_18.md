# 五、.1 教程 1: 安装和使用 google 浏览器插件 ipfs companion

IPFS 伴侣（IPFS Companion）是一个由 IPFS 官方应用社区（IPFS-Shipyard）孵化出来的应用项目。 它是一个浏览器插件，可以帮助用户在本地更好的运行、管理自己的节点，并随时查看 IPFS 节点的资源信息。 ![](img/3a1ec5fe23c925760fe571f5562c4807.jpg)

## 课程目标

1.  学会如何安装 ipfs companion
2.  学会如何使用 ipfs companion

## 学习步骤

### 第一步：安装 ipfs companion 插件

*   Firefox Beta 版本 : [`ipfs.io/ipfs/QmR8W5wg8BuAyBTruHnHovfWRavwvidVh3qtyinXi6NnLa`](https://ipfs.io/ipfs/QmR8W5wg8BuAyBTruHnHovfWRavwvidVh3qtyinXi6NnLa)
*   Chrome Beta 版本: [`chrome.google.com/webstore/detail/ipfs-companion-beta-a211e/hjoieblefckbooibpepigmacodalfndh`](https://chrome.google.com/webstore/detail/ipfs-companion-beta-a211e/hjoieblefckbooibpepigmacodalfndh)

ipfs companion 是个浏览器插件，目前只支持 Firefox 和 Google 浏览器。如上两个版本。 我本机是 Google 浏览器，安装的就是 Chrome Beta 版本的插件。没有这两个浏览器的朋友需要下载下。

打开 https://chrome.google.com/webstore/detail/ipfs-companion-beta-a211e/hjoieblefckbooibpepigmacodalfndh 后，页面有`添加扩展程序`字样，点击。然后就会开始下载并自行安装。

完成后，安装就 ok 了，就这么简单。在浏览器里就会和我的一样，有那么蓝色 ipfs 小方块了。

### 第二步：启动你的 ipfs 节点

在浏览器中打开 ipfs companion，显示的是离线状态，那是因为你没有启动你的节点。 很简单，在终端输入`ipfs daemon`启动下就可以了。

启动后显示了本机节点的网关地址、API、使用的 ipfs 版本和已连入的 ipfs 节点数量。 网关你可以点击切换网关，自定义的和公共网关之间切换。

### 第三步：使用 ipfs companion 分享你的文件

1.点击`通过 IPFS 分享文件`，就会进入，下面界面： ![](img/8eb412f2d620d6036280743f25a29f78.jpg)

2.点击`选一个文件`，共享你的某个文件到 ipfs 网络中。我选择了一个文件后，等了一小会，返回了文件访问页面。

![](img/a93b0645c410106077f93cdb6b96dbb8.jpg)

说明，我的文件共享到 ipfs 网络了。

图片下的`上传选项`，可以根据自己需求处理。选项如下图：

![](img/2a133f1b296766201db25c668a0333b5.jpg)

3.再点一下 ipfs companion 扩展应用，会出现 ![](img/13f2b3459084342594b3e05c16e017b8.jpg)

会发现多了一些内容。你可以复制共享文件的网关地址，保存起来；共享的文件在本地存储是锁定的，不会被系统清理掉，你可以解除锁定。

这些操作整体就是 ipfs 在终端的操作，现在是图形户操作，更亲民了。

### 第四步：检查本地 ipfs 节点的信息

点击`打开 Web UI`进入本机 ipfs 节点的信息界面 ![](img/6b4af0dcb31dbcae71bbab34dd734ce9.jpg)

页面显示了你存储了多少数据，以及连入的节点数，和动态地展示本机带宽的速度。 拉开`Advanced`数据，会显示本机节点的网关、API、公钥和地址信息。

### 第五步：管理文件

点击图中的`文件`。这里就像我们的电脑存储盘操作类似，文件可以拖拽改变保存位置，可以新建文件夹，打开文件。可以将电脑中文件和文件夹添加到 ipfs 网络中。可以通过 hash 值搜索文件。

![](img/58af1b332ab84c14f81331c0edfbe450.jpg)

### 第六步：连入的节点

点击`Peers`出现你连接全球的节点列表。 ![](img/3bb694f4183c31aeef521ee8a1bcb7b1.jpg)

### 第 7 步：设置

点击`Settings`会展示默认的你的 config 信息，你可以进行修改，然后点击`Reset`重新设置，但是谨慎修改。

![](img/8db599c8efb99445a32426cfc3b1bb0a.jpg)

### 第八步：关闭

当你不想继续运行 ipfs companion 时，点击关闭图形即关闭了程序。