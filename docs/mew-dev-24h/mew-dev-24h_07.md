# 第五章 【以太坊钱包开发 五】钱包项目整体架构设计

> 本课程是以太坊钱包开发，后端使用的 NodeJS 搭建，客户端使用的 web 前端，VSCode 开发工具，Ubuntu16.04 开发环境，node v8.11.3，npm v5.6.0。
> 
> 在 Kovan 测试网络上进行开发。

## 一、前端架构

咱们的开发重点是在后端实现上，因此为了让大家快速上手，web 客户端没有使用其它流行的框架，这里只使用了 jQuery 框架简化代码，另外还有个 jQuery Validate 插件简化了表单验证。

*   web 前端整体技术：

    **`html + css + javascript + jQuery`**。

*   web 前端功能：

    1.  新建账号
        *   提供 keystore 文件
        *   提供私钥
    2.  解锁账号
        *   通过私钥解锁
        *   通过 keystore+密码解锁
        *   通过助记词解锁
    3.  转账
        *   以太币转账
        *   Token 代币转账

![以太坊钱包](img/8238bc773e33b229d590fdaff6d08931.jpg)

## 二、后端架构

这个钱包应用程序与以坊节点进行交互，使用 web3.js 库提供的 jsAPI 访问以太坊区块链数据，因此我们用 NodeJS 搭建后端服务，使用成熟的 MVC 架构，http 框架是 koa，需用到如下第三方库：

*   koa：富有强大功能的 HTTP 中间件框架，使 Web 应用程序和 API 更易于编写。它的特点优雅、简洁、表达力强、自由度高。
*   koa-body：功能齐全的 koa body 解析器中间件。支持`multipart`，`urlencoded`和`json`请求体。
*   koa-router：koa 的路由中间件。
*   koa-static：静态文件服务器中间件。
*   koa-views：是模板渲染中间件，在模版引擎下使用，支持的模版引擎包含：ejs、jazz、haml、react 等。
*   ejs：是一种 JavaScript 模版引擎，可以动态的设置变量值到 html。需要与模板渲染中间件 koa-views 配合使用。
*   web3.js：以太坊 JavaScript API。
*   ethereumjs-tx：用于创建、操作和签名以太坊交易的模块。
*   bip39：随机产生新的 mnemonic code，并可以将其转成 binary 的 种子。
*   ethereumjs-util：Ethereum 的一个工具库。
*   ethereumjs-wallet：生成和管理公私钥，下面使用其中 hdkey 子套件来创建 HD 钱包。

## 三、项目初始化

新建项目跟文件夹 MyEtherWallet，然后按照如下步骤执行

```js
lixu@ubuntu:~$ cd '/home/lixu/Documents/demo/MyEtherWallet' 
lixu@ubuntu:~/Documents/demo/MyEtherWallet$ npm init
```

然后不断回车初始化项目。然后后自动生成`package.json`文件，是项目包的配置文件，下面我们引入项目中需要用到的库，拷贝下面 json 到`package.json`文件的最后一个字段。

```js
,
  "dependencies": {
    "bignumber": "¹.1.0",
    "ejs": "².6.1",
    "ethereumjs-tx": "¹.3.7",
    "koa": "².5.2",
    "koa-body": "⁴.0.4",
    "koa-router": "⁷.4.0",
    "koa-static": "⁵.0.0",
    "koa-views": "⁶.1.4",
    "web3": "¹.0.0-beta.35"
  }
```

项目的界面如下：

![71E261DF-7DE6-4E08-BFB1-AE20699780FD](img/e4eaf5be7ff7d8d5e094373071d376c8.jpg)

然后运行以下命令按照上面的依赖库。

```js
npm install
```

下载完成后会将所有的依赖库下载到项目根目录自动新建的`node_modules`文件夹。

## 四、项目源码

按照如下结构搭建项目。

![20C31EC8-B97D-484B-9FE8-6B9D4865F130](img/4a532ef94998b9ac3e674443cc349b84.jpg)

### index.js

项目的入口文件。首先实例化 koa 对象，然后将 koaBody、static、views、路由注册到中间件，服务绑定到 3000 端口。

```js
let koa = require("koa")
//通过 koa 创建一个应用程序
let app = new koa()
//导入./router/route 这个包，赋值给的 router 就是 ./router/router 导出的数据
let router = require("./router/router")
let static = require("koa-static")
let path = require("path")
let views = require("koa-views")
let koaBody = require("koa-body")

app.use(async (ctx, next) => {
    console.log(`${ctx.method} ${ctx.url} ..........`)
    await next()
})

//针对于文件上传的时候，可以解析多个字段
app.use(koaBody({multipart:true}))
//注册静态文件的库到中间件
app.use(static(path.join(__dirname, "static")))
//注册模板引擎的库到中间件
app.use(views(path.join(__dirname, "views"), {extension:"ejs", map:{html:"ejs"}}))
app.use(router.routes())

console.log("正在监听 3000 端口")
app.listen(3000)
```

