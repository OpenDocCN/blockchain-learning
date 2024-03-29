# 3.3 一分钟启动我们的分布式网络

## 目标

1.  深入理解 Hyperledger Fabric 网络启动过程
2.  掌握网络启动命令及其所需参数

## 任务实现

> 网络启动之前所需的所有内容我们已经准备就绪，下面我们深入分析网络中各节点运行时所需要指定的必备信息

### 3.3.1 网络服务如何配置

启动网络，就是启动提供网络服务的各个节点。那么这些节点如何启动，需要哪些信息，由于要启动多个网络节点，Hyperledger Fabric 采用了容器技术，所以需要一个简化的方式来集中化管理这这些节点容器，我们使用 docker-compose 这个工具个来实现一步到位的节点容器管理，实现方式只需要编写相应的配置文件即可。

Hyperledger Fabric 同样给我们提供了一个 docker-compose 工具的示例配置文件，该配置文件在 fabric-samples/first-network 目录下，文件名称为： docker-compose-cli.yaml， 我们打开这个配置文件可以看到如下内容：

```go
version: '2'

volumes:
  orderer.example.com:
  peer0.org1.example.com:
  peer1.org1.example.com:
  peer0.org2.example.com:
  peer1.org2.example.com:

networks:
  byfn:

services:

  orderer.example.com:
    extends:
      file:   base/docker-compose-base.yaml
      service: orderer.example.com
    container_name: orderer.example.com
    networks:
      - byfn

  peer0.org1.example.com:
    container_name: peer0.org1.example.com
    extends:
      file:  base/docker-compose-base.yaml
      service: peer0.org1.example.com
    networks:
      - byfn

  peer1.org1.example.com:
    container_name: peer1.org1.example.com
    extends:
      file:  base/docker-compose-base.yaml
      service: peer1.org1.example.com
    networks:
      - byfn

  peer0.org2.example.com:
    container_name: peer0.org2.example.com
    extends:
      file:  base/docker-compose-base.yaml
      service: peer0.org2.example.com
    networks:
      - byfn

  peer1.org2.example.com:
    container_name: peer1.org2.example.com
    extends:
      file:  base/docker-compose-base.yaml
      service: peer1.org2.example.com
    networks:
      - byfn

  cli:
    container_name: cli
    image: hyperledger/fabric-tools:$IMAGE_TAG
    tty: true
    stdin_open: true
    environment:
      - GOPATH=/opt/gopath
      - CORE_VM_ENDPOINT=unix:///host/var/run/docker.sock
      #- CORE_LOGGING_LEVEL=DEBUG
      - CORE_LOGGING_LEVEL=INFO
      - CORE_PEER_ID=cli
      - CORE_PEER_ADDRESS=peer0.org1.example.com:7051
      - CORE_PEER_LOCALMSPID=Org1MSP
      - CORE_PEER_TLS_ENABLED=true
      - CORE_PEER_TLS_CERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/server.crt
      - CORE_PEER_TLS_KEY_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/server.key
      - CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt
      - CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
    working_dir: /opt/gopath/src/github.com/hyperledger/fabric/peer
    command: /bin/bash
    volumes:
        - /var/run/:/host/var/run/
        - ./../chaincode/:/opt/gopath/src/github.com/chaincode
        - ./crypto-config:/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/
        - ./scripts:/opt/gopath/src/github.com/hyperledger/fabric/peer/scripts/
        - ./channel-artifacts:/opt/gopath/src/github.com/hyperledger/fabric/peer/channel-artifacts
    depends_on:
      - orderer.example.com
      - peer0.org1.example.com
      - peer1.org1.example.com
      - peer0.org2.example.com
      - peer1.org2.example.com
    networks:
      - byfn 
```

该配置文件中指定了网络中各个节点容器（共计六个容器，一个 Orderer，属于两个 Orgs 组织的四个 Peer，还有一个 CLI）的信息；我们仔细观察会发现 orderer 与各 peer 容器都设置了 container_name 与 networks 信息；其它信息都由 extends 指向了 base/docker-compose-base.yaml 文件。

