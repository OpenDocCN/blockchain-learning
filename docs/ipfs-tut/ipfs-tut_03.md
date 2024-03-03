# 1.2 教程 2: 初始化你的 IPFS 资源库

## 教程目标

通过学习本教程，你能够：

1.  初始化你的本地 ipfs 存储库。
2.  在你的本地 ipfs 存储库查看存储的内容。
3.  打开 IPFS 配置文件

## 学习步骤

### 第一步：初始化存储库 repo

使用命令`ipfs init`初始化你的 ipfs 存储库，如下图显示： ![](img/174017e64c355a46047026c31afe99e3.jpg) 你会发现在你的电脑账户目录下，产生了一个隐藏文件./ipfs。同时还为你的本机产生了一个 IPFS 节点的 ID。这个./ipfs 就是你在 IPFS 网络中的存储库。你可以把数据块存储到./ipfs/blocks 下，共享给整个 ipfs 网络。

注：如果你之前已经初始化过 ipfs，再次执行`ipfs init`命令时，会出现以下内容， ![](img/6112e020f5fded1f56cd20b459441a42.jpg)

### 第二步：检查 IPFS 安装是否成功

我们在第一步初始化时，显示内容的最后提醒我们可以通过命令 ipfs cat /ipfs/QmS4ustL54uo8FzR9455qaxZwuMiUhyvMcX9Ba8nUH4uVv/readme 检查 IPFS 安装是否成功，如果执行该命令，出现下图，则说明安装成功。 ![](img/mark)

我们还可以使用命令 ipfs cat /ipfs/QmS4ustL54uo8FzR9455qaxZwuMiUhyvMcX9Ba8nUH4uVv/security-notes 查看 ipfs 使用须知。 ![](img/4278eefeb3705714e5f3649e3ffd3ed1.jpg)

### 第三步：找到本机作为 ipfs 节点，其存储库的位置

ipfs 会将本地的对象存储在~/.ipfs 中。

该目录下的内容显示如下： ![](img/e079404c574860ad42f6d840f2bbbd7e.jpg)

每个 IPFS 节点上传的数据和下载的数据，都会存储在该目录下。例如上文中的 QmS4ustL54uo8FzR9455qaxZwuMiUhyvMcX9Ba8nUH4uVv/readme 文件就存储在这里。你还可以使用 grep 命令查找文件的具体位置。

### 第四步：打开 IPFS 的配置文件

ipfs 存储库的配置信息通常存储在的 json 文件中~/.ipfs/config。要查看当前配置，请运行： `ipfs config show`。

此配置文件中的一个有用的详细信息是 Datastore.Path。它指定了你 ipfs 存储的内容存储在哪里。正如我们在第 3 步中看到的那样，通常路径是~/.ipfs。