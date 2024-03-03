# 6.2 TOKEN 管理--模块设计

# TOKEN 管理

本小节主要分析与介绍交易所合约中 TOKEN 管理业务模块的数据结构、业务逻辑判断、约束条件、操作权限以及可能存在的潜在安全漏洞等问题。

* * *

## 功能介绍

TOKEN 管理： 主要用于负责对交易所中所支持 TOKEN 的基础数据定义。比如，EOS、KARMA、DICE 等基于 EOS 链发布的 TOKEN。需要说明的是：该交易所只支持 EOSTOKEN，跨链 TOKEN 是不支持的，比如 ETH 等。

比如：EOS 的代币名称为 EOS，合约名称为 eosio.token。需要注意的是：代币名称相同，并不意味着是是相同币种；同时还需要判断是否为同一个合约地址。否则会导致充值到交易所中的是假的代币。

## 数据结构

| 字段 | 数据类型 | 说明 |
| --- | --- | --- |
| 主键 | uint64_t | 可由用户填写， 也可由系统自动生成 |
| token 名称 | string | 代币名称 |
| 合约名称 | extended_symbol | 合约名称，从 eoslib 库进行类型查询。举例: "symbol": "4,EOS", "contract": "eosio.token" |

## 业务约束

*   只允许合约管理员进行操作
*   不允许存在重复的币种名称
*   不允许存在重复的合约名称
*   删除 token 时需确保关联数据已经删除

## 权限要求

*   仅限合约管理员操作

## 安全问题

*   权限验证
*   方法分级授权
    可以将管理员根据业务不同进行细粒度权限授权。确保关键业务保留在权限高的管理员手中。比如：可以将修改 TOKEN 方法的管理员与管理资金的管理员进行权限切割，以达到资金的相对安全。

## 技术实现

由项目成员自主设计与实现。

## 涉及命令知识

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

通过本章节的学习思考与代码实现，我们完成了对交易所 TOKEN 的添加、修改、删除三个功能方法。

* * *

> 在教程中如出现错误🐛或不易理解的知识点，欢迎加我微信指正! Name: zhangliang | WeChat: rushking2009 | Mail: zhangliang@cldy.org