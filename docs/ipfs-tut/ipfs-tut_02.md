# 一、.1 教程 1:下载并安装 IPFS

## 教程目标

通过本教程的学习，你将学会：

*   在你的操作系统上下载并安装 IPFS
*   查看你正在使用的 IPFS 版本
*   查看 ipfs 命令清单

## 学习步骤

### 第一步：下载 IPFS 可执行文件压缩包

链接 IPFS 官方安装教程网址 https://docs.ipfs.io/introduction/install/ ，并根据你自己的系统安装 IPFS。

懒的打开链接同学，没关系，跟着我一步步来。 打开链接 https://dist.ipfs.io/#go-ipfs ，页面下图所示 ![](img/001cb674c67c753e117afd027001f1a4.jpg)

一般用户系统都是 amd64 处理器，所以你可以根据自己的操作系统，按照上图标记，下载对应的二进制文件压缩包。

下载完后你会发现，压缩包的名称是 go-ipfs 开头的，为什么呢？IPFS 源码实现的语言有 Golang 和 Javascript，分别是 go-ipfs 和 js-ipfs。而官方给出的二进制文件是基于 go-ipfs 编译的，所以下载下来的压缩包开头是 go-ipfs 了。

### 第二步：解压 IPFS 压缩包

Mac 和 Linux 下载下来的压缩文件后缀是.tar.gz，而 Windows 是 zip 类型的压缩文件。

你可以在自己的电脑上找到下载好的 IPFS 压缩包，双击解压。也可以在终端命令行输入命令解压，Mac 和 Linux 的命令如下： `tar xvfz go-ipfs_v0.4.18_darwin-amd64.tar.gz`

解压之后，你会得到一个 go-ipfs 的文件夹，里面的目录内容如下： ![](img/14b7f7c62d7fffab080a9a55d1357190.jpg) 在终端打开 go-ipfs，查看 ![](img/f04d98c54c4635d111a42e4b0ebede4a.jpg)

### 第三步：设置 ipfs 可执行文件的环境变量

Mac 和 Linux 系统，可以在命令行设置环境变量： ![](img/77d3a89a33fad25d6cd245c2353ce752.jpg)

Windows 系统，假如你的可执行文件 ipfs.exe 路径是 d:\go-ipfs\ipfs.exe，那么将该路径拷贝到 windows 系统目录，以便在任何目录中可以启动 ipfs.exe。

### 第四步：查看 IPFS 的版本

当遇到问题时，查看你当前使用的 ipfs 版本是很重要的，查看命令是：`ipfs version`。

### 第五步：查看 IPFS 帮助栏和命令行

如果你需要提醒怎么使用某些 ipfs 的命令，请输入命令： `ipfs help`

如果你想查看 ipfs 的所有命令，请输入命令： `ipfs commands`