# 第二章 Hyperledger Fabric 的链码实现

## 从零到壹构建基于 fabric-sdk-node 的项目开发实战之二

### 编写链码

对于一个完整的应用而言，操作账本状态是通过链码来实现的，所以我们必须编写链码来实现对账本状态的操作

*   在此，我们不去考虑多么复杂的业务，只是实现一个简单的转账及余额查询的功能
*   另外，链代码可以使用不同的语言来编写实现，我们在示例中使用 `Golang`
*   其它语言，如： `node`、`java` 等语言如何编写链代码在此我们不讨论

在使用 `Golang` 编写链码之前，我们需要创建存放链代码文件的目录，然后创建链代码文件并编辑：

```js
$ cd $HOME/kevin-fabric-sdk-node/artifacts
$ mkdir -p src/github.com/example_cc
$ cd src/github.com/example_cc
$ vim example_cc.go
```

`example_cc.go` 文件完整内容如下：

```js
/*
Copyright IBM Corp. 2016 All Rights Reserved.
Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at
         http://www.apache.org/licenses/LICENSE-2.0
Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
*/
package main

import (
    "fmt"
    "strconv"

    "github.com/hyperledger/fabric/core/chaincode/shim"
    pb "github.com/hyperledger/fabric/protos/peer"
)

var logger = shim.NewLogger("example_cc0")

// SimpleChaincode example simple Chaincode implementation
type SimpleChaincode struct {
}

func (t *SimpleChaincode) Init(stub shim.ChaincodeStubInterface) pb.Response  {
    logger.Info("########### example_cc0 Init ###########")

    _, args := stub.GetFunctionAndParameters()
    var A, B string    // Entities
    var Aval, Bval int // Asset holdings
    var err error

    // Initialize the chaincode
    A = args[0]
    Aval, err = strconv.Atoi(args[1])
    if err != nil {
        return shim.Error("Expecting integer value for asset holding")
    }
    B = args[2]
    Bval, err = strconv.Atoi(args[3])
    if err != nil {
        return shim.Error("Expecting integer value for asset holding")
    }
    logger.Info("Aval = %d, Bval = %d\n", Aval, Bval)

    // Write the state to the ledger
    err = stub.PutState(A, []byte(strconv.Itoa(Aval)))
    if err != nil {
        return shim.Error(err.Error())
    }

    err = stub.PutState(B, []byte(strconv.Itoa(Bval)))
    if err != nil {
        return shim.Error(err.Error())
    }

    return shim.Success(nil)
}

// Transaction makes payment of X units from A to B
func (t *SimpleChaincode) Invoke(stub shim.ChaincodeStubInterface) pb.Response {
    logger.Info("########### example_cc0 Invoke ###########")

    function, args := stub.GetFunctionAndParameters()

    if function == "delete" {
        // Deletes an entity from its state
        return t.delete(stub, args)
    }

    if function == "query" {
        // queries an entity state
        return t.query(stub, args)
    }
    if function == "move" {
        // Deletes an entity from its state
        return t.move(stub, args)
    }

    logger.Errorf("Unknown action, check the first argument, must be one of 'delete', 'query', or 'move'. But got: %v", args[0])
    return shim.Error(fmt.Sprintf("Unknown action, check the first argument, must be one of 'delete', 'query', or 'move'. But got: %v", args[0]))
}

func (t *SimpleChaincode) move(stub shim.ChaincodeStubInterface, args []string) pb.Response {
    // must be an invoke
    var A, B string    // Entities
    var Aval, Bval int // Asset holdings
    var X int          // Transaction value
    var err error

    if len(args) != 3 {
        return shim.Error("Incorrect number of arguments. Expecting 4, function followed by 2 names and 1 value")
    }

    A = args[0]
    B = args[1]

    // Get the state from the ledger
    // TODO: will be nice to have a GetAllState call to ledger
    Avalbytes, err := stub.GetState(A)
    if err != nil {
        return shim.Error("Failed to get state")
    }
    if Avalbytes == nil {
        return shim.Error("Entity not found")
    }
    Aval, _ = strconv.Atoi(string(Avalbytes))

    Bvalbytes, err := stub.GetState(B)
    if err != nil {
        return shim.Error("Failed to get state")
    }
    if Bvalbytes == nil {
        return shim.Error("Entity not found")
    }
    Bval, _ = strconv.Atoi(string(Bvalbytes))

    // Perform the execution
    X, err = strconv.Atoi(args[2])
    if err != nil {
        return shim.Error("Invalid transaction amount, expecting a integer value")
    }
    Aval = Aval - X
    Bval = Bval + X
    logger.Infof("Aval = %d, Bval = %d\n", Aval, Bval)

    // Write the state back to the ledger
    err = stub.PutState(A, []byte(strconv.Itoa(Aval)))
    if err != nil {
        return shim.Error(err.Error())
    }

    err = stub.PutState(B, []byte(strconv.Itoa(Bval)))
    if err != nil {
        return shim.Error(err.Error())
    }

        return shim.Success(nil);
}

// Deletes an entity from state
func (t *SimpleChaincode) delete(stub shim.ChaincodeStubInterface, args []string) pb.Response {
    if len(args) != 1 {
        return shim.Error("Incorrect number of arguments. Expecting 1")
    }

    A := args[0]

    // Delete the key from the state in ledger
    err := stub.DelState(A)
    if err != nil {
        return shim.Error("Failed to delete state")
    }

    return shim.Success(nil)
}

// Query callback representing the query of a chaincode
func (t *SimpleChaincode) query(stub shim.ChaincodeStubInterface, args []string) pb.Response {

    var A string // Entities
    var err error

    if len(args) != 1 {
        return shim.Error("Incorrect number of arguments. Expecting name of the person to query")
    }

    A = args[0]

    // Get the state from the ledger
    Avalbytes, err := stub.GetState(A)
    if err != nil {
        jsonResp := "{\"Error\":\"Failed to get state for " + A + "\"}"
        return shim.Error(jsonResp)
    }

    if Avalbytes == nil {
        jsonResp := "{\"Error\":\"Nil amount for " + A + "\"}"
        return shim.Error(jsonResp)
    }

    jsonResp := "{\"Name\":\"" + A + "\",\"Amount\":\"" + string(Avalbytes) + "\"}"
    logger.Infof("Query Response:%s\n", jsonResp)
    return shim.Success(Avalbytes)
}

func main() {
    err := shim.Start(new(SimpleChaincode))
    if err != nil {
        logger.Errorf("Error starting Simple chaincode: %s", err)
    }
}
```

