# 第六章 测试基于 fabric-sdk-node 的应用

## 从零到壹构建基于 fabric-sdk-node 的项目开发实战之五

### 运行示例程序

一切就绪，现在可以开始测试我们的应用程序了。

#### 终端窗口 1

*   进入项目根目录：

    ```js
    $ cd $HOME/kevin-fabric-sdk-node
    ```

*   使用 docker-compose 命令启动网络

    ```js
    $ docker-compose -f artifacts/docker-compose.yaml up
    ```

#### 终端窗口 2

*   进入项目根目录，安装 `gulp`：

    ```js
    $ cd $HOME/kevin-fabric-sdk-node
    $ npm install -g gulp
    ```

*   安装 `fabric-client` 与 `fabric-ca-client` 模块

    ```js
    $ npm install
    ```

    执行成功如下信息：

    ![npm-install-ok](img/ee8a01da45db80c5cac513835096d0bb.jpg)

*   启动 node app 指定 PORT 为 4000

    ```js
    $ PORT=4000 node app
    ```

    启动成功如下图所示：

    ![node app](img/486406f0b5ce3fb8c3dd1b0a8ec296db.jpg)

#### 终端窗口 3

*   执行 REST APIs 参见 REST APIs 调用请求

### REST APIs 调用请求

#### 登录请求

在终端窗口 3 中执行：

```js
$ cd $HOME/kevin-fabric-sdk-node
```

*   在 **Org1** 组织中注册并登记一个新的用户:

```js
$ curl -s -X POST \
    http://localhost:4000/users \
    -H "content-type: application/x-www-form-urlencoded" \
    -d 'username=Jim&orgName=Org1'
```

> 如果出现 502 错误，那么使用 `unset http_proxy` 取消/关闭代理即可。

**终端输出如下响应信息:**

```js
{
  "success": true,
  "secret": "RaxhMgevgJcm",
  "message": "Jim enrolled Successfully",
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE1NDExNTc2OTUsInVzZXJuYW1lIjoiSmltIiwib3JnTmFtZSI6Ik9yZzEiLCJpYXQiOjE1NDExMjE2OTV9.TqN-T3xQOCkPf_HYAxd21RbmZ7FAOjcPyXadzVrfWYs"
}
```

**注意：**响应信息包含成功/失败的状态，**注册密钥**和**JSON Web 令牌（JWT）**，它是后续请求的请求标头中的所必需字符串

#### 创建通道请求

```js
$ curl -s -X POST \
  http://localhost:4000/channels \
  -H "authorization: Bearer <put JSON Web Token here>" \
  -H "content-type: application/json" \
  -d '{
    "channelName":"kevinkongyixueyuan",
    "channelConfigPath":"../artifacts/channel/mychannel.tx"
}'
```

**请注意**，Header **授权**必须包含`POST /users`通话中返回的 **JWT**

执行成功终端输出如下响应信息：

```js
{
    "success":true,
    "message":"Channel 'kevinkongyixueyuan' created Successfully"
}
```

#### 加入通道请求

```js
$ curl -s -X POST \
  http://localhost:4000/channels/kevinkongyixueyuan/peers \
  -H "authorization: Bearer <put JSON Web Token here>" \
  -H "content-type: application/json" \
  -d '{
    "peers": ["peer0.org1.kevin.kongyixueyuan.com","peer1.org1.kevin.kongyixueyuan.com"]
}'
```

执行成功终端输出如下响应信息：

```js
{
    "success":true,
    "message":"Successfully joined peers in organization Org1 to the channel:kevinkongyixueyuan"
}
```

#### 安装链码

```js
$ curl -s -X POST \
  http://localhost:4000/chaincodes \
  -H "authorization: Bearer <put JSON Web Token here>" \
  -H "content-type: application/json" \
  -d '{
    "peers": ["peer0.org1.kevin.kongyixueyuan.com","peer1.org1.kevin.kongyixueyuan.com"],
    "chaincodeName":"mycc",
    "chaincodePath":"github.com/example_cc",
    "chaincodeType": "golang",
    "chaincodeVersion":"v0"
}'
```

