# 第九章 【分布式存储系统 etcd】etcd-manage 项目介绍

# etcd-manage 项目

本项目受到[e3w](https://github.com/soyking/e3w)启发

项目旨在于方便开发者简单明了的查看、添加、修改和删除存储于 etcd 中的配置。

相比于 e3w 本程序运行简单，已将 ui 静态文件打包到了可执行文件中只需在终端运行此程序即可自动打开浏览器访问子集的 etcd。

**与 e3w 好的地方**

1.  支持多 etcd 服务，更适用于开发者日常开发场景，一般都会有开发环境、测试环境和线上环境的不同切换 etcd 服务端。
2.  支持 Basic Auth 用户验证登录，可以通过配置文件添加用户并设置用户角色实现用户可访问 etcd server 的空。

**Demo**

demo 地址： [Demo](http://140.143.234.132:10280/ui/#/keys)

用户名：admin

密码：123456

## 项目效果图

首先先启动一个集群，还是像之前一样，一个 Mac 系统，以及两个 Ubuntu 虚拟机。如下图：

![etcd_xiangmu1](img/db62ad9b886109f0b2c6c2936cbff741.jpg)

然后再 mac 下，打开 goland 的终端，编译并运行程序：

```go
localhost:etcd-manage ruby$ go build -o etcd-manage ./
localhost:etcd-manage ruby$ ./etcd-manage 
```

启动程序：

![etcd_xiangmu2](img/c37424ea20bc4e09858cb8bff33dc80b.jpg)

然后会自动的打开浏览器：

![etcd_xiangmu3](img/65bd071a81149fadd44bbef5561ace0f.jpg)

可以点开看看，我们启动了 3 个客户端：

![etcd_xiangmu5](img/6e608ac9fc428ed232cd2d0c84711a17.jpg)

也有三个服务：

![etcd_xiangmu6](img/fff87782e85c23de076718972008bb02.jpg)

先不急着操作，可以在 Ubuntu 中也运行这个程序：

![etcd_xiangmu4](img/1c2bb1d95ba355429a74f30bd172740f.jpg)

也会打开浏览器，接着我们在 mac 下的浏览器上进行操作，先点击 Add 按钮：

![etcd_xiangmu7](img/fef7ce1905b2051a736eeacaae29fb47.jpg)

我们先添加一个 dir：

![etcd_xiangmu8](img/b77bc213de7cbe2567f2c077098148cf.jpg)

然后再添加一个 key：

![etcd_xiangmu9](img/6bce037671d0da64b137001ee438e7c2.jpg)

mac 下效果如下：

![etcd_xiangmu10](img/7d1856339e9b7b869cf939127302882b.jpg)

然后我么在 Ubuntu 下刷新浏览器：

![etcd_xiangmu11](img/af8282b55c0b3029a030f4a1568ad5bd.jpg)

然后在 Ubuntu 中选中这两个文件，点击删除按钮：

![etcd_xiangmu13](img/244f1a919aaacc8fe3e5f0250464f719.jpg)

删除后效果如下：

![etcd_xiangmu14](img/960efce5a075bd863779b6588e60ec9f.jpg)

然后我们回到 Mac 下，刷新浏览器：

![etcd_xiangmu15](img/7d35356d3bab7b68b74c450f5ec86eff.jpg)

可以点开看一下日志：

![etcd_xiangmu12](img/8e1f574329da84b3367b4d2c817fae95.jpg)

[源代码](https://github.com/rubyhan1314/myetcd-manage)