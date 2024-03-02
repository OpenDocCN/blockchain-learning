# 七、.3 技术整合-EOSSDK（eosjs）

本小节将主要介绍在现有 Node 框架服务中，整合与链端的服务交互。比如：如何访问链上数据、如何对合约中的数据进行不同字段的排序与检索以及如何对执行方法的签名与提交等。

* * *

`eosjs`是一个基于 EOS 链节点 HTTP 接口服务封装的 javascript 语言版本 SDK 实现。 到目前为止， 仅提供了该语言实现。当然，我们也可以根据节点所提供的[RPC API](https://developers.eos.io/eosio-nodeos/reference)进行不同语言版本的实现。

本项目将适用于 eosjs@ 16.0.9 版本，所以在安装时需经指定对应版本。

**A. 安装**

```js
npm install eosjs@16.0.9 or yarn add eosjs@16.0.9
```

**B. 集成示例**

```js
//1\. 引入 sdk 文件
const Eos = require('eosjs')

//2\. 实例化 sdk
let eos = Eos({
    chainId: null, //链 id。用于区分不同链网络，正式网与本地测试该值是不同的
    keyProvider: ['PrivateKeys...'], //签名私钥列表，主要用这些私钥进行方法签名
    httpEndpoint: 'http://127.0.0.1:8888', //区块链节点服务地址，可自己搭本地节点也可以使用其他超级点
    expireInSeconds: 60,    //交易过期时间
    broadcast: true,        //是否广播
})

／/3\. 提交交易
eos.transaction({
    actions: [{
        account: <合约帐户|e.g. hackdappexch>,
        name: <合约方法|e.g. execute_trade>,
        authorization: [{
            actor: <合约帐户|e.g. hackdappexch>,
            permission: <合约授权用户|e.g. hackdappoper@autotrade>
        }],
        data: <合约方法参数>,
    }, ],
})
```

关于 Eos 实例化时所用到的更多可选参数，可从以下链接中进行详细了解 [`github.com/EOSIO/eosjs/blob/v16.0.9/README.md#configuration`](https://github.com/EOSIO/eosjs/blob/v16.0.9/README.md#configuration)

**C. 使用示例**

1.  查询合约数据
    在实际业务的场景中，可能我们需要查询链上数据进行一些业务处理。比如：查询用户的未成交订单或历史成交订单列表，或根据 ID 查询单个订单信息。

    ```js
    eos.getTableRows({
        json: true,
        code: <合约帐户>,
        scope: <数据范围|可自由设定义，可以为合约帐户信息也可以为其他数值>,
        table: <合约表>, 
        key_type: <索引字段类型|primarykey 为 1, 根据自定义索引键依次排序>,
        index_position: <按表中哪个字段进行索引查询或排序>,
        limit: <查询数量限制>
    })
    ```

2.  执行合约方法（方式一）

    ```js
    eos.transaction({
        actions: [{
            account: <合约帐户|e.g. hackdappexch>,
            name: <合约方法|e.g. execute_trade>,
            authorization: [{
                actor: <合约帐户|e.g. hackdappexch>,
                permission: <合约授权用户|e.g. hackdappoper@autotrade>
            }],
            data: <合约方法参数>,
        }, ],
    }) 
    ```

3.  执行合约方法（方式二）

    ```js
    // @returns {Promise}
    eos.contract([contract_name], [options], [callback])

    // 单个方法执行
    eos.contract("hackdappexch").then(mycontract=>{
        mycontract.hi("hackdapp")
    })

    // 多个方法执行
    eos.transaction(['合约一','合约二'], (contract1, contract2) => {
        contract1.hi(..)
        contract2.hi(..)
    }) 
    ```

4.  查询 EOS 帐户余额

    ```js
    eos.getCurrencyBalance([contract_name], [contract_name], [TOKEN])

    eos.getCurrencyBalance('myaccount', 'myaccount', 'PHI')
    ```

**D. 在线文档** [`eosio.github.io/eosjs/`](https://eosio.github.io/eosjs/) [`developers.eos.io/eosio-nodeos/reference`](https://developers.eos.io/eosio-nodeos/reference)

**E. 下载地址** [`eosio.github.io/eosjs/`](https://eosio.github.io/eosjs/)

* * *

通过本小节的学习、思考与动手实践， 我们学会了如何实例化 eossdk，如何通过 sdk 访问合约数据，查询用户余额以及交易签名等操作。

* * *

> 在教程中如出现错误🐛或不易理解的知识点，欢迎加我微信指正! Name: zhangliang | WeChat: rushking2009 | Mail: zhangliang@cldy.org

![Show me your code.](img/9c507c40d372f5692d061c802a44deb2.jpg "加群了解")![](img/aab6c923225b0a35b6580de17534641d.jpg)

注： 有想了解**愿码全思维 IT 工程师加速器**的朋友，可以扫码加群咨询。

* * *

### **changelog**

2019-03-04 zhangliang

*   初次发稿