### 设置应用环境

#### config.json

返回至项目根目录下

```js
$ cd $HOME/kevin-fabric-sdk-node
```

创建 `config.json` 文件，指定应用相关的信息

```js
$ vim config.json
```

`config.json` 文件内容如下：

```js
{
   "host":"localhost",
   "port":"4000",
   "jwt_expiretime": "36000",
   "channelName":"kevinkongyixueyuan",
   "CC_SRC_PATH":"../artifacts",
   "eventWaitTime":"30000",
   "admins":[
      {
         "username":"admin",
         "secret":"adminpw"
      }
   ]
}
```

#### package.json

创建 `package.json` 文件，指定应用依赖信息

```js
$ vim package.json
```

`package.json` 文件内容如下：

```js
{
  "name": "kevin-fabric-sdk-node",
  "version": "1.0.0",
  "description": "A balance-transfer example node program to demonstrate using node.js SDK APIs",
  "main": "app.js",
  "scripts": {
    "start": "node app.js"
  },
  "keywords": [
    "fabric-client sample app",
    "balance-transfer node sample",
    "v1.0 fabric nodesdk sample"
  ],
  "engines": {
    "node": ">=8.9.4 <9.0",
    "npm": ">=5.6.0 <6.0"
  },
  "license": "Apache-2.0",
  "dependencies": {
    "body-parser": "¹.17.1",
    "cookie-parser": "¹.4.3",
    "cors": "².8.3",
    "express": "⁴.15.2",
    "express-bearer-token": "².1.0",
    "express-jwt": "⁵.1.0",
    "express-session": "¹.15.2",
    "fabric-ca-client": "~1.2.0",
    "fabric-client": "~1.2.0",
    "fs-extra": "².0.0",
    "jsonwebtoken": "⁷.3.0",
    "log4js": "⁰.6.38"
  }
}
```

#### config.js

创建 `config.js` 文件，指定网络配置相关信息

```js
$ vim config.js
```

`config.js` 文件内容如下：

```js
var util = require('util');
var path = require('path');
var hfc = require('fabric-client');

var file = 'network-config%s.yaml';

var env = process.env.TARGET_NETWORK;
if (env)
    file = util.format(file, '-' + env);
else
    file = util.format(file, '');
// indicate to the application where the setup file is located so it able
// to have the hfc load it to initalize the fabric client instance
hfc.setConfigSetting('network-connection-profile-path',path.join(__dirname, 'artifacts' ,file));
hfc.setConfigSetting('Org1-connection-profile-path',path.join(__dirname, 'artifacts', 'org1.yaml'));
hfc.setConfigSetting('Org2-connection-profile-path',path.join(__dirname, 'artifacts', 'org2.yaml'));
// some other settings the application might need to know
hfc.addConfigFile(path.join(__dirname, 'config.json'));
```