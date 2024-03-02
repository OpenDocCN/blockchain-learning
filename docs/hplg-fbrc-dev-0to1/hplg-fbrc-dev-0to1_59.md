# 十三、.6 项目交互演示

### 13.6.1 启动 Web 服务

最后编辑 `main.go` ，以便启动 Web 界面实现 Web 应用程序

```go
$ vim main.go 
```

添加如下内容:

```go
import(
    [......]
    "github.com/kongyixueyuan.com/education/web/controller"
    "github.com/kongyixueyuan.com/education/web"
)

func main(){}
    [......]

    app := controller.Application{
        Setup: &serviceSetup,
    }
    web.WebStart(app)
} 
```

应用项目开发完成后，可以直接启动用来查看效果。在命令提示符中输入 `make` 命令：

```go
$ make 
```

### 13.6.2 访问页面

项目启动成功之后，打开浏览器访问: [htt://localhost:9000/](http://localhost:9000/)

根据访问的 URL 地址系统自动响应登录页面

![login](img/179a326e43a4cb542e88b7d30874e28f.jpg)

输入管理员账号及密码登录验证成功，则进入系统首页面

![login](img/fea0dff552bac4ca469148c6301e0dba.jpg)

在首页面中点击 `查询范围`链接，进入 `help`页面，

![login](img/90221ebccff6055341b9f337ccc7ee3f.jpg)

点击添加学历信息链接进入，添加学历信息页面

![login](img/2daebce87df5c81ce2a6f813478db91e.jpg)

根据学历证书编号与姓名查询页面

![login](img/b612d8eebfe2e91a2367f354b428e5e9.jpg)

根据学历证书编号与姓名查询结果页面

![certNoAndName](img/f9c5479b27de93c175610ac9a1478939.jpg)

根据身份证号码查询页面

![login](img/3befc95f5ffe1648f017446c6d0218fe.jpg)

根据身份证号码查询页面查询结果页面

![login](img/f15c4f8c84b831cde1eac09b80ceefa0.jpg)

编辑页面

![login](img/cb3cc74c0d97db7af1d202b0275b9474.jpg)

编辑成功自动跳转到根据身份证号码查询结果页面

![login](img/b96c5d2004a226ac029ef994b361b85e.jpg)

项目完整源代码，请 [点击此处](https://github.com/kevin-hf/education)