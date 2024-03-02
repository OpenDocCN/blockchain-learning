# 四、.2 web 层实现

## 从零到壹实现 Marbles 资产管理系统 （Fabric-SDK-Node）之－用户交互开发

### 指定依赖

打开命令提示符/终端并导航到 marbles 目录。

```go
$ cd $HOME/kevin-marbles 
```

打开 package.json 文件并编辑

```go
$ vim package.json 
```

在文件中添加 marbles 项目的依赖 pug，及 gulp，如下所示：

```go
[......]
    "dependencies"
        "pug": "2.0.3",

    },

    "devDependencies": {
        "gulp": "3.9.*",
        "gulp-sass": "*",
        "gulp-concat": "*",
        "gulp-clean-css": "*",
        "gulp-rename": "*"
    },
    [......]
} 
```

配置指定之后，使用 npm 命令安装指定依赖：

```go
$ npm install gulp -g
$ npm install 
```

### 创建 scss 目录

Sass 是一个将脚本解析成 CSS 的脚本语言，扩展了 CSS3，增加了规则、变量、混入、选择器、继承等等特性。Sass 生成良好格式化的 CSS 代码，易于组织和维护。学习地址请[点击此处](https://www.sass.hk/)

```go
$ mkdir scss 
```

添加相应的 scss 文件，详细请 [点击此处](https://github.com/kevin-hf/kevin-marbles/scss/)

### 创建 doc_images 目录

将应用所需图片保存在 doc-images 目录中，详细请 [点击此处](https://github.com/kevin-hf/kevin-marbles/doc-images/)

### 视图开发

Web 视图部分我们使用 Pug 来实现。

Pug 是一款健壮、灵活、功能丰富的模板引擎，专门为 Node.js 平台开发。

Pug 学习，请 [点击此处](https://pugjs.org/api/getting-started.html)

创建 views 目录

```go
$ mkdr views 
```

创建并编写相应的页面文件，详细请 [点击此处](https://github.com/kevin-hf/kevin-marbles/views/)

### 创建 public 目录

public 目录用于存放 CSS、图片及客户端脚本相关的文件，详细请 [点击此处](https://github.com/kevin-hf/kevin-marbles/public/)

### 设置路由

为了能让用户成功访问页面，需要创建一个路由，我们将代码封装在一个名为 site_router.js 的脚本文件中，并保存在 routes 的目录中，所以首先创建 routes 目录

```go
$ mkdir routes && cd routes 
```

在 routes 目录中创建一个名为 site_router.js 的文件并编辑

```go
$ vim site_router.js 
```

文件完整内容如下：

```go
'use strict';
/* global process */
/*******************************************************************************
 * Copyright (c) 2015 IBM Corp.
 *
 * All rights reserved.
 *
 *******************************************************************************/
var express = require('express');
var cachebust_js = Date.now();
var cachebust_css = Date.now();

module.exports = function (logger, cp) {
    var app = express();

    // ============================================================================================================================
    // Root
    // ============================================================================================================================
    app.get('/', function (req, res) {
        res.redirect('/home');
    });

    // ============================================================================================================================
    // Login
    // ============================================================================================================================
    app.get('/login', function (req, res) {
        res.render('login', { title: 'Marbles - Login', bag: build_bag(req) });
    });

    app.post('/login', function (req, res) {
        req.session.user = { username: 'Admin' };
        res.redirect('/home');
    });

    app.get('/logout', function (req, res) {
        req.session.destroy();
        res.redirect('/login');
    });

    // ============================================================================================================================
    // Home
    // ============================================================================================================================
    app.get('/home', function (req, res) {
        route_me(req, res);
    });

    app.get('/create', function (req, res) {
        route_me(req, res);
    });

    function route_me(req, res) {
        //if (!req.session.user || !req.session.user.username) {    // no session? send them to login
        //    res.redirect('/login');
        //} else {
        res.render('marbles', { title: 'Marbles - Home', bag: build_bag(req) });
        //}
    }

    //anything in here gets passed to the Pug template engine
    function build_bag(req) {
        return {
            e: process.error,    //send any setup errors
            config_filename: cp.config_filename,
            cp_filename: cp.config.cred_filename,
            jshash: cachebust_js,    //js cache busting hash (not important)
            csshash: cachebust_css,    //css cache busting hash (not important)
            marble_company: process.env.marble_company,
            creds: get_credential_data(),
            using_env: cp.using_env,
        };
    }

    //get cred data
    function get_credential_data() {
        const channel = cp.getChannelId();
        const first_org = cp.getClientOrg();
        const first_ca = cp.getFirstCaName(first_org);
        const first_peer = cp.getFirstPeerName(channel);
        const first_orderer = cp.getFirstOrdererName(channel);
        var ret = {
            admin_id: cp.getEnrollObj(first_ca, 0).enrollId,
            admin_secret: cp.getEnrollObj(first_ca, 0).enrollSecret,
            orderer: cp.getOrderersUrl(first_orderer),
            ca: cp.getCasUrl(first_ca),
            peer: cp.getPeersUrl(first_peer),
            chaincode_id: cp.getChaincodeId(),
            channel: cp.getChannelId(),
            chaincode_version: cp.getChaincodeVersion(),
            marble_owners: cp.getMarbleUsernames(),
        };
        for (var i in ret) {
            if (ret[i] == null) ret[i] = '';    //set to blank if not found
        }
        return ret;
    }

    return app;
}; 
```