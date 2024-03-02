# 七、.7 实时币价功能设计与实现

本小节将主要介绍如何实现当前交易所所有交易对币价的 http 数据查询，比如：当前价格、最高价、最低价以及 24 小时成交量等信息。除此之外，还需要实现对交易所展示页面实时币价的消息推送。

* * *

## 功能介绍

实时币价功能, 主要用于展示交易所所有交易对当前的市场行情，并根据市场交易成交记录实时更新前端页面中的交易对价格及成交量等信息。

## 接口服务

### 查询交易对价格

查询当前时间所有交易对的币价、涨跌幅、成交量等信息。

**触发条件** 页面首次加载时，加载所有交易对价格信息。

**数据接口**

*   **接口地址:** `/api/price/query`
*   **请求参数**
    无
*   **提交方式：** `get`
*   **响应结果：**

    ```js
    {
        "result": "ok",
        "data": [
            [1, 0.0049, -0.0585, 4371833.4722, 0.005445, 0.004669], //交易对 id、价格、涨幅百分比、成交量(24H)、高(24H)、低(24H)
            [... ...]
        ]
    }
    ```

### 推送交易对实时价格

实时推送变化交易对的币价、涨跌幅、成交量等信息。

**触发条件** 当市场有新成交订单时，触发变化交易对价格推送。

**数据接口**

*   **接口地址(TOPIC)** `price.update`
*   **请求参数** 无
*   **响应结果：**

    ```js
    {
        "method": "price.update",
        "data": [
            //交易对 id、价格、涨幅百分比、成交量(24H)、高(24H)、低(24H)
            [1, 0.0049, -0.0585, 4371833.4722, 0.005445, 0.004669], 
            [... ...]
        ]
    }
    ```

## 技术实现

1.  查询每一个交易对最后一个成交订单的价格
2.  在同步区块及报表数据后，增量推送该交易对的实时价格至前端

* * *

通过本小节的学习、思考与动手实践，我们完成了对外交易对实时币价的接口查询以及 socket 数据推送。

* * *

> 在教程中如出现错误🐛或不易理解的知识点，欢迎加我微信指正! Name: zhangliang | WeChat: rushking2009 | Mail: zhangliang@cldy.org

![Show me your code.](img/9c507c40d372f5692d061c802a44deb2.jpg "加群了解")![](img/aab6c923225b0a35b6580de17534641d.jpg)

注： 有想了解**愿码全思维 IT 工程师加速器**的朋友，可以扫码加群咨询。

* * *

### **changelog**

2019-03-04 zhangliang \zhangliang@cldy.org\

*   初次发稿

2019-03-05 zhangliang

*   数据接口完善