> **注意：** 当使用 node.js 链*代码时，*必须将*chaincodeType*设置为**node**，并且必须将*chaincodePath*设置为 node.js chaincode 的位置。也加入了$PWD
> 
> ```js
> 如下:
> $ curl -s -X POST \
> http://localhost:4000/chaincodes \
> -H "authorization: Bearer <put JSON Web Token here>" \
> -H "content-type: application/json" \
> -d '{
> "peers": ["peer0.org1.kevin.kongyixueyuan.com","peer1.org1.kevin.kongyixueyuan.com"],
> "chaincodeName":"mycc",
> "chaincodePath":"$PWD/artifacts/src/github.com/example_cc/node",
> "chaincodeType": "node",
> "chaincodeVersion":"v0"
> }'
> ```

执行成功终端输出如下响应信息：

```js
{
    "success":true,
    "message":"Successfully install chaincode"
}
```

#### 实例化链码

```js
$ curl -s -X POST \
  http://localhost:4000/channels/kevinkongyixueyuan/chaincodes \
  -H "authorization: Bearer <put JSON Web Token here>" \
  -H "content-type: application/json" \
  -d '{
    "peers": ["peer0.org1.kevin.kongyixueyuan.com","peer1.org1.kevin.kongyixueyuan.com"],
    "chaincodeName":"mycc",
    "chaincodeVersion":"v0",
    "chaincodeType": "golang",
    "args":["a","100","b","200"]
}'
```

执行成功终端输出如下：

```js
{
    "success":true,
    "message":"Successfully instantiate chaingcode in organization Org1 to the channel 'kevinkongyixueyuan'"
}
```

#### 调用链码请求

```js
$ curl -s -X POST \
  http://localhost:4000/channels/kevinkongyixueyuan/chaincodes/mycc \
  -H "authorization: Bearer <put JSON Web Token here>" \
  -H "content-type: application/json" \
  -d '{
    "peers": ["peer0.org1.kevin.kongyixueyuan.com","peer1.org1.kevin.kongyixueyuan.com"],
    "fcn":"move",
    "args":["a","b","10"]
}'
```

**注意：** 当使用 node.js 链代码时，必须将*chaincodeType*设置为**node**

执行成功终端中输出响应的**事务 ID**

```js
fad37683c5e716a9102cd15a1bd6555440374cd39d9464e3ee805802c7be3d58
```

**注意：**请保存响应信息的事务 ID，在后续查询事务中会使用到此字符串。

#### 查询链码

```js
$ curl -s -X GET \
  "http://localhost:4000/channels/kevinkongyixueyuan/chaincodes/mycc?peer=peer0.org1.kevin.kongyixueyuan.com&fcn=query&args=%5B%22a%22%5D" \
  -H "authorization: Bearer <put JSON Web Token here>" \
  -H "content-type: application/json" 
```

执行成功终端中输出如下响应信息：

```js
a now has 90 after the move
```

#### 根据块号查询块

```js
$ curl -s -X GET \
  "http://localhost:4000/channels/kevinkongyixueyuan/blocks/1?peer=peer0.org1.kevin.kongyixueyuan.com" \
  -H "authorization: Bearer <put JSON Web Token here>" \
  -H "content-type: application/json" 
```

#### 根据 TransactionID 查询事务

```js
$ curl -s -X GET http://localhost:4000/channels/kevinkongyixueyuan/transactions/$TRX_ID?peer=peer0.org1.kevin.kongyixueyuan.com \
  -H "authorization: Bearer <put JSON Web Token here>" \
  -H "content-type: application/json"
```

**注意**：事务 ID 来自于之前的事务调用

#### 查询 ChainInfo

