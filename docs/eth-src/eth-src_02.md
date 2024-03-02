# 第二章 源码环境配置

# 01\. Go 环境配置

Go 官网: [`golang.org`](https://golang.org "https://golang.org")

## Linux

```go
# 1\. 下载 go 安装包
> cd /usr/local
> wget https://dl.google.com/go/go1.11.1.linux-amd64.tar.gz
> tar -zxvf go1.11.1.linux-amd64.tar.gz

# 2\. 配置环境变量
> echo "export PATH=\$PATH:/usr/local/go/bin" >> /etc/profile
> source /etc/profile
> echo "export GOPATH=~/go" >> ~/.bash_profile
> source ~/.bash_profile 
```

## Window

下载地址: [`dl.google.com/go/go1.11.1.windows-amd64.msi`](https://dl.google.com/go/go1.11.1.windows-amd64.msi)
安装文件后，需要自行在环境变量中配置 PATH 及 GOPATH 两个变量

## Mac

```go
# 1\. 安装 go 包
brew install go

# 2\. 配置环境谈量
> echo "export GOPATH=~/go" >> ~/.bash_profile
> source ~/.bash_profile 
```

or
下载地址: [`dl.google.com/go/go1.11.1.darwin-amd64.pkg`](https://dl.google.com/go/go1.11.1.darwin-amd64.pkg)

参考资料：

1.  [官网下载链接](https://golang.org/dl/)

# 02\. 工程配置

## 克隆工程

```go
# 1\. 创建式目录
> mkdir -p $GOPATH/src/github.com/ethereum
> cd $GOPATH/src/github.com/ethereum

# 2\. 克隆以太坊源码
> git clone --depth 1 --branch release/1.8.17 https://github.com/ethereum/go-ethereum 
```

## 编译工程

```go
> cd go-ethereum
> make all 
```

## 导入编辑器

![](img/a082084beabd1cab20556f08bc92591e.jpg)