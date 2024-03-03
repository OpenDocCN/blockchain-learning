# 2.5 DICE 游戏项目——前端接口

> 包含 websocket、HTTP、合约、TP API、Scatter API 多部分接口

## 一、websocket

### 所有投注

接口：newgames

方式：监听

备注：每一秒更新一次，若无新数据则不会推送消息

投注奖金算法：`value.amount * 0.985 / ((num-1) / 100.0)`

返回：数组，数组中的元素是游戏对象

**游戏对象数据结构**

```js
createtime: 1545017754  //时间
finish: 1               //后端是否已处理
id: 1                   //已完成的游戏 id
num: 72                 //小于该号码
refuser: "lixurecomm"   //推荐者
result: 0               //输赢，0:输、1:赢
result_num: 83          //开奖号码
seed: "seedQaZEUfO886yRz6YzEx" //前端种子
hashseed: "8df0bcd20897e169fedc4085a742a7ffecabcb9d40dcc0b1a2cab7ed6fd5ba4c"//后端种子
user: "user"            //投注者
value: "0.1000 EOS"     //投注金额
```

### 分红数据

数据包含：

*   英雄榜用户押注数据
*   分红所有代币的总额
*   英雄榜分红金额列表
*   代币质押总数

接口：bonusData

方式：监听

备注：每十秒更新一次

返回：数据结构如下

```js
{ totalbetList:
   [ { user: 'lixu', value: '1000.2000 EOS' },
     { user: 'lixutest', value: '400.1000 EOS' } ], // 1.英雄榜用户押注数据
  bonusList: [ { balance: '10.0000 JXB' }, { balance: '110.0000 EOS' } ], //2.分红所有代币的总额
  heroAmountList: [ 11, 5, 2, 1 ], //3.英雄榜分红金额列表
  staketotalAmount: '105.0000' } //4.代币质押总数
```

分红池—预估收益算法：

1.  `(自己质押代币数量/代币质押总数)*分红 EOS 代币的总额`
2.  `(自己质押代币数量/代币质押总数)*分红 JXB 代币的总额`

分红池—每万 JXB 预期收益算法：与预估收益算法一样。

## 二、HTTP

**返回的数据结构**

成功：`{"code":0,"status":"success","data":""}`

失败：`{"code":1,"status":"fail","data":""}`

**端口**

*   dice：3002
*   pool：3003

> Dice 游戏接口

### 首页—最新三十条所有投注

方式：get

接口：/dice/newgamelist

参数：无

eg：

req：`http://127.0.0.1:3002/dice/newgamelist`

res：`{"code":0,"status":"success","data":[游戏对象,游戏对象,游戏对象]}`

### 首页—转账前获取 seed

方式：get

接口：/dice/seed

参数：无

eg：

req：`http://127.0.0.1:3002/dice/seed`

res：

```js
{"code":0,"status":"success","data":"vS9keFfTZzOSslDEHQ"}
```

### 我的—我的余额

方式：get

接口：/account/balance

参数：account，账号名称，eg：lixu、user

eg：

req：`http://127.0.0.1:3003/account/balance?account=user`

res：`{"code":0,"status":"success","data":{"EOS":72.0238,"JXB":1030.8374}}`

### 我的－网络资源数据

方式：get

接口：/account/netinfo

参数：account，账号名称，eg：lixu、user

eg：

req: `http://127.0.0.1:3003/account/netinfo?account=user`

res:

```js
{"code":0,"status":"success","data":{"cpu_used":284.137,"cpu_total":2922.616,"net_used":10.2939453125,"net_total":2666.4208984375}}
```

> pool 分红合约接口

### 首页—代币奖励数量

方式：get

接口：/pool/bonus/rewardamount

参数：无

eg：

req：`http://127.0.0.1:3003/pool/bonus/rewardamount`

res：`{"code":0,"status":"success","data":"2.500000e+01"}`

### 首页—英雄榜历史数据

方式：get

接口：/pool/hero/list

参数：timevalue，格式：年月日时，eg：2019010517

eg：

req：`http://127.0.0.1:3003/pool/hero/list?timevalue=2019010517`

res：与 websocket 的分红数据返回的数据结构类似。

```js
{"code":0,"status":"success","data":{
    "totalbetList":[{"user":"lixu12341234","value":"0.2000 EOS"},{"user":"lixuuser1111","value":"0.1400 EOS"}],
    "heroAmountList":[4,2,1]}
}
```

### 分红池－我的质押数量

方式：get

接口：/pool/bonus/mystake

参数：account，账号名称，eg：lixu、user

eg：

