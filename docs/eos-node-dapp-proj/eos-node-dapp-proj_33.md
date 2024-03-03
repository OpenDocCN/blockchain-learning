# 9.4 权限配置与合约发布

# 合约发布与权限配置

本小节将介绍如何将智能合约发布至线上正式环境以及为了保证交易所的资金及业务的正常安全运行，我们将通过权限分级的方式隔离在程序中运行的私钥运行级别。

* * *

在开始操作之前， 需要我们提前准备四个帐号：合约帐号(管理员)、运维帐号、调度帐号。

## 新建本地钱包

因为整个权限配置的过程，都是通过命令窗口的形式进行操作的，所以为了操作的方便，我们需要创建一个钱包，并导入前面准备的三个帐号私钥。

**首先**，在本地创建钱包，并将生成的钱包密码保存至对应的文件当中；后续解锁钱的时候需要用到该密码，所以必须妥善保管。

注： 市面上 99.99%的资金丢失其实都是对于私钥的保管不当所导致的。

```js
cleos wallet create -n mypocket --to-console | tail -1 | sed -e 's/^"//' -e 's/"$//' > mypocket_wallet_password.txt
```

**然后**，导入私钥

```js
cleos wallet import -n mypocket --private-key <私钥>
```

## 合约发布

```js
cleos set contract <合约名> "<合约 abi/wasm 文件存储目录>" -p <合约名>@active
```

示例：发布交易所合约 hackdappexch 至公链环境。

```js
cleos set contract hackdappexch "contracts/dexchange/" -p hackdappexch@active
```

注意： 执行发布命令时，需要解锁钱包。

## 权限配置

为了保证系统的正常运行及合约安全，所以我们需要对合约的权限进行分级隔离，即将合约中的方法按照资金、基础数据、搓合业务分别映射给管理员、运维、调度帐号。

### 建立权限组

```js
cleos set account permission <合约名称> <权限组> '{"threshold" : 1, "keys": [], "accounts":[] }' active -p <合约名称>@active
```

示例：为合约(hackdappexch)创建一个名为 auth.trade 的权限组.

```js
cleos set account permission hackdappexch auth.trade '{"threshold" : 1, "keys": [], "accounts":[] }' active -p hackdappexch@active
```

在执行命令时，需要注意钱包是否处于解锁状态。

### 权限方法映射

```js
cleos set action permission <合约名称> <合约名称> <合约方法> <权限组>  -p <合约名称>@active
```

示例： 将合约(hackdappexch)中的方法(executetrade)映射到 auth.trade 权限组中。

```js
cleos set action permission hackdappexch hackdappexch executetrade auth.trade  -p hackdappexch@active
```

### 授权

将合约中的某个权限组授权给某个帐号或地址。 1）授权给帐号

```js
cleos set account permission <合约名称> <权限组> '{"threshold" : 1, "keys": [], "accounts":[{"permission":{"actor":"<EOS 帐号>","permission":"active"},"weight":1}] }' active -p <合约名称>@active
```

2）授权给地址

```js
cleos set account permission <合约名称> <权限组> '{"threshold" : 1, "keys": [{"key":"<EOS 地址>","weight":1}], "accounts":[] }' active -p <合约名称>@active
```

示例：将交易所合约中的搓合权限组授权给 EOS 地址`EOS7H8xqsUyAwCPDYfQ5RQYSFKzxeX5cuucMLAC6g31GuQEG9hdKz`。

```js
cleos set account permission hackdappexch auth.trade  '{"threshold" : 1, "keys": [{"key":"EOS7H8xqsUyAwCPDYfQ5RQYSFKzxeX5cuucMLAC6g31GuQEG9hdKz","weight":1}]}' active -p hackdappexch@active
```

* * *

通过本小节的学习、思考与动手实践，我们熟悉并完成了合约的发布、权限定义及授权的整个流程与操作。

* * *

> 在教程中如出现错误🐛或不易理解的知识点，欢迎加我微信指正! Name: zhangliang | WeChat: rushking2009 | Mail: zhangliang@cldy.org

![Show me your code.](img/9c507c40d372f5692d061c802a44deb2.jpg "加群了解")![](img/aab6c923225b0a35b6580de17534641d.jpg)

注： 有想了解**愿码全思维 IT 工程师加速器**的朋友，可以扫码加群咨询。

* * *

**changelog** 2019-03-06 zhangliang

*   初次发稿