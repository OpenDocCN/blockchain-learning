# 三、.3 测试

## 从零到壹实现 Marbles 资产管理系统 （Fabric-SDK-Node）之－测试

### 创建网络启动脚本

创建 start.sh 脚本并编辑， 该脚本文件主要作用如下：

*   **设置环境变量**
*   **根据指定的配置文件启动区块链网络**
*   **创建通道**
*   **将指定的 peer 节点加入到通道中**

```js
$ cd ~/kevin-marbles/scripts
$ vim start.sh 
```

start.sh 脚本完整内容如下：

```js
#!/bin/bash
set -ev
export MSYS_NO_PATHCONV=1
export COMPOSE_PROJECT_NAME=net

docker-compose -f ../artifacts/docker-compose.yaml down

rm -rf ../hfc-key-store

docker-compose -f ../artifacts/docker-compose.yaml up -d ca.org1.kevin.chaindesk.cn orderer.kevin.chaindesk.cn peer0.org1.kevin.chaindesk.cn

# wait for Hyperledger Fabric to start
# incase of errors when running later commands, issue export FABRIC_START_TIMEOUT=<larger number>
export FABRIC_START_TIMEOUT=10
#echo ${FABRIC_START_TIMEOUT}
sleep ${FABRIC_START_TIMEOUT}

# Create the channel
docker exec -e "CORE_PEER_LOCALMSPID=org1.kevin.chaindesk.cn" -e "CORE_PEER_MSPCONFIGPATH=/etc/hyperledger/msp/users/Admin@org1.kevin.chaindesk.cn/msp" peer0.org1.kevin.chaindesk.cn peer channel create -o orderer.kevin.chaindesk.cn:7050 -c kevinchaindesk -f /etc/hyperledger/configtx/channel.tx
# Join peer0.org1.kevin.chaindesk.cn to the channel.
docker exec -e "CORE_PEER_LOCALMSPID=org1.kevin.chaindesk.cn" -e "CORE_PEER_MSPCONFIGPATH=/etc/hyperledger/msp/users/Admin@org1.kevin.chaindesk.cn/msp" peer0.org1.kevin.chaindesk.cn peer channel join -b kevinchaindesk.block 
```

为 start.sh 脚本添加可执行权限

```js
$ chmod 777 ./start.sh 
```

start.sh 脚本具有了可执行权限之后，我们就可以执行 start.sh 脚本，用来启动我们的区块链网络。

### 启动网络

```js
$ cd ~/kevin-marbles/scripts
$ ./start.sh 
```

输出如下类似信息：

![env_start](img/14d892b951673d64786b5011b42869f9.jpg)

完成后，我们需要将链代码放到 peer 的文件系统上。记住，chaincode 定义了弹珠（资产）是什么，并且实现了我们的弹珠交易的业务逻辑。可在此目录中找到弹珠链码`<marbles root>/chaincode/src/`。目录中包含了定义的弹珠结构体及链码相应的所有业务处理逻辑。

我们将使用的脚本`install_chaincode.js`位于`<marbles root>/scripts`文件夹中。它将读入我们的 Marble 配置文件和连接配置文件数据。您可以通过编辑`install_chaincode.js`文件来更改 Marble 链码 ID 或版本。

### 安装链码

使用以下命令安装 Marble 链码文件

```js
$ cd ~/kevin-marbles/scripts
$ node install_chaincode.js 
```

命令执行成功输出如下类似信息：

![install_chaincode](img/7ec19e8281a1a0d735aeb94115ca7baf.jpg)

### 实例化链码

使用如下命令实例化链码

```js
$ node instantiate_chaincode.js 
```

命令执行成功输出如下类似信息：

![instantiate_chaincode](img/fe211e8284285e3d8153c006a36063b9.jpg)

### 关闭并清理网络

```js
$ cd ~/kevin-marbles/scripts
$ vim stop.sh 
```

stop.sh 脚本完整内容如下

```js
#!/bin/bash

set -ev

# Shut down the Docker containers that might be currently running.
# docker-compose -f ../artifacts/docker-compose.yaml stop

set -e

# Shut down the Docker containers for the system tests.
docker-compose -f ../artifacts/docker-compose.yaml kill && docker-compose -f ../artifacts/docker-compose.yaml down

# remove the local state
rm -rf ../hfc-key-store

# remove chaincode docker images
docker rm -f $(docker ps -aq)
docker rmi -f $(docker images | grep dev | awk '{print $3}')

# Your system is now clean 
```

为 stop.sh 脚本添加可执行权限

```js
$ chmod 777 ./stop.sh 
```

执行

```js
$ ./stop.sh 
```

输出如下类似信息：

![env_clean](img/bcd24d3c90a26b8cbe601766325cb5e7.jpg)