CLI 容器指定了所代表的 peer 节点（CORE_PEER_ADDRESS=peer0.org1.example.com:7051），通过 **volumes** 指定了将系统中的链码、组织结构及证书、生成的配置文件映射到容器中指定的目录下。且通过 **depends_on** 指定了所依赖的相关容器。

### 3.3.2 关联的 docker-compose-base.yaml

配置文件

```go
version: '2'

services:

  orderer.example.com:
    container_name: orderer.example.com
    image: hyperledger/fabric-orderer:$IMAGE_TAG
    environment:
      - ORDERER_GENERAL_LOGLEVEL=INFO
      - ORDERER_GENERAL_LISTENADDRESS=0.0.0.0
      - ORDERER_GENERAL_GENESISMETHOD=file
      - ORDERER_GENERAL_GENESISFILE=/var/hyperledger/orderer/orderer.genesis.block
      - ORDERER_GENERAL_LOCALMSPID=OrdererMSP
      - ORDERER_GENERAL_LOCALMSPDIR=/var/hyperledger/orderer/msp
      # enabled TLS
      - ORDERER_GENERAL_TLS_ENABLED=true
      - ORDERER_GENERAL_TLS_PRIVATEKEY=/var/hyperledger/orderer/tls/server.key
      - ORDERER_GENERAL_TLS_CERTIFICATE=/var/hyperledger/orderer/tls/server.crt
      - ORDERER_GENERAL_TLS_ROOTCAS=[/var/hyperledger/orderer/tls/ca.crt]
    working_dir: /opt/gopath/src/github.com/hyperledger/fabric
    command: orderer
    volumes:
    - ../channel-artifacts/genesis.block:/var/hyperledger/orderer/orderer.genesis.block
    - ../crypto-config/ordererOrganizations/example.com/orderers/orderer.example.com/msp:/var/hyperledger/orderer/msp
    - ../crypto-config/ordererOrganizations/example.com/orderers/orderer.example.com/tls/:/var/hyperledger/orderer/tls
    - orderer.example.com:/var/hyperledger/production/orderer
    ports:
      - 7050:7050

  peer0.org1.example.com:
    container_name: peer0.org1.example.com
    extends:
      file: peer-base.yaml
      service: peer-base
    environment:
      - CORE_PEER_ID=peer0.org1.example.com
      - CORE_PEER_ADDRESS=peer0.org1.example.com:7051
      - CORE_PEER_GOSSIP_BOOTSTRAP=peer1.org1.example.com:7051
      - CORE_PEER_GOSSIP_EXTERNALENDPOINT=peer0.org1.example.com:7051
      - CORE_PEER_LOCALMSPID=Org1MSP
    volumes:
        - /var/run/:/host/var/run/
        - ../crypto-config/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/msp:/etc/hyperledger/fabric/msp
        - ../crypto-config/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls:/etc/hyperledger/fabric/tls
        - peer0.org1.example.com:/var/hyperledger/production
    ports:
      - 7051:7051
      - 7053:7053

  peer1.org1.example.com:
    container_name: peer1.org1.example.com
    extends:
      file: peer-base.yaml
      service: peer-base
    environment:
      - CORE_PEER_ID=peer1.org1.example.com
      - CORE_PEER_ADDRESS=peer1.org1.example.com:7051
      - CORE_PEER_GOSSIP_EXTERNALENDPOINT=peer1.org1.example.com:7051
      - CORE_PEER_GOSSIP_BOOTSTRAP=peer0.org1.example.com:7051
      - CORE_PEER_LOCALMSPID=Org1MSP
    volumes:
        - /var/run/:/host/var/run/
        - ../crypto-config/peerOrganizations/org1.example.com/peers/peer1.org1.example.com/msp:/etc/hyperledger/fabric/msp
        - ../crypto-config/peerOrganizations/org1.example.com/peers/peer1.org1.example.com/tls:/etc/hyperledger/fabric/tls
        - peer1.org1.example.com:/var/hyperledger/production

    ports:
      - 8051:7051
      - 8053:7053

  peer0.org2.example.com:
    container_name: peer0.org2.example.com
    extends:
      file: peer-base.yaml
      service: peer-base
    environment:
      - CORE_PEER_ID=peer0.org2.example.com
      - CORE_PEER_ADDRESS=peer0.org2.example.com:7051
      - CORE_PEER_GOSSIP_EXTERNALENDPOINT=peer0.org2.example.com:7051
      - CORE_PEER_GOSSIP_BOOTSTRAP=peer1.org2.example.com:7051
      - CORE_PEER_LOCALMSPID=Org2MSP
    volumes:
        - /var/run/:/host/var/run/
        - ../crypto-config/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/msp:/etc/hyperledger/fabric/msp
        - ../crypto-config/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls:/etc/hyperledger/fabric/tls
        - peer0.org2.example.com:/var/hyperledger/production
    ports:
      - 9051:7051
      - 9053:7053

  peer1.org2.example.com:
    container_name: peer1.org2.example.com
    extends:
      file: peer-base.yaml
      service: peer-base
    environment:
      - CORE_PEER_ID=peer1.org2.example.com
      - CORE_PEER_ADDRESS=peer1.org2.example.com:7051
      - CORE_PEER_GOSSIP_EXTERNALENDPOINT=peer1.org2.example.com:7051
      - CORE_PEER_GOSSIP_BOOTSTRAP=peer0.org2.example.com:7051
      - CORE_PEER_LOCALMSPID=Org2MSP
    volumes:
        - /var/run/:/host/var/run/
        - ../crypto-config/peerOrganizations/org2.example.com/peers/peer1.org2.example.com/msp:/etc/hyperledger/fabric/msp
        - ../crypto-config/peerOrganizations/org2.example.com/peers/peer1.org2.example.com/tls:/etc/hyperledger/fabric/tls
        - peer1.org2.example.com:/var/hyperledger/production
    ports:
      - 10051:7051
      - 10053:7053 
```

