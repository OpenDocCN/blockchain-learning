# 6.7 用户订单管理设计与实现

# 订单功能设计与实现

本小节主要讲解如何设计与实现交易所中的买单、卖单功能。比如：如何定义订单数据结构、约束条件、与资金表订单薄之前有哪些级联操作等等。

## 功能介绍

订单功能：主要用于实时接收与存储用户买卖单请求并在搓合成功或取消订单时更新订单状态。

## 数据结构

| **用户订单表** | 字段 | 数据类型 | 说明 |
| --- | --- | --- | --- |
| 主键 id | uint64_t | 由系统生成 |
| 订单类型 | uint64_t | 订单类型：pairid*100+1 为买单；pairid*100+2 为卖单 |
| 订单薄 id | uint64_t |  |
| token 单价 | uint64_t |  |
| 总交易量 | eosio::asset |  |
| 未成交交易量 | eosio::asset |  |
| 备注 | string |  |
| 创建时间 | eosio: :time _point _sec |  |
| 订单状态 | int | 状态类型：完成成交；部分成交；取消订单； |

1.  **订单类型**， 之所以使用 pairid*100+1 或 pairid*100+2 的方式来定义。是为了将交易对信息、买/卖类型合在一起，可以巧妙的使用 EOS 智能合约的存储方式来模拟买、卖队列，从而解决订单薄数据存储及搓合排序的问题。

| **订单薄表** | 字段 | 数据类型 | 说明 |
| --- | --- | --- | --- |
| 主键 id | uint64_t | 由系统生成 |
| 用户订单 id | uint64_t | 关联用户订单，搓合成功后方便状态更新 |
| 订单量 | eosio::asset | 买入或卖出的兑换代币。 比如： 2311.2341 DICE |
| 订单单价 | uint64_t | 单个兑换代币的基准代币价格。 |
| 下单时间 | eosio : :time_point_sec |  |
| 备注 | string |  |

**订单单价**，在交易对功能设计与实现章节曾经介绍过关于价格精度字段的设置，所以在此处的值也是根据交易对配置来进行设置的。比如：0.128592 单价在存储时会转换为 128592, 即乘以 100000。

## 业务约束

*   是否满足交易对最小订单金额
*   订单金额是否小于等于帐户可用余额
*   订单金额是否大于 0
*   订单所属交易对是否存在

## 安全问题

*   权限验证
*   方法分级授权
    可根据管理角色进行细粒度方法权限授权，确保关键业务由高级别管理员控制。比如：对于资金的提现功能只允许最高级别管理员操作；而合约的基础数据操作可交由运维管理员进行操作。

## 技术实现

由项目成员自主设计与实现。

## 参考知识点

### 智能合约

订单类型生成规则

```js
uint64_t gen_buytype_bypair(uint64_t pair_id){
    return pair_id*100 + 1;
}

uint64_t gen_selltype_bypair(uint64_t pair_id){
    return pair_id*100 + 2;
}
```

订单薄队列的实现方式

```js
OrderIndex buyorderstable(_self, gen_buytype_bypair(pair_id));

OrderIndex sellorderstable(_self, gen_selltype_bypair(pair_id));
```

### 命令

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

通过本章节的学习以及动手实践，我们理解并掌握了对交易所整个用户买卖订单的内部实现业务逻辑并完成了整个订单功能(创建买单、创建卖单、)的代码实现。

* * *

> | 在教程中如出现错误🐛或不易理解的知识点，欢迎加我微信指正! Name: zhangliang | WeChat: rushking2009 | Mail: zhangliang@cldy.org |
> | --- | --- | --- |
> 
> **changelog** 2019-03-01 zhangliang \zhangliang@cldy.org\
> 
> *   初次发稿