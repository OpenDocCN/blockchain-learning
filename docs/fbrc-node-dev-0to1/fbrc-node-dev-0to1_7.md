# 第七章 实现自动化测试

## 从零到壹构建基于 fabric-sdk-node 的项目开发实战之六

### 自动化测试

之前的方式，需要我们每次都要输入命令，这样操作的话比较麻烦，所以现在我们对其进行简化，没有必要每次都输入一长串的命令来执行，而是将命令写在指定的脚本文件中，以后直接执行该脚本文件即可，此方式大大简化了测试步骤，提高效率。

#### 终端窗口 1

进入项目根目录：

```js
$ cd $HOME/kevin-fabric-sdk-node 
```

##### 创建 `runApp.sh` 文件并编辑

```js
$ vim runApp.sh 
```

`runApp.sh` 脚本文件具体作用如下：

*   在本地计算机上启动通过配置指定的网络环境
*   安装 `fabric-client` 和 `fabric-ca-client` 节点模块
*   并在 PORT 4000 上启动应用程序

`runApp.sh` 文件完整内容如下：

```js
#!/bin/bash
#
# Copyright IBM Corp. All Rights Reserved.
#
# SPDX-License-Identifier: Apache-2.0
#

function dkcl(){
        CONTAINER_IDS=$(docker ps -aq)
    echo
        if [ -z "$CONTAINER_IDS" -o "$CONTAINER_IDS" = " " ]; then
                echo "========== No containers available for deletion =========="
        else
                docker rm -f $CONTAINER_IDS
        fi
    echo
}

function dkrm(){
        DOCKER_IMAGE_IDS=$(docker images | grep "dev\|none\|test-vp\|peer[0-9]-" | awk '{print $3}')
    echo
        if [ -z "$DOCKER_IMAGE_IDS" -o "$DOCKER_IMAGE_IDS" = " " ]; then
        echo "========== No images available for deletion ==========="
        else
                docker rmi -f $DOCKER_IMAGE_IDS
        fi
    echo
}

function restartNetwork() {
    echo

        #teardown the network and clean the containers and intermediate images
    cd artifacts
    docker-compose down
    dkcl
    dkrm

    #Cleanup the material
    rm -rf /tmp/hfc-test-kvs_peerOrg* $HOME/.hfc-key-store/ /tmp/fabric-client-kvs_peerOrg*

    #Start the network
    docker-compose up -d
    cd -
    echo
}

function installNodeModules() {
    echo
    if [ -d node_modules ]; then
        echo "============== node modules installed already ============="
    else
        echo "============== Installing node modules ============="
        npm install
    fi
    echo
}

restartNetwork

installNodeModules

PORT=4000 node app 
```

##### 添加可执行权限并执行 `runApp.sh` 脚本：

```js
$ chmod 777 ./runApp.sh
$ ./runApp.sh 
```

#### 终端窗口 2