该配置文件中指定了 Orderer 与 Peers 节点的相关信息。

Orderer 设置如下信息：

*   **environment：**指定日志级别、监听地址、生成初始区块的提供方式、初始区块配置文件路径、本地 MSPID 及对应的目录、开启 TLS 验证及对应的证书、私钥信息等诸多重要信息。
*   **working_dir：**进入容器后的默认工作目录
*   **volumes：**指定系统中的初始区块配置文件、MSP、TLS 目录映射到容器中的指定路径下。
*   **ports：** 指定当前节点的监听端口。

各 Peers 设置了如下信息：

*   **extends：**基本信息来源于哪个文件。

*   **environment：**指定了容器的的 ID、监听地址及端口号、本地 MSPID。
*   **volumes：**将系统的 msp 及 tls 目录映射到容器中的指定路径下。
*   **ports：** 指定当前节点的监听端口。

### 3.3.3 又被关联的 peer-base.yaml

配置文件内容如下:

```go
version: '2'

services:
  peer-base:
    image: hyperledger/fabric-peer:$IMAGE_TAG
    environment:
      - CORE_VM_ENDPOINT=unix:///host/var/run/docker.sock
      # the following setting starts chaincode containers on the same
      # bridge network as the peers
      # https://docs.docker.com/compose/networking/
      - CORE_VM_DOCKER_HOSTCONFIG_NETWORKMODE=${COMPOSE_PROJECT_NAME}_byfn
      - CORE_LOGGING_LEVEL=INFO
      #- CORE_LOGGING_LEVEL=DEBUG
      - CORE_PEER_TLS_ENABLED=true
      - CORE_PEER_GOSSIP_USELEADERELECTION=true
      - CORE_PEER_GOSSIP_ORGLEADER=false
      - CORE_PEER_PROFILE_ENABLED=true
      - CORE_PEER_TLS_CERT_FILE=/etc/hyperledger/fabric/tls/server.crt
      - CORE_PEER_TLS_KEY_FILE=/etc/hyperledger/fabric/tls/server.key
      - CORE_PEER_TLS_ROOTCERT_FILE=/etc/hyperledger/fabric/tls/ca.crt
    working_dir: /opt/gopath/src/github.com/hyperledger/fabric/peer
    command: peer node start 
```

