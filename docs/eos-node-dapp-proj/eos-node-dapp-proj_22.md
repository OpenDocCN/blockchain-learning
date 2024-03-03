# 7.4 搓合定时调度功能设计与实现

上一小节介绍了如何利用 eosjs 与链进行数据交互；那么本小节将主要介绍如何定时触发调度检查买卖订单薄是否存在搓合订单以及如何触发搓合方法等。

* * *

## 功能介绍

搓合调度功能，主要是通过 Node 调度服务每隔 x 分钟查询智能合约所有交易对的订单薄中买卖队列中是否存在可以搓合的订单，当发现有符合条件的订单时，自动执行搓合方法进行搓合逻辑处理，并将搓合后的订单明细上链。

## 调度配置

调度实现是由 node-schedule 组件进行实现的，该组件支持 cron 表达式配置。

## 权限要求

考虑到智能合约的资金和业务数据的安全性，需要对智能合约的方法进行分级权限管理。所以对于执行调度的私钥只授权智能合约搓合方法的权限。

## 技术实现

对于该功能主要分为两部分实现：

1.  通过 eosjs 按照价格优先时间优先的策略对订单薄买卖订单进行排序并检查是否存在可搓合订单。 至于排序的实现可以借用智能合约索引器的特点进行实现。
2.  对可搓合的订单进行链上搓合。对于搓合数据上链的实现可能通过调用合约方法的方法进行链上数据存储。比如：

    ```js
    auto logdata = assemmblelog(pair_id, price_precision, precision, fill_quote_quantity, sellprice, ......);       
    action(
        permission_level{ _self, "active"_n },
        _self, "log"_n,
        std::make_tuple(logdata)
    ).send();
    ```

另外，在调度的过程中，需要对调度任务进行加锁处理，防止重复检查调用问题的出现。如果要进行多机部署的化，需要考虑分布式锁的实现。

如果还需要将功能进行分离部署，比如：将报表接口服务、调度服务进行单独部署，那么需要在启动时需要支持参数化的配置。

## 相关知识点

[Cron Expression Generator & Explainer - Quartz](https://www.freeformatter.com/cron-expression-generator-quartz.html)

* * *

通过本小节的学习、思考与动手实践，我们完成了搓合订单匹配功能、提交搓合交易、定时调度以及成交订单上链功能。

* * *

> 在教程中如出现错误🐛或不易理解的知识点，欢迎加我微信指正! Name: zhangliang | WeChat: rushking2009 | Mail: zhangliang@cldy.org

![Show me your code.](img/9c507c40d372f5692d061c802a44deb2.jpg "加群了解")![](img/aab6c923225b0a35b6580de17534641d.jpg)

注： 有想了解**愿码全思维 IT 工程师加速器**的朋友，可以扫码加群咨询。

* * *

### **changelog**

2019-03-04 zhangliang \zhangliang@cldy.org\

*   初次发稿