### newaccount.html

前端：新建账号的页面。

```js
<html>

<head>
    <title>创建钱包</title>
    <script src="/js/lib/jquery-3.3.1.min.js"></script>
    <script src="/js/lib/jquery.url.js"></script>
    <script src="/js/wallet.js"></script>
    <link rel="stylesheet" href="/css/wallet.css">
</head>

<body>

    <%include block/nav.html%>

    <div id="main">
        创建账号
    </div>
</body>

</html>
```

### nav.html

前端的导航栏，使用`$("#nav").load("/html/nav.html")`方式载入。

```js
<div id="nav">
    <div id="nav-center">
        <ul>
            <li><a href="http://www.kongyixueyuan.com">孔壹学院</a></li>
            <li><a href="/account/new.html">新建账户</a></li>
            <li><a href="/transaction.html">转账</a></li>
        </ul>
    </div>
</div>
```

### myUtils.js

项目工具类，提供获取 web3 实例、返回给前端成功与失败的基本数据结构。

```js
module.exports = {
    getweb3: () => {
        let Web3 = require("web3")
        var web3 = new Web3(Web3.givenProvider || 'https://kovan.infura.io/v3/bc76......ca805');

        return web3
    },

    success: (data) => {
        responseData = {
            code:0,
            status:"success",
            data:data
        }
        return responseData
    },

    fail: (msg) => {
        responseData = {
            code:1,
            status:"fail",
            msg:msg
        }
        return responseData
    }
}
```

### wallet.js

前端唯一的 js 文件。

```js
$(document).ready(function () {
    alert("welcome!")
})
```

### wallet.css

前端唯一的 css 文件。

```js
#main{
    /*background-color: #8bc34a;*/
    margin: 100px 50px 50px 50px;

}
.error{
    color: red;
}
a{
    color: black;
    text-decoration: none;
}
a:hover{
    color: #666;
}
body{
    margin: 0px;
}
.global-color{
    color: #0abc9c;
}
a[class=button]{
    background-color: beige;
    padding: 2px 10px;
    border: 1px solid gray;
}

/*导航-------------------------------------------------------------------------------------------------------*/
#nav{
    display: flex;
    justify-content: space-between;
    background-color: #0abc9c;
    position: fixed;
    top: 0px;
    left: 0px;
    right: 0px;
}
#nav li{
    display: inline-block;
    margin: 10px 2px;
}
#nav ul{
    padding: 0px;
}
#nav a{
    padding: 10px;
    font-size: 24px;
}
#nav-left{
    margin-left: 20px;
}
#nav-right{
    margin-right: 20px;
}
```

### router.js

路由文件。

```js
let router = require("koa-router")()

router.get("/", async (ctx) => {
    await ctx.render("newaccount.html")
})

module.exports = router
```

## 五、项目运行效果

![2018-09-25 11.07.50](img/1e8e2f6fb9a0674d95e6b2990916cc33.jpg)

**参考资料**

koa 的 github：[`github.com/koajs/koa`](https://github.com/koajs/koa)

koa-views 的 github：[`github.com/queckezz/koa-views`](https://github.com/queckezz/koa-views)

koa-body 的 github：[`github.com/dlau/koa-body`](https://github.com/dlau/koa-body)

koa-router 的 github：[`github.com/alexmingoia/koa-router`](https://github.com/alexmingoia/koa-router)

koa-static 的 github：[`github.com/koajs/static`](https://github.com/koajs/static)

ejs 的 github：[`github.com/tj/ejs`](https://github.com/tj/ejs)

web3.js 的 github：[`github.com/ethereum/web3.js`](https://github.com/ethereum/web3.js)

ethereumjs-tx 的 github：[`github.com/ethereumjs/ethereumjs-tx`](https://github.com/ethereumjs/ethereumjs-tx)

BIP39 的 github：[`github.com/bitcoinjs/bip39`](https://github.com/bitcoinjs/bip39)

ethereumjs-wallet 的 github：[`github.com/ethereumjs/ethereumjs-wallet`](https://github.com/ethereumjs/ethereumjs-wallet)

ethereumjs-util 的 github：[`github.com/ethereumjs/ethereumjs-util`](https://github.com/ethereumjs/ethereumjs-util)

**[项目源码 Github 地址](https://github.com/lixuCode/MyEtherWallet)**

**版权声明：博客中的文章版权归博主所有，未经授权禁止转载，转载请联系作者（微信：lixu1770105）取得同意并注明出处。**