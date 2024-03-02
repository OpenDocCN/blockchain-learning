# 九、.3 服务器组件配置与安装(Mysql/Nginx/PM2)

本小节将主要介绍如何配置项目所依赖的各项服务组件。比如：如何通过反向代理 web 服务、myqsl 安装与权限配置。

* * *

本次我们所使用的服务器为阿里云 ECS 服务器实例，操作系统为 Ubuntu 16.04.5 LTS。

## 一、数据库 Mysql

数据库作为项目的基础服务组件，主要用于存储链上成交订单数据以及不同维度的报表数据，以此作为链和页面的数据缓存服务。一方面可以提高用户使用体验，一方面提供中间缓存服务，当链上数据不可用或时效较慢时，能够保证服务的可用性。

### 1\. 安装软体

```js
sudo apt update 

sudo apt install mysql-server
```

### 2\. 配置服务权限

此命令主要用于配置数据库一些安全事项。比如设置关闭对外访问权限，更改密码策略等

```js
mysql_secure_installation
```

### 3\. 创建数据库

```js
CREATE DATABASE IF NOT EXISTS <数据库名> default charset utf8 COLLATE utf8_general_ci;
```

### 4\. 创建数据库帐户

```js
CREATE USER '<帐户>'@'localhost' IDENTIFIED BY '<密码>';
```

### 5\. 设置帐户权限

```js
GRANT ALL ON <数据库>.* TO '<帐户>'@'localhost' WITH GRANT OPTION;

FLUSH PRIVILEGES

show grants for '<帐户>'@'%';
```

### 6\. 测试

```js
> mysql -u <帐户> -h localhost -p 
> show databases; 
```

## 二、Nginx

Nginx 作为系统中必不可少的一个组件。在实际应用场景中，主要利用 nginx 的反向代理及负载均衡来保证整体项目服务的可用性。

### 1\. 安装

```js
> sudo apt update
> sudo apt install nginx
```

服务安装后之后，nginx 安装目录如下:

```js
.
├── conf.d
│   └── default.conf
├── fastcgi_params
├── koi-utf
├── koi-win
├── mime.types
├── modules -> /usr/lib/nginx/modules
├── nginx.conf
├── scgi_params
├── uwsgi_params
└── win-utf
```

后续配置我们将会在 conf.d 目录下进行文件修改。

### 2\. 检查 nginx 服务状态

```js
> systemctl status nginx
● nginx.service - A high performance web server and a reverse proxy server
   Loaded: loaded (/lib/systemd/system/nginx.service; enabled; vendor preset: enabled)
   Active: active (running) since Fri 2018-04-20 16:08:19 UTC; 3 days ago
     Docs: man:nginx(8)
 Main PID: 2369 (nginx)
    Tasks: 2 (limit: 1153)
   CGroup: /system.slice/nginx.service
           ├─2369 nginx: master process /usr/sbin/nginx -g daemon on; master_process on;
           └─2380 nginx: worker process
```

如上图所示， 如果 Active 为 active 则表明 nginx 已经安装并运行成功。

同样，我也可以通过浏览器访问, `http://locahost`， 检查服务是否可用。 ![](img/e0cd5fd310919332caaf8e41ac86cc89.jpg)

### 3\. 服务配置

**1\. 配置负载均衡** 为了保障服务的可用性，我们针对接口服务可以提供两台以上服务实例，这样即使一台突然宕机， 也不会影响到整个应用的运行。

编辑**/etc/nginx/conf.d/default.conf**配置文件，添加以下代码：

```js
upstream myapi_server {
  server 192.169.0.1:4381 weight=3;
  server 192.169.0.2:4381 weight=3;
  server 192.169.0.3:4381 weight=3;
} 

server {
  listen          80;
  server_name     api.dappgame.io;

  location / {
      proxy_pass      http://myapi_server;
      proxy_set_header X-Real-IP $remote_addr;
  }
}
```

**upstream**中的多机 server 其实是在不同服务器上部署的同一程序代码，且可以根据每个机器的性能设置其权重以提高或降低命中率。

当大量服务请求访问我们的 web 服务时，首先会访问到 nginx 服务，然后根据权重配置反向代理到不同服务器应用服务上，从而达到负载均衡的目的，也提高了整体系统的可用性。

**2\. 配置 web 服务**

同样 nginx 本身也一个 web 服务器，相比 apache，它更加轻量且占用很少的内存及资源。

编辑**/etc/nginx/conf.d/default.conf**配置文件，添加以下代码：

```js
server {
    listen      80;
    server_name dappgame.io;
    root        /data/www;

    location / {
        index   index.html index.php;
    }

    location ~* \.(gif|jpg|png)$ {
        expires 30d;
    }
}
```

**listen**：对外访问端口； **server_name**：对外访问域名地址，通过此项设置可以达到同一台服务器多个 web 程序复用同一 80 端口的需求。如以下示例配置：

```js
server {
    listen      80;
    server_name api_v1.dappgame.io;
    root        /data/www_v1;

    location / {
        index   index.html index.php;
    }

    location ~* \.(gif|jpg|png)$ {
        expires 30d;
    }
}

server {
    listen      80;
    server_name api_v2.dappgame.io;
    root        /data/www_v2;

    location / {
        index   index.html index.php;
    }

    location ~* \.(gif|jpg|png)$ {
        expires 30d;
    }
}
```

* * *

通过本小节的学习、思考与动手实践，我们完成了数据库 mysql 的安装及用户配置， 以及数据库实例的初始化工作。除此之外，还安装了 nginx 系统服务，并在此基础上配置了 web 服务及负载均衡。

* * *

> 在教程中如出现错误🐛或不易理解的知识点，欢迎加我微信指正! Name: zhangliang | WeChat: rushking2009 | Mail: zhangliang@cldy.org

![Show me your code.](img/9c507c40d372f5692d061c802a44deb2.jpg "加群了解")![](img/aab6c923225b0a35b6580de17534641d.jpg)

注： 有想了解**愿码全思维 IT 工程师加速器**的朋友，可以扫码加群咨询。

* * *

### **changelog**

2019-03-11 zhangliang

*   初次发稿