该配置文件设置了所有 peer 容器的基本的共同信息，日志级别，是否开启 TLS 验证，是否采用 Leader 选举， 是否将当前节点设为 Leader， TLS 证书、私钥、根证书的路径、容器的默认工作路径、容器启动命令。

### 3.3.4 启动网络

万事具备，只欠东风，下面我们通过一条命令来方便的启动 Hyperledger Fabric 网络中所有节点。

```go
$ sudo docker-compose -f docker-compose-cli.yaml up -d 
```

命令执行后终端输出如下：

```go
Creating network "net_byfn" with the default driver
Creating volume "net_peer0.org2.example.com" with default driver
Creating volume "net_peer1.org2.example.com" with default driver
Creating volume "net_peer1.org1.example.com" with default driver
Creating volume "net_peer0.org1.example.com" with default driver
Creating volume "net_orderer.example.com" with default driver
Creating orderer.example.com
Creating peer1.org2.example.com
Creating peer0.org2.example.com
Creating peer1.org1.example.com
Creating peer0.org1.example.com
Creating cli 
```

docker-compose 命令可以加许多的子命令，从而实现不同的操作， `up` 子命令是根据指定的配置文件启动相应的容器（网络环境）。还有一个 `down` 子命令则是关闭已启动的容器（网络环境）。如：

*   使用指定的 docker-compose-cli.yaml 配置文件关闭网络：

    ```go
    $ sudo docker-compose -f docker-compose-cli.yaml down 
    ```

**参数说明：**

*   **-f：** 指定启动容器时用所使用的 docker-compose 配置文件。

*   **-d：** 指定是否显示网络启动过程中的实时日志信息，如果需要查看详细网络启动日志，则可以不提供此参数。

网络启动顺序：首先启动 Orderer 服务节点，然后启动 Peer 节点，终端输出日志内容如下:

```go
......
orderer.example.com       | 02:48:25.080 UTC [orderer/common/server] initializeServerConfig -> INFO 002 Starting orderer with TLS enabled
orderer.example.com       | 02:48:25.101 UTC [fsblkstorage] newBlockfileMgr -> INFO 003 Getting block information from block storage
orderer.example.com       | 02:48:25.138 UTC [orderer/commmon/multichannel] NewRegistrar -> INFO 004 Starting system channel 'testchainid' with genesis block hash 67662e918ab76b4a8863cc625d67fcc31e9cb3a7c3c4f9f707af1c05ba5be686 and orderer type solo
orderer.example.com       | 02:48:25.138 UTC [orderer/common/server] Start -> INFO 005 Starting orderer:
...... 
```

> Peer 节点启动后，默认情况下没有加入网络中的任何应用通道，也不会与 Orderer 服务建立连接。需要通过客户端对其进行操作，让它加入网络和指定的应用通道中。

启动会我们使用如下命令查看网络信息：

```go
$ sudo docker ps 
```

命令执行后会发现有六个容器处于活动状态（分别为：cli、peer1.org1.example.com、peer0.org1.example.com、peer1.org2.example.com、orderer.example.com、peer0.org2.example.com），说明网络启动成功。

## FAQ

1.  启动网络报错误怎么办？

    如果在启动网络时没有使用 -d 参数，那么在启动后会输出如下图所示的错误信息：

    ![启动网络的错误信息](img/dee3a3b306a848855c5aec9b93d99d03.jpg)

    此错误信息对于我们后期操作没有影响，可以无需理会。

    如果在查看详细日志时发现有错误，那么就需要根据对应的错误提示信息进行处理。注：红色的内容并不代表错误，一般是为了方便区分各节点而使用不同的颜色。