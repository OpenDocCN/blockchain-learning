# 第一章 搭建 Hyperledger Fabric 网络环境

[TOC]

## 从零到壹构建基于 fabric-sdk-node 的项目开发实战之一

> 我们通过对 fabric-samples 中的一个 NodeJS 示例应用程序（balance-transfer）的分析，来演示 fabric-client 和 fabric-ca-client 及 fabric-sdk-node API 的使用

### 先决条件和安装设置:

*   **Ubuntu 16.04**
*   **Vim、[Git Client](https://git-scm.com/downloads)**
*   [Docker](https://www.docker.com/products/overview) - v17.03+
*   [Docker Compose](https://docs.docker.com/compose/overview/) - v1.8+
*   **Golang** v1.10+
*   **Node.js** v8.4.0+

#### 安装 vim、git

```js
$ sudo apt install vim
$ sudo apt install git
$ sudo apt install curl
```

#### 安装 docker

**需要 Docker 版本 17.03.0-ce 或更高版本。**

```js
$ docker version 
$ sudo apt install docker.io
```

安装完成后执行版本查询命令

```js
$ sudo docker version
```

![docker 版本](img/3d1880a8cac46baaf0098d6d4e98fbbb.jpg)

#### 安装 docker-compose

**docker-compose 1.8 或更高版本是必需的。**

我们目前无法一次性轻松管理多个容器。 为了解决这个问题，需要**docker-compose** 。

```js
$ docker-compose version 
$ sudo apt install docker-compose
```

安装完成后查询：

```js
$ docker-compose version 
```

![docker-compose 版本](img/11efada7ac940aa1a5b263082a03727b.jpg)

将当前用户添加到 docker 组

```js
$ sudo usermod -aG docker kevin
```

添加成功后**必须注销/退出并重新登录**(如果使用的是远程连接工具，退出终端重新连接即可)

#### 安装 Golang

**需要版本 1.10.x 或更高。**如果您使用的是 Hyperledger Fabric 1.1.x 版本，那么 Golang 版本在 1.9.x 以上

```js
 $ go version 
 $ wget https://dl.google.com/go/go1.10.3.linux-amd64.tar.gz
```

> 下载受网络环境影响，如果您本地有相应的 tar.gz 包，则使用下面的命令直接解压到指定的路径下即可。

使用 tar 命令将下载后的压缩包文件解压到指定的 /usr/local/ 路径下

```js
$ sudo tar -zxvf go1.10.3.linux-amd64.tar.gz -C /usr/local/
```

设置 GOPATH & GOROOT 环境变量, 通过 `go env` 查看 GOPATH 路径

```js
$ sudo vim /etc/profile
```

> 如果只想让当前登录用户使用 Golang， 其它用户不能使用， 则编辑当前用户$HOME 目录下的 .bashrc 或 .profile 文件， 在该文件中添加相应的环境变量即可。

在 profile 文件最后添加如下内容:

```js
export GOPATH=$HOME/go
export GOROOT=/usr/local/go
export PATH=$GOROOT/bin:$PATH
```

使用 source 命令，使刚刚添加的配置信息生效：

```js
$ source /etc/profile
```

通过 go version 命令验证是否成功：

```js
$ go version
```

![Go 版本](img/0019de4439e04d68891d041ab86d63a3.jpg)

#### 安装 Node

**安装 nvm**

```js
$ sudo apt update
$ curl -o- https://raw.githubusercontent.com/creationix/nvm/v0.33.10/install.sh | bash

$ export NVM_DIR="$HOME/.nvm"
$ [ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh" 
```

**安装 Node**

```js
$ nvm install v8.11.1
```

安装完成终端输出如下信息：

![node-npm-version](img/07fc391e87439497c180b0472e73698a.jpg)

也可以使用命令查看相关的版本信息：

*   检查 Node 版本

    ```js
    $ node -v
    ```

    输出: `v8.11.1`

*   检查 npm 版本

    ```js
    $ npm -v
    ```

    输出: `5.6.0`

### 配置网络环境

使用 `git` 命令克隆 `kevin-fabric-sdk-node` 目录到当前登录用户的 $HOME 路径

```js
$ cd ~
$ git clone https://github.com/kevin-hf/kongyixueyuanOrg.git
```

修改目录名称 kongyixueyuanOrg 为 kevin-fabric-sdk-node 之后进入该目录，修改 `artifacts` 文件夹的所属关系为当前用户

```js
$ mv kongyixueyuanOrg kevin-fabric-sdk-node
$ cd kevin-fabric-sdk-node
$ sudo chown -R kevin:kevin ./artifacts
```

> 提示： kevin 为安装 Ubuntu 16.04 系统时创建的用户

进入 `artifacts` 目录

```js
$ cd artifacts
```

为了构建区块链网络，使用 `docker` 构建处理不同角色的虚拟计算机。 在这里我们将尽可能保持简单。如果确定您的系统中已经存在相关的所需容器，或可以使用其它方式获取，则无需执行如下命令。否则请将 `fixtures` 目录下的 `pull_images.sh` 文件添加可执行权限后直接执行。

#### 下载 Docker images

```js
$ chmod 777 ./pull_images.sh
$ ./pull_images.sh 
```

> 提示：`pull_images.sh` 文件是下载 Fabric 环境所需容器的一个可执行脚本，下载过程需要一段时间（视网速情况而定），请耐心等待。
> 
> 另：请确定您的系统支持虚拟技术。

下载完成后终端输出如下：

![docker-images-Download](img/a7d0b3a1f4b2a76863b803f921ef7f75.jpg)

#### 配置 docker-compose.yaml

首先，我们需要进入项目的 `artifacts` 目录下

```js
$ cd $HOME/kevin-fabric-sdk-node/artifacts
```

##### 创建 base.yaml 文件并编辑

```js
$ vim base.yaml
```

`base.yaml` 文件完整内容如下：

```js
# Copyright IBM Corp. All Rights Reserved.
#
# SPDX-License-Identifier: Apache-2.0
#

version: '2'
services:
  peer-base:
    image: hyperledger/fabric-peer
    environment:
      - CORE_VM_ENDPOINT=unix:///host/var/run/docker.sock
      # the following setting starts chaincode containers on the same
      # bridge network as the peers
      # https://docs.docker.com/compose/networking/
      - CORE_VM_DOCKER_HOSTCONFIG_NETWORKMODE=artifacts_default
      - CORE_LOGGING_LEVEL=DEBUG
      - CORE_PEER_GOSSIP_USELEADERELECTION=true
      - CORE_PEER_GOSSIP_ORGLEADER=false
      # The following setting skips the gossip handshake since we are
      # are not doing mutual TLS
      - CORE_PEER_GOSSIP_SKIPHANDSHAKE=true
      - CORE_PEER_MSPCONFIGPATH=/etc/hyperledger/crypto/peer/msp
      - CORE_PEER_TLS_ENABLED=true
      - CORE_PEER_TLS_KEY_FILE=/etc/hyperledger/crypto/peer/tls/server.key
      - CORE_PEER_TLS_CERT_FILE=/etc/hyperledger/crypto/peer/tls/server.crt
      - CORE_PEER_TLS_ROOTCERT_FILE=/etc/hyperledger/crypto/peer/tls/ca.crt
    working_dir: /opt/gopath/src/github.com/hyperledger/fabric/peer
    command: peer node start
    volumes:
        - /var/run/:/host/var/run/
```

可以看到，我们在 `base.yaml` 文件中主要指定了 peer 容器相关的一些公共的环境参数，后继我们需要使用到。

##### 创建 docker-compose.yaml 文件并编辑

```js
$ vim docker-compose.yaml 
```

`docker-compose.yaml` 文件完整内容如下：

```js
#
# Copyright IBM Corp. All Rights Reserved.
#
# SPDX-License-Identifier: Apache-2.0
#
version: '2'

services:

  ca.org1.kevin.kongyixueyuan.com:
    image: hyperledger/fabric-ca
    environment:
      - FABRIC_CA_HOME=/etc/hyperledger/fabric-ca-server
      - FABRIC_CA_SERVER_CA_NAME=ca-org1
      - FABRIC_CA_SERVER_CA_CERTFILE=/etc/hyperledger/fabric-ca-server-config/ca.org1.kevin.kongyixueyuan.com-cert.pem
      - FABRIC_CA_SERVER_CA_KEYFILE=/etc/hyperledger/fabric-ca-server-config/3361e8b40d7226a425978bbbd403f310b3710dec3e5ca785d5aabc227196c933_sk
      - FABRIC_CA_SERVER_TLS_ENABLED=true
      - FABRIC_CA_SERVER_TLS_CERTFILE=/etc/hyperledger/fabric-ca-server-config/ca.org1.kevin.kongyixueyuan.com-cert.pem
      - FABRIC_CA_SERVER_TLS_KEYFILE=/etc/hyperledger/fabric-ca-server-config/3361e8b40d7226a425978bbbd403f310b3710dec3e5ca785d5aabc227196c933_sk
    ports:
      - "7054:7054"
    command: sh -c 'fabric-ca-server start -b admin:adminpw -d'
    volumes:
      - ./channel/crypto-config/peerOrganizations/org1.kevin.kongyixueyuan.com/ca/:/etc/hyperledger/fabric-ca-server-config
    container_name: ca_peerOrg1

  ca.org2.kevin.kongyixueyuan.com:
    image: hyperledger/fabric-ca
    environment:
      - FABRIC_CA_HOME=/etc/hyperledger/fabric-ca-server
      - FABRIC_CA_SERVER_CA_NAME=ca-org2
      - FABRIC_CA_SERVER_CA_CERTFILE=/etc/hyperledger/fabric-ca-server-config/ca.org2.kevin.kongyixueyuan.com-cert.pem
      - FABRIC_CA_SERVER_CA_KEYFILE=/etc/hyperledger/fabric-ca-server-config/7d800a7f792c874f013573376875389dc4f3f4d47534b749c2b6a44269b49f5b_sk
      - FABRIC_CA_SERVER_TLS_ENABLED=true
      - FABRIC_CA_SERVER_TLS_CERTFILE=/etc/hyperledger/fabric-ca-server-config/ca.org2.kevin.kongyixueyuan.com-cert.pem
      - FABRIC_CA_SERVER_TLS_KEYFILE=/etc/hyperledger/fabric-ca-server-config/7d800a7f792c874f013573376875389dc4f3f4d47534b749c2b6a44269b49f5b_sk
    ports:
      - "8054:7054"
    command: sh -c 'fabric-ca-server start -b admin:adminpw -d'
    volumes:
      - ./channel/crypto-config/peerOrganizations/org2.kevin.kongyixueyuan.com/ca/:/etc/hyperledger/fabric-ca-server-config
    container_name: ca_peerOrg2

  orderer.kevin.kongyixueyuan.com:
    container_name: orderer.kevin.kongyixueyuan.com
    image: hyperledger/fabric-orderer
    environment:
      - ORDERER_GENERAL_LOGLEVEL=debug
      - ORDERER_GENERAL_LISTENADDRESS=0.0.0.0
      - ORDERER_GENERAL_GENESISMETHOD=file
      - ORDERER_GENERAL_GENESISFILE=/etc/hyperledger/configtx/genesis.block
      - ORDERER_GENERAL_LOCALMSPID=kevin.kongyixueyuan.com
      - ORDERER_GENERAL_LOCALMSPDIR=/etc/hyperledger/crypto/orderer/msp
      - ORDERER_GENERAL_TLS_ENABLED=true
      - ORDERER_GENERAL_TLS_PRIVATEKEY=/etc/hyperledger/crypto/orderer/tls/server.key
      - ORDERER_GENERAL_TLS_CERTIFICATE=/etc/hyperledger/crypto/orderer/tls/server.crt
      - ORDERER_GENERAL_TLS_ROOTCAS=[/etc/hyperledger/crypto/orderer/tls/ca.crt, /etc/hyperledger/crypto/peerOrg1/tls/ca.crt, /etc/hyperledger/crypto/peerOrg2/tls/ca.crt]
    working_dir: /opt/gopath/src/github.com/hyperledger/fabric/orderers
    command: orderer
    ports:
      - 7050:7050
    volumes:
        - ./channel:/etc/hyperledger/configtx
        - ./channel/crypto-config/ordererOrganizations/kevin.kongyixueyuan.com/orderers/orderer.kevin.kongyixueyuan.com/:/etc/hyperledger/crypto/orderer
        - ./channel/crypto-config/peerOrganizations/org1.kevin.kongyixueyuan.com/peers/peer0.org1.kevin.kongyixueyuan.com/:/etc/hyperledger/crypto/peerOrg1
        - ./channel/crypto-config/peerOrganizations/org2.kevin.kongyixueyuan.com/peers/peer0.org2.kevin.kongyixueyuan.com/:/etc/hyperledger/crypto/peerOrg2

  peer0.org1.kevin.kongyixueyuan.com:
    container_name: peer0.org1.kevin.kongyixueyuan.com
    extends:
      file:   base.yaml
      service: peer-base
    environment:
      - CORE_PEER_ID=peer0.org1.kevin.kongyixueyuan.com
      - CORE_PEER_LOCALMSPID=org1.kevin.kongyixueyuan.com
      - CORE_PEER_ADDRESS=peer0.org1.kevin.kongyixueyuan.com:7051
    ports:
      - 7051:7051
      - 7053:7053
    volumes:
        - ./channel/crypto-config/peerOrganizations/org1.kevin.kongyixueyuan.com/peers/peer0.org1.kevin.kongyixueyuan.com/:/etc/hyperledger/crypto/peer
    depends_on:
      - orderer.kevin.kongyixueyuan.com

  peer1.org1.kevin.kongyixueyuan.com:
    container_name: peer1.org1.kevin.kongyixueyuan.com
    extends:
      file:   base.yaml
      service: peer-base
    environment:
      - CORE_PEER_ID=peer1.org1.kevin.kongyixueyuan.com
      - CORE_PEER_LOCALMSPID=org1.kevin.kongyixueyuan.com
      - CORE_PEER_ADDRESS=peer1.org1.kevin.kongyixueyuan.com:7051
    ports:
      - 7056:7051
      - 7058:7053
    volumes:
        - ./channel/crypto-config/peerOrganizations/org1.kevin.kongyixueyuan.com/peers/peer1.org1.kevin.kongyixueyuan.com/:/etc/hyperledger/crypto/peer
    depends_on:
      - orderer.kevin.kongyixueyuan.com

  peer0.org2.kevin.kongyixueyuan.com:
    container_name: peer0.org2.kevin.kongyixueyuan.com
    extends:
      file:   base.yaml
      service: peer-base
    environment:
      - CORE_PEER_ID=peer0.org2.kevin.kongyixueyuan.com
      - CORE_PEER_LOCALMSPID=org2.kevin.kongyixueyuan.com
      - CORE_PEER_ADDRESS=peer0.org2.kevin.kongyixueyuan.com:7051
    ports:
      - 8051:7051
      - 8053:7053
    volumes:
        - ./channel/crypto-config/peerOrganizations/org2.kevin.kongyixueyuan.com/peers/peer0.org2.kevin.kongyixueyuan.com/:/etc/hyperledger/crypto/peer
    depends_on:
      - orderer.kevin.kongyixueyuan.com

  peer1.org2.kevin.kongyixueyuan.com:
    container_name: peer1.org2.kevin.kongyixueyuan.com
    extends:
      file:   base.yaml
      service: peer-base
    environment:
      - CORE_PEER_ID=peer1.org2.kevin.kongyixueyuan.com
      - CORE_PEER_LOCALMSPID=org2.kevin.kongyixueyuan.com
      - CORE_PEER_ADDRESS=peer1.org2.kevin.kongyixueyuan.com:7051
    ports:
      - 8056:7051
      - 8058:7053
    volumes:
        - ./channel/crypto-config/peerOrganizations/org2.kevin.kongyixueyuan.com/peers/peer1.org2.kevin.kongyixueyuan.com/:/etc/hyperledger/crypto/peer
    depends_on:
      - orderer.kevin.kongyixueyuan.com
```

完成上面的相关配置后，我们将具有以下 `docker` 容器配置的本地网络环境：

*   2 个 CA 节点.
*   一个 SOLO Orderer 节点
*   4 个 Peer 节点（每个 Org 有 2 个 peer 节点）

#### Artifacts

*   Crypto 材料已使用 Hyperledger Fabric 中的**cryptogen**工具生成，并安装到所有对等端，orderering 节点和 CA 容器。有关加密工具的更多详细信息，请[点击此处](http://hyperledger-fabric.readthedocs.io/en/latest/build_network.html#crypto-generator)。
*   Orderer 创世块（genesis.block）和通道配置事务（mychannel.tx）已使用 Hyperledger Fabric 中的**configtxgen**工具预生成，并放置在 `artifacts` 文件夹中。有关 `configtxgen` 工具的更多详细信息，请 [访问此处](http://hyperledger-fabric.readthedocs.io/en/latest/build_network.html#configuration-transaction-generator)。

#### 测试网络环境

为了检查网络是否正常工作，我们可以使用 `docker-compose` 命令同时启动或停止所有容器。 进入`artifacts`文件夹，运行启动或停止的相关命令。

*   启动网络

    ```js
    $ cd $HOME/kevin-fabric-sdk-node/artifacts
    $ docker-compose up -d
    ```

*   查看 docker 的活动容器

    ```js
    $ docker ps
    ```

    ![docker-ps](img/3a293d5223a64e0c3b26c66045376970.jpg)

*   关闭网络

    ```js
    $ docker-compose down
    ```

    > 注意：使用停止网络的命令时，如果没有指定对应的 yaml 文件所在路径，那么，该命令必须在 `docker-compose.yaml` 或 `docker-compose.yml` 文件所在目录下执行。

### 参考资料

*   [Hyperledger 官网](https://www.hyperledger.org/)

*   [Hyperledger Fabric 在线文档](https://hyperledger-fabric.readthedocs.io/en/latest//)

*   [Hyperledger Fabric-SDK-Node](https://github.com/hyperledger/fabric-sdk-node)

*   [Node SDK documentation](https://fabric-sdk-node.github.io/)

*   [fabric-samples balance-transfer](https://github.com/hyperledger/fabric-samples/tree/release/balance-transfer)