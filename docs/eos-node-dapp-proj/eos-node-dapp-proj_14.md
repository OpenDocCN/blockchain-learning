# 6.5 系统配置管理设计与实现

# 系统配置功能设计与实现

本小节主要讲解交易所所有业务逻辑中需要用到的全局配置数据定义与维护。比如：对交易所进行业务升级时锁定交易所，暂停所有业务操作；接收手续费的帐户等。

## 功能介绍

系统配置功能：主要用于维护交易所整体业务所需的全局配置参数定义。

示例：

*   交易所准备升级维护时，可能通过修改系统配置中的运行状态字段即可。

## 数据结构

| 字段 | 数据类型 | 说明 |
| --- | --- | --- |
| 运行状态 | bool | 0: 运行中; 1: 已暂停。 缺省为 0 |
| 手续费系统接收帐户 | eosio::name | 交易所自己的帐户，用于统一接收手续费 |

注: `eosio::name`为 eoslib 提供的数据类型。值内容字符长度不允许超过 12 位。

## 业务约束

*   仅限合约管理员操作

## 安全问题

*   权限验证
*   方法分级授权
    可根据管理角色进行细粒度方法权限授权，确保关键业务由高级别管理员控制。比如：对于资金的提现功能只允许最高级别管理员操作；而合约的基础数据操作可交由运维管理员进行操作。

## 技术实现

由项目成员自主设计与实现。

注：因为系统管理并不像其它功能需要存储多条记录，所以可以采用单例的方式来获取系统参数。

## 参考知识点

### 单例存储与查询

```js
# 示例: 
//定义单例
typedef singleton< "exchangecfg"_n, configstruct>  CfgHelper;

//实例化
CfgHelper cfg_helper(_self, _self.value);

//赋值
configstruct cfg = configstruct();
cfghelper.set(cfg, _self);

//查询
cfghelper.get();
```

### 命令方法

*   查看 Docker 服务

    ```js
    docker ps
    ```

*   进入 Docker 命令窗口

    ```js
     docker exec -it <container name> /bin/bash
    ```

*   解锁钱包

    ```js
    cleos wallet unlock -n <钱包名称> --password <钱包密码>

    # 例如: 打开名称为 hackdappexch 的钱包
    cleos wallet unlock -n hackdappexch --password $(cat hackdappexch_wallet_password.txt);
    ```

*   发布合约

    ```js
    cleos set contract <合约名> <合约目录> -p <合约名>@active
    ```

*   调用合约方法

    ```js
    cleos push action <合约名> <合约方法> [<参数 1>,<参数 2>] -p <合约名>@active
    ```

*   查询数据表

    ```js
    cleos get table <合约名> <scope> <表定义>
    # scope 值取决于实例化索引时所传入的值, 一般实例化索引时传入 _self,即合约本身。
    ```

* * *

通过本章节的学习、思考以及动手实践， 我们完成了系统配置的数据定义给维护工作。

* * *

> 在教程中如出现错误🐛或不易理解的知识点，欢迎加我微信指正! Name: zhangliang | WeChat: rushking2009 | Mail: zhangliang@cldy.org

* * *

**changelog** 2019-03-01 zhangliang \zhangliang@cldy.org\

*   初次发稿