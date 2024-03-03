# 2.1 设置环境-网络连接信息

### 指定依赖

进入 marbles 项目根目录:

```js
$ cd $HOME/kevin-marbles 
```

创建 package.json 文件并编辑

```js
$ vim package.json 
```

文件主要指定 marbles 项目的依赖第三方库及相关的版本号。

package.json 文件完整内容如下：

```js
{
    "name": "marbles",
    "version": "4.0.0",
    "description": "A demonstration Node.js web application on Hyperledger Fabric.fabric-sdk-node",
    "main": "app.js",
    "scripts": {
        "start": "node app.js"
    },
    "dependencies": {
        "async": "2.5.*",
        "compression": "1.7.*",
        "cookie-parser": "1.4.*",
        "cors": "2.8.*",
        "express": "4.14.*",
        "express-session": "1.14.*",
        "fabric-ca-client": "1.0.5",
        "fabric-client": "1.0.5",
        "grpc": "1.10.1",
        "pug": "2.0.3",
        "serve-static": "1.13.*",
        "winston": "2.4.*",
        "ws": "1.1.5"
    },
    "engines": {
        "node": "⁶.2.0"
    },
    "license": "Apache-2.0"
} 
```

#### 下载依赖

我们需要一些弹珠依赖项来运行安装/实例化脚本。通过导航回弹珠目录的根目录并输入这些命令来安装大理石 npm 依赖项。如果您已经运行了这些命令，则可以再次运行它们

```js
$ cd ~/kevin-marbles
$ npm install 
```

### 应用网络环境

为了保证应用能够正确访问网络，我们会在项目根目录下创建一个存储配置信息的 config 目录。该目录用于存储所有与网络连接相关的信息配置文件。

创建 config 目录并进入该目录

```js
$ cd $HOME/kevin-marbles
$ mkdir config && cd config 
```

#### 配置网络连接信息

在 config 目录中创建一个名为 connection_profile_local.json 的文件并编辑，由该文件中的信息指定区块链网络中各节点的相关信息

```js
$ vim connection_profile_local.json 
```

在 connection_profile_local.json 文件中指定配置了如下内容：

*   **应用程序组织名称**
*   **SDK 要使用的键值存储的路径（用于存储加密材料）**
*   **通道信息**
*   **organizations 信息**
*   **peers 信息**
*   **orderers 信息**
*   **CA 信息**

connection_profile_local.json 文件完整内容如下：

```js
{
    "name": "Docker Compose Network",
    "x-networkId": "not-important",
    "x-type": "hlfv1",
    "description": "Connection Profile for an Hyperledger Fabric network on a local machine",
    "version": "1.0.0",
    "client": {
        "organization": "org1.kevin.chaindesk.cn",
        "credentialStore": {
            "path": "../hfc-key-store"
        }
    },
    "channels": {
        "kevinchaindesk": {
            "orderers": [
                "fabric-orderer"
            ],
            "peers": {
                "fabric-peer-org1": {
                    "x-chaincode": {}
                }
            },
            "chaincodes": [
                "marbles:v4"
            ],
            "x-blockDelay": 10000
        }
    },
    "organizations": {
        "org1.kevin.chaindesk.cn": {
            "mspid": "org1.kevin.chaindesk.cn",
            "peers": [
                "fabric-peer-org1"
            ],
            "certificateAuthorities": [
                "fabric-ca"
            ],
            "x-adminCert": {
                "path": "../artifacts/crypto-config/peerOrganizations/org1.kevin.chaindesk.cn/users/Admin@org1.kevin.chaindesk.cn/msp/admincerts/Admin@org1.kevin.chaindesk.cn-cert.pem"
            },
            "x-adminKeyStore": {
                "path": "../artifacts/crypto-config/peerOrganizations/org1.kevin.chaindesk.cn/users/Admin@org1.kevin.chaindesk.cn/msp/keystore/"
            }
        }
    },
    "orderers": {
        "fabric-orderer": {
            "url": "grpc://localhost:7050"
        }
    },
    "peers": {
        "fabric-peer-org1": {
            "url": "grpc://localhost:7051",
            "eventUrl": "grpc://localhost:7053"
        }
    },
    "certificateAuthorities": {
        "fabric-ca": {
            "url": "http://localhost:7054",
            "httpOptions": {
                "verify": true
            },
            "registrar": [
                {
                    "enrollId": "admin",
                    "enrollSecret": "adminpw"
                }
            ],
            "caName": null
        }
    }
} 
```

### 启动设置文件 marbles_local.json

创建 marbles_local.json 文件并编辑

```js
$ vim marbles_local.json 
```

该文件是应用项目启动时的指定使用文件，其中在文件中分别指定配置的连接概要文件名称，公司名称，初始 Marble 拥有者名称，应用项目启动时的监听端口号等信息。

文件完整内容如下：

```js
{
    "cred_filename": "connection_profile_local.json",
    "use_events": true,
    "keep_alive_secs": 120,
    "company": "ChainDesk Marbles",
    "usernames": [
        "Hanxiaodong",
        "Lixu",
        "Hanru"
    ],
    "port": 3000
} 
```