为了使 shell 脚本能够正确解析 JSON，您必须安装 `jq`，详情请 [参见此处](https://stedolan.github.io/jq/)

**安装 jq**

```js
$ sudo apt update
$ sudo apt install -y jq 
```

进入项目根目录中：

```js
$ cd $HOME/kevin-fabric-sdk-node 
```

##### 创建并编辑 `testAPIs.sh` 脚本文件

```js
$ vim testAPIs.sh 
```

`testAPIs.sh` 脚本文件的主要作用如下：

*   创建用户
*   创建通道
*   安装链码
*   实例化链码
*   调用链码
*   调用链码查询
*   执行其它查询

`testAPIs.sh` 文件完整内容如下：

```js
#!/bin/bash
#
# Copyright IBM Corp. All Rights Reserved.
#
# SPDX-License-Identifier: Apache-2.0
#

jq --version > /dev/null 2>&1
if [ $? -ne 0 ]; then
    echo "Please Install 'jq' https://stedolan.github.io/jq/ to execute this script"
    echo
    exit 1
fi

starttime=$(date +%s)

# Print the usage message
function printHelp () {
  echo "Usage: "
  echo "  ./testAPIs.sh -l golang|node"
  echo "    -l <language> - chaincode language (defaults to \"golang\")"
}
# Language defaults to "golang"
LANGUAGE="golang"

# Parse commandline args
while getopts "h?l:" opt; do
  case "$opt" in
    h|\?)
      printHelp
      exit 0
    ;;
    l)  LANGUAGE=$OPTARG
    ;;
  esac
done

##set chaincode path
function setChaincodePath(){
    LANGUAGE=`echo "$LANGUAGE" | tr '[:upper:]' '[:lower:]'`
    case "$LANGUAGE" in
        "golang")
        CC_SRC_PATH="github.com/example_cc"
        ;;
        "node")
        CC_SRC_PATH="$PWD/artifacts/src/github.com/example_cc/node"
        ;;
        *) printf "\n ------ Language $LANGUAGE is not supported yet ------\n"$
        exit 1
    esac
}

setChaincodePath

echo "POST request Enroll on Org1  ..."
echo
ORG1_TOKEN=$(curl -s -X POST \
  http://localhost:4000/users \
  -H "content-type: application/x-www-form-urlencoded" \
  -d 'username=Jim&orgName=Org1')
echo $ORG1_TOKEN
ORG1_TOKEN=$(echo $ORG1_TOKEN | jq ".token" | sed "s/\"//g")
echo
echo "ORG1 token is $ORG1_TOKEN"
echo
echo "POST request Enroll on Org2 ..."
echo
ORG2_TOKEN=$(curl -s -X POST \
  http://localhost:4000/users \
  -H "content-type: application/x-www-form-urlencoded" \
  -d 'username=Barry&orgName=Org2')
echo $ORG2_TOKEN
ORG2_TOKEN=$(echo $ORG2_TOKEN | jq ".token" | sed "s/\"//g")
echo
echo "ORG2 token is $ORG2_TOKEN"
echo
echo
echo "POST request Create channel  ..."
echo
curl -s -X POST \
  http://localhost:4000/channels \
  -H "authorization: Bearer $ORG1_TOKEN" \
  -H "content-type: application/json" \
  -d '{
    "channelName":"kevinkongyixueyuan",
    "channelConfigPath":"../artifacts/channel/mychannel.tx"
}'
echo
echo
sleep 5
echo "POST request Join channel on Org1"
echo
curl -s -X POST \
  http://localhost:4000/channels/kevinkongyixueyuan/peers \
  -H "authorization: Bearer $ORG1_TOKEN" \
  -H "content-type: application/json" \
  -d '{
    "peers": ["peer0.org1.kevin.kongyixueyuan.com","peer1.org1.kevin.kongyixueyuan.com"]
}'
echo
echo

echo "POST request Join channel on Org2"
echo
curl -s -X POST \
  http://localhost:4000/channels/kevinkongyixueyuan/peers \
  -H "authorization: Bearer $ORG2_TOKEN" \
  -H "content-type: application/json" \
  -d '{
    "peers": ["peer0.org2.kevin.kongyixueyuan.com","peer1.org2.kevin.kongyixueyuan.com"]
}'
echo
echo

echo "POST Install chaincode on Org1"
echo
curl -s -X POST \
  http://localhost:4000/chaincodes \
  -H "authorization: Bearer $ORG1_TOKEN" \
  -H "content-type: application/json" \
  -d "{
    \"peers\": [\"peer0.org1.kevin.kongyixueyuan.com\",\"peer1.org1.kevin.kongyixueyuan.com\"],
    \"chaincodeName\":\"mycc\",
    \"chaincodePath\":\"$CC_SRC_PATH\",
    \"chaincodeType\": \"$LANGUAGE\",
    \"chaincodeVersion\":\"v0\"
}"
echo
echo

echo "POST Install chaincode on Org2"
echo
curl -s -X POST \
  http://localhost:4000/chaincodes \
  -H "authorization: Bearer $ORG2_TOKEN" \
  -H "content-type: application/json" \
  -d "{
    \"peers\": [\"peer0.org2.kevin.kongyixueyuan.com\",\"peer1.org2.kevin.kongyixueyuan.com\"],
    \"chaincodeName\":\"mycc\",
    \"chaincodePath\":\"$CC_SRC_PATH\",
    \"chaincodeType\": \"$LANGUAGE\",
    \"chaincodeVersion\":\"v0\"
}"
echo
echo

echo "POST instantiate chaincode on peer1 of Org1"
echo
curl -s -X POST \
  http://localhost:4000/channels/kevinkongyixueyuan/chaincodes \
  -H "authorization: Bearer $ORG1_TOKEN" \
  -H "content-type: application/json" \
  -d "{
    \"chaincodeName\":\"mycc\",
    \"chaincodeVersion\":\"v0\",
    \"chaincodeType\": \"$LANGUAGE\",
    \"args\":[\"a\",\"100\",\"b\",\"200\"]
}"
echo
echo

echo "POST invoke chaincode on peers of Org1"
echo
TRX_ID=$(curl -s -X POST \
  http://localhost:4000/channels/kevinkongyixueyuan/chaincodes/mycc \
  -H "authorization: Bearer $ORG1_TOKEN" \
  -H "content-type: application/json" \
  -d '{
    "peers": ["peer0.org1.kevin.kongyixueyuan.com","peer1.org1.kevin.kongyixueyuan.com"],
    "fcn":"move",
    "args":["a","b","10"]
}')
echo "Transaction ID is $TRX_ID"
echo
echo

echo "GET query chaincode on peer1 of Org1"
echo
curl -s -X GET \
  "http://localhost:4000/channels/kevinkongyixueyuan/chaincodes/mycc?peer=peer0.org1.kevin.kongyixueyuan.com&fcn=query&args=%5B%22a%22%5D" \
  -H "authorization: Bearer $ORG1_TOKEN" \
  -H "content-type: application/json"
echo
echo

echo "GET query Block by blockNumber"
echo
curl -s -X GET \
  "http://localhost:4000/channels/kevinkongyixueyuan/blocks/1?peer=peer0.org1.kevin.kongyixueyuan.com" \
  -H "authorization: Bearer $ORG1_TOKEN" \
  -H "content-type: application/json"
echo
echo

echo "GET query Transaction by TransactionID"
echo
curl -s -X GET http://localhost:4000/channels/kevinkongyixueyuan/transactions/$TRX_ID?peer=peer0.org1.kevin.kongyixueyuan.com \
  -H "authorization: Bearer $ORG1_TOKEN" \
  -H "content-type: application/json"
echo
echo

############################################################################
### TODO: What to pass to fetch the Block information
############################################################################
#echo "GET query Block by Hash"
#echo
#hash=????
#curl -s -X GET \
#  "http://localhost:4000/channels/mychannel/blocks?hash=$hash&peer=peer1" \
#  -H "authorization: Bearer $ORG1_TOKEN" \
#  -H "cache-control: no-cache" \
#  -H "content-type: application/json" \
#  -H "x-access-token: $ORG1_TOKEN"
#echo
#echo

echo "GET query ChainInfo"
echo
curl -s -X GET \
  "http://localhost:4000/channels/kevinkongyixueyuan?peer=peer0.org1.kevin.kongyixueyuan.com" \
  -H "authorization: Bearer $ORG1_TOKEN" \
  -H "content-type: application/json"
echo
echo

echo "GET query Installed chaincodes"
echo
curl -s -X GET \
  "http://localhost:4000/chaincodes?peer=peer0.org1.kevin.kongyixueyuan.com" \
  -H "authorization: Bearer $ORG1_TOKEN" \
  -H "content-type: application/json"
echo
echo

echo "GET query Instantiated chaincodes"
echo
curl -s -X GET \
  "http://localhost:4000/channels/kevinkongyixueyuan/chaincodes?peer=peer0.org1.kevin.kongyixueyuan.com" \
  -H "authorization: Bearer $ORG1_TOKEN" \
  -H "content-type: application/json"
echo
echo

echo "GET query Channels"
echo
curl -s -X GET \
  "http://localhost:4000/channels?peer=peer0.org1.kevin.kongyixueyuan.com" \
  -H "authorization: Bearer $ORG1_TOKEN" \
  -H "content-type: application/json"
echo
echo

echo "Total execution time : $(($(date +%s)-starttime)) secs ..." 
```

应用程序在终端 1 中启动之后，接下来，我们通过执行 **testAPIs.sh** 脚本来测试 API ：

##### 添加可执行权限并执行 `testAPIs.sh` 脚本：

```js
$ chmod 777 ./testAPIs.sh
$ ./testAPIs.sh 
```

### 关闭并清理网络

创建清理网络的脚本文件：

```js
$ cd $HOME/kevin-fabric-sdk-node
$ vim stopAPP.sh 
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
$ chmod 777 ./stopAPP.sh
$ ./stopAPP.sh 
```

应用完整源代码下载地址，[请点击此处](https://github.com/kevin-hf/kevin-fabric-sdk-node)

​

### 参考资料

*   [Hyperledger Fabric-SDK-Node](https://github.com/hyperledger/fabric-sdk-node)
*   [Node SDK documentation](https://fabric-sdk-node.github.io/)
*   [fabric-samples balance-transfer](https://github.com/hyperledger/fabric-samples/tree/release/balance-transfer)