```js
$ curl -s -X GET \
  "http://localhost:4000/channels/kevinkongyixueyuan?peer=peer0.org1.kevin.kongyixueyuan.com" \
  -H "authorization: Bearer <put JSON Web Token here>" \
  -H "content-type: application/json" 
```

执行成功终端中输出如下响应信息：

```js
{"height":{"low":3,"high":0,"unsigned":true},"currentBlockHash":{"buffer":{"type":"Buffer","data":[8,3,18,32,90,58,37,82,74,234,234,178,22,93,187,246,79,177,37,66,84,226,156,100,61,35,120,220,166,146,193,20,143,34,221,57,26,32,137,149,94,253,17,79,72,43,212,158,153,70,25,218,146,32,22,125,198,104,249,102,21,157,183,90,12,28,111,246,17,66]},"offset":4,"markedOffset":-1,"limit":36,"littleEndian":true,"noAssert":false},"previousBlockHash":{"buffer":{"type":"Buffer","data":[8,3,18,32,90,58,37,82,74,234,234,178,22,93,187,246,79,177,37,66,84,226,156,100,61,35,120,220,166,146,193,20,143,34,221,57,26,32,137,149,94,253,17,79,72,43,212,158,153,70,25,218,146,32,22,125,198,104,249,102,21,157,183,90,12,28,111,246,17,66]},"offset":38,"markedOffset":-1,"limit":70,"littleEndian":true,"noAssert":false}}
```

#### 查询已安装的链代码

```js
$ curl -s -X GET \
  "http://localhost:4000/chaincodes?peer=peer0.org1.kevin.kongyixueyuan.com" \
  -H "authorization: Bearer <put JSON Web Token here>" \
  -H "content-type: application/json" 
```

执行成功终端中输出如下响应信息：

```js
["name: mycc, version: v0, path: github.com/example_cc"]
```

#### 查询实例化链代码

```js
$ curl -s -X GET \
  "http://localhost:4000/channels/kevinkongyixueyuan/chaincodes?peer=peer0.org1.kevin.kongyixueyuan.com" \
  -H "authorization: Bearer <put JSON Web Token here>" \
  -H "content-type: application/json" 
```

执行成功终端中输出如下响应信息：

```js
["name: mycc, version: v0, path: github.com/example_cc"]
```

#### 查询通道

```js
$ curl -s -X GET \
  "http://localhost:4000/channels?peer=peer0.org1.kevin.kongyixueyuan.com" \
  -H "authorization: Bearer <put JSON Web Token here>" \
  -H "content-type: application/json" 
```

执行成功终端中输出如下响应信息：

```js
{"channels":[{"channel_id":"kevinkongyixueyuan"}]}
```

#### 发现 IP 地址

要检索某个网络实体的 IP 地址，请使用如下命令：

```js
# this will return the IP Address for peer0
$ docker inspect peer0 | grep IPAddress
```

### 关闭并清理网络

创建清理网络的脚本文件：

```js
$ cd $HOME/kevin-fabric-sdk-node
$ vim shutdownnetwork.sh
```

文件中添加如下内容：

```js
echo "shutdown fabric-network"
echo
docker-compose -f artifacts/docker-compose.yaml down
echo

echo "docker rm -f $(docker ps -aq)"
echo
docker rm -f $(docker ps -aq)
echo

echo "docker rmi -f $(docker images | grep dev | awk '{print $3}')"
echo
docker rmi -f $(docker images | grep dev | awk '{print $3}')
echo

echo "Delete current dir org-kv"
echo
rm -rf fabric-client-kv-org[1-2]
echo

echo "Delete tmp dir org-kv"
echo
rm -rf /tmp/fabric-client-kv-org[1-2]
echo

echo "shutdown and clear OK"
```

添加可执行权限并执行：

```js
$ chmod 777 ./shutdownnetwork.sh
$ ./shutdownnetwork.sh
```

应用完整源代码下载地址，[请点击此处](https://github.com/kevin-hf/kevin-fabric-sdk-node)