req: `http://127.0.0.1:3003/pool/bonus/mystake?account=user`

res: `{"code":0,"status":"success","data":"100.0000 JXB"}`

### 分红池－分红记录

方式：get

接口：/pool/bonus/records

参数：timevalue：格式：年月日，eg：20190108

eg：

req: `http://127.0.0.1:3003/pool/bonus/records?timevalue=20190108`

res:

```js
{"code":0,"status":"success","data":
 [{"time":"14:00:00","value":{"JXB":10,"EOS":100.0154}},
  {"time":"15:00:00","value":{"JXB":10,"EOS":100.0014}},
  {"time":"16:00:00","value":{"JXB":12.0028,"EOS":45.0016}},
  {"time":"Total","value":{"JXB":32.0028,"EOS":245.01839999999999}}]
}
```

备注：返回该天的所有分红记录与该天的分红总和，倒序排列，若无记录则为空。

### 分红池－我的余额

方式：get

接口：/pool/bonus/balance

参数：account，账号名称，eg：lixu、user

eg：

req: `http://127.0.0.1:3003/pool/bonus/balance?account=user`

res:

```js
{"code":0,"status":"success","data":{"JXB":66.6666,"EOS":450.9709999999}}
```

备注：余额数量保留十位小数。

### VIP－等级

方式：get

接口：/pool/vip/totalamount

参数：无

eg：

req: `http://127.0.0.1:3003/pool/vip/grade`

res:

```js
{"code":0,"status":"success","data":[[1,1000,"0.01%"],[2,5000,"0.02%"],[3,10000,"0.03%"],[4,50000,"0.04%"],[5,100000,"0.05%"],[6,500000,"0.07%"],[7,1000000,"0.09%"],[8,5000000,"0.11%"],[9,10000000,"0.13%"],[10,50000000,"0.15%"]]}
```

备注：“VIP－等级”数据会保持不变，不用刷新获取新数据。

### VIP－我的累计押注额

方式：get

接口：/pool/vip/totalamount

参数：account，账号名称，eg：lixu、user

eg：

req: `http://127.0.0.1:3003/pool/vip/totalamount?account=user`

res:

```js
{"code":0,"status":"success","data":[1010.2,"EOS"]}
```

备注：数据会根据用户押注持续增加。代币余额数量保留四位小数。

## 三、合约

**本地网络的合约账号**

```js
eosContractAccount: "eosio.token",
myTokenContractAccount: "clubtoken111",
poolContractAccount: "clubpool1111",
gameContractAccount: "clubadmin111",
```

### 押注 EOS－转账

合约账号：`eosContractAccount`

合约接口定义：`void transfer(account_name from,　account_name to,　asset quantity,　string memo)`

### 押注代币－转账

合约账号：`myTokenContractAccount`

合约接口定义：`void transfer(account_name from,　account_name to,　asset quantity,　string memo)`

### 分红池－质押代币

合约账号：`myTokenContractAccount`

合约接口定义：`void stake(account_name owner, asset value)`

### 分红池－赎回代币

合约账号：`myTokenContractAccount`

合约接口定义：`void unstake(account_name owner, asset value)`

### 分红池－提现

合约账号：`poolContractAccount`

合约接口定义：`void drawdividend(const account_name user)`

## 四、TokenPocket 钱包 API

备注：用于移动端。

SDK：[`github.com/TP-Lab/tp-eosjs`](https://github.com/TP-Lab/tp-eosjs)

### 连接 TP

```js
document.write("<script language=javascript src='/dist/tp.js'></script>");

alert("连接：" + tp.isConnected())
```

### 钱包的当前账号

备注：登录时，获取当前账号显示即可。退出时，不用调用任何方法。

```js
tp.getCurrentWallet().then(function (data) {
        alert(JSON.stringify(data))
    })
```

### 发送交易

以转账 EOS 为例

```js
account = ""
to = ""
tp.pushEosAction({
    actions: [
        {
            account: 'eosio.token',
            name: 'transfer',
            authorization: [{
                actor: account,
                permission: 'active'
            }],
            data: {
                from: account,
                to: to,
                quantity: '1.3000 EOS',
                memo: 'something to say'
            }
        }
    ],
    address: 'EOS5dcyMZJzcNyXYsZrEuTVPMBDbVskuPotxbcZDvR6f8v1X3iuaf',
    account: account
})
```

## 五、Scatter API

备注：用于 PC 端，兼容大部分知名钱包 App。

SDK：[`get-scatter.com/docs/getting-started`](https://get-scatter.com/docs/getting-started)

**版权声明：博客中的文章版权归博主所有，转载请联系作者（微信：lixu1770105）。**