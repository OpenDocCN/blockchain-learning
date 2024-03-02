# 一、.2 编写链码

## 从零到壹实现 Marbles 资产管理系统 （Fabric-SDK-Node）之－链码开发

### 编写链码

返回项目根目录下：

```go
$ cd $HOME/kevin-marbles 
```

新建存储链码源文件的所在目录：

```go
$ mkdir -p chaincode/src/marbles
$ cd chaincode/src/marbles 
```

#### 新建链码主文件 marbles.go 并编辑：

链码主文件主要声明与 Marble 相关的结构体，实现链码的接口及根据不同的请求调用不同的函数。

```go
$ vim marbles.go 
```

文件完整内容如下：

```go
/*
Licensed to the Apache Software Foundation (ASF) under one
or more contributor license agreements.  See the NOTICE file
distributed with this work for additional information
regarding copyright ownership.  The ASF licenses this file
to you under the Apache License, Version 2.0 (the
"License"); you may not use this file except in compliance
with the License.  You may obtain a copy of the License at

  http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing,
software distributed under the License is distributed on an
"AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
KIND, either express or implied.  See the License for the
specific language governing permissions and limitations
under the License.
*/

package main

import (
    "fmt"
    "strconv"

    "github.com/hyperledger/fabric/core/chaincode/shim"
    pb "github.com/hyperledger/fabric/protos/peer"
)

type SimpleChaincode struct {
}

// ----- Marbles ----- //
type Marble struct {
    ObjectType string        `json:"docType"` 
    Id       string          `json:"id"`     
    Color      string        `json:"color"`
    Size       int           `json:"size"`  
    Owner      OwnerRelation `json:"owner"`
}

type Owner struct {
    ObjectType string `json:"docType"`
    Id         string `json:"id"`
    Username   string `json:"username"`
    Company    string `json:"company"`
    Enabled    bool   `json:"enabled"`
}

type OwnerRelation struct {
    Id         string `json:"id"`
    Username   string `json:"username"` 
    Company    string `json:"company"`
}

func main() {
    err := shim.Start(new(SimpleChaincode))
    if err != nil {
        fmt.Printf("Error starting Simple chaincode - %s", err)
    }
}

func (t *SimpleChaincode) Init(stub shim.ChaincodeStubInterface) pb.Response {
    fmt.Println("Marbles Is Starting Up")
    funcName, args := stub.GetFunctionAndParameters()
    var number int
    var err error
    txId := stub.GetTxID()

    fmt.Println("Init() is running")
    fmt.Println("Transaction ID:", txId)
    fmt.Println("  GetFunctionAndParameters() function:", funcName)
    fmt.Println("  GetFunctionAndParameters() args count:", len(args))
    fmt.Println("  GetFunctionAndParameters() args found:", args)

    // expecting 1 arg for instantiate or upgrade
    if len(args) == 1 {
        fmt.Println("  GetFunctionAndParameters() arg[0] length", len(args[0]))

        // expecting arg[0] to be length 0 for upgrade
        if len(args[0]) == 0 {
            fmt.Println("  Uh oh, args[0] is empty...")
        } else {
            fmt.Println("  Great news everyone, args[0] is not empty")

            // convert numeric string to integer
            number, err = strconv.Atoi(args[0])
            if err != nil {
                return shim.Error("Expecting a numeric string argument to Init() for instantiate")
            }

            // this is a very simple test. let's write to the ledger and error out on any errors
            // it's handy to read this right away to verify network is healthy if it wrote the correct value
            err = stub.PutState("selftest", []byte(strconv.Itoa(number)))
            if err != nil {
                return shim.Error(err.Error())                  //self-test fail
            }
        }
    }

    // showing the alternative argument shim function
    alt := stub.GetStringArgs()
    fmt.Println("  GetStringArgs() args count:", len(alt))
    fmt.Println("  GetStringArgs() args found:", alt)

    // store compatible marbles application version
    err = stub.PutState("marbles_ui", []byte("4.0.1"))
    if err != nil {
        return shim.Error(err.Error())
    }

    fmt.Println("Ready for action")   //self-test pass
    return shim.Success(nil)
}

func (t *SimpleChaincode) Invoke(stub shim.ChaincodeStubInterface) pb.Response {
    function, args := stub.GetFunctionAndParameters()
    fmt.Println(" ")
    fmt.Println("starting invoke, for - " + function)

    // Handle different functions
    if function == "init" {   //initialize the chaincode state, used as reset
        return t.Init(stub)
    } else if function == "read" {  //generic read ledger
        return read(stub, args)
    } else if function == "write" {  //generic writes to ledger
        return write(stub, args)
    } else if function == "delete_marble" {    //deletes a marble from state
        return delete_marble(stub, args)
    } else if function == "init_marble" {   //create a new marble
        return init_marble(stub, args)
    } else if function == "set_owner" {  //change owner of a marble
        return set_owner(stub, args)
    } else if function == "init_owner"{  //create a new marble owner
        return init_owner(stub, args)
    } else if function == "read_everything"{   //read everything, (owners + marbles + companies)
        return read_everything(stub)
    } else if function == "getHistory"{  //read history of a marble (audit)
        return getHistory(stub, args)
    } else if function == "getMarblesByRange"{ //read a bunch of marbles by start and stop id
        return getMarblesByRange(stub, args)
    } else if function == "disable_owner"{  //disable a marble owner from appearing on the UI
        return disable_owner(stub, args)
    }

    fmt.Println("Received unknown invoke function name - " + function)
    return shim.Error("Received unknown invoke function name - '" + function + "'")
}

func (t *SimpleChaincode) Query(stub shim.ChaincodeStubInterface) pb.Response {
    return shim.Error("Unknown supported call - Query()")
} 
```

#### 创建 lib.go 文件并编辑

lib.go 文件中主要声明并实现了根据指定的 ID 对 Marble 与 Owner 状态查询的公共函数。

```go
$ vim lib.go 
```

文件完整内容如下

```go
/*
Licensed to the Apache Software Foundation (ASF) under one
or more contributor license agreements.  See the NOTICE file
distributed with this work for additional information
regarding copyright ownership.  The ASF licenses this file
to you under the Apache License, Version 2.0 (the
"License"); you may not use this file except in compliance
with the License.  You may obtain a copy of the License at

  http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing,
software distributed under the License is distributed on an
"AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
KIND, either express or implied.  See the License for the
specific language governing permissions and limitations
under the License.
*/

package main

import (
    "encoding/json"
    "errors"
    "strconv"

    "github.com/hyperledger/fabric/core/chaincode/shim"
)

func get_marble(stub shim.ChaincodeStubInterface, id string) (Marble, error) {
    var marble Marble
    marbleAsBytes, err := stub.GetState(id) 
    if err != nil {  
        return marble, errors.New("Failed to find marble - " + id)
    }
    json.Unmarshal(marbleAsBytes, &marble) 

    if marble.Id != id {  
        return marble, errors.New("Marble does not exist - " + id)
    }

    return marble, nil
}

func get_owner(stub shim.ChaincodeStubInterface, id string) (Owner, error) {
    var owner Owner
    ownerAsBytes, err := stub.GetState(id)  
    if err != nil {  
        return owner, errors.New("Failed to get owner - " + id)
    }
    json.Unmarshal(ownerAsBytes, &owner)   

    if len(owner.Username) == 0 {  
        return owner, errors.New("Owner does not exist - " + id + ", '" + owner.Username + "' '" + owner.Company + "'")
    }

    return owner, nil
}

// ========================================================
// Input Sanitation - dumb input checking, look for empty strings
// ========================================================
func sanitize_arguments(strs []string) error{
    for i, val:= range strs {
        if len(val) <= 0 {
            return errors.New("Argument " + strconv.Itoa(i) + " must be a non-empty string")
        }
        if len(val) > 32 {
            return errors.New("Argument " + strconv.Itoa(i) + " must be <= 32 characters")
        }
    }
    return nil
} 
```

#### 创建 read_ledger.go 文件并编辑（查询状态）

```go
$ vim read_ledger.go 
```

文件完整内容如下

```go
/*
Licensed to the Apache Software Foundation (ASF) under one
or more contributor license agreements.  See the NOTICE file
distributed with this work for additional information
regarding copyright ownership.  The ASF licenses this file
to you under the Apache License, Version 2.0 (the
"License"); you may not use this file except in compliance
with the License.  You may obtain a copy of the License at

  http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing,
software distributed under the License is distributed on an
"AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
KIND, either express or implied.  See the License for the
specific language governing permissions and limitations
under the License.
*/

package main

import (
    "bytes"
    "encoding/json"
    "fmt"

    "github.com/hyperledger/fabric/core/chaincode/shim"
    pb "github.com/hyperledger/fabric/protos/peer"
)

// ============================================================================================================================
// Read - read a generic variable from ledger
//
// Shows Off GetState() - reading a key/value from the ledger
//
// Inputs - Array of strings
//  0
//  key
//  "abc"
// 
// Returns - string
// ============================================================================================================================
func read(stub shim.ChaincodeStubInterface, args []string) pb.Response {
    var key, jsonResp string
    var err error
    fmt.Println("starting read")

    if len(args) != 1 {
        return shim.Error("Incorrect number of arguments. Expecting key of the var to query")
    }

    // input sanitation
    err = sanitize_arguments(args)
    if err != nil {
        return shim.Error(err.Error())
    }

    key = args[0]
    valAsbytes, err := stub.GetState(key)  //get the var from ledger
    if err != nil {
        jsonResp = "{\"Error\":\"Failed to get state for " + key + "\"}"
        return shim.Error(jsonResp)
    }

    fmt.Println("- end read")
    return shim.Success(valAsbytes)  //send it onward
}

// ============================================================================================================================
// Get everything we need (owners + marbles + companies)
//
// Inputs - none
//
// Returns:
// {
//    "owners": [{
//            "id": "o99999999",
//            "company": "United Marbles"
//            "username": "alice"
//    }],
//    "marbles": [{
//        "id": "m1490898165086",
//        "color": "white",
//        "docType" :"marble",
//        "owner": {
//            "company": "United Marbles"
//            "username": "alice"
//        },
//        "size" : 35
//    }]
// }
// ============================================================================================================================
func read_everything(stub shim.ChaincodeStubInterface) pb.Response {
    type Everything struct {
        Owners   []Owner   `json:"owners"`
        Marbles  []Marble  `json:"marbles"`
    }
    var everything Everything

    // ---- Get All Marbles ---- //
    resultsIterator, err := stub.GetStateByRange("m0", "m9999999999999999999")
    if err != nil {
        return shim.Error(err.Error())
    }
    defer resultsIterator.Close()

    for resultsIterator.HasNext() {
        aKeyValue, err := resultsIterator.Next()
        if err != nil {
            return shim.Error(err.Error())
        }
        queryKeyAsStr := aKeyValue.Key
        queryValAsBytes := aKeyValue.Value
        fmt.Println("on marble id - ", queryKeyAsStr)
        var marble Marble
        json.Unmarshal(queryValAsBytes, &marble)  //un stringify it aka JSON.parse()
        everything.Marbles = append(everything.Marbles, marble)   //add this marble to the list
    }
    fmt.Println("marble array - ", everything.Marbles)

    // ---- Get All Owners ---- //
    ownersIterator, err := stub.GetStateByRange("o0", "o9999999999999999999")
    if err != nil {
        return shim.Error(err.Error())
    }
    defer ownersIterator.Close()

    for ownersIterator.HasNext() {
        aKeyValue, err := ownersIterator.Next()
        if err != nil {
            return shim.Error(err.Error())
        }
        queryKeyAsStr := aKeyValue.Key
        queryValAsBytes := aKeyValue.Value
        fmt.Println("on owner id - ", queryKeyAsStr)
        var owner Owner
        json.Unmarshal(queryValAsBytes, &owner)  //un stringify it aka JSON.parse()

        if owner.Enabled {  //only return enabled owners
            everything.Owners = append(everything.Owners, owner)  //add this marble to the list
        }
    }
    fmt.Println("owner array - ", everything.Owners)

    //change to array of bytes
    everythingAsBytes, _ := json.Marshal(everything)   //convert to array of bytes
    return shim.Success(everythingAsBytes)
}

// ============================================================================================================================
// Get history of asset
//
// Shows Off GetHistoryForKey() - reading complete history of a key/value
//
// Inputs - Array of strings
//  0
//  id
//  "m01490985296352SjAyM"
// ============================================================================================================================
func getHistory(stub shim.ChaincodeStubInterface, args []string) pb.Response {
    type AuditHistory struct {
        TxId    string   `json:"txId"`
        Value   Marble   `json:"value"`
    }
    var history []AuditHistory;
    var marble Marble

    if len(args) != 1 {
        return shim.Error("Incorrect number of arguments. Expecting 1")
    }

    marbleId := args[0]
    fmt.Printf("- start getHistoryForMarble: %s\n", marbleId)

    // Get History
    resultsIterator, err := stub.GetHistoryForKey(marbleId)
    if err != nil {
        return shim.Error(err.Error())
    }
    defer resultsIterator.Close()

    for resultsIterator.HasNext() {
        historyData, err := resultsIterator.Next()
        if err != nil {
            return shim.Error(err.Error())
        }

        var tx AuditHistory
        tx.TxId = historyData.TxId   //copy transaction id over
        json.Unmarshal(historyData.Value, &marble)     //un stringify it aka JSON.parse()
        if historyData.Value == nil {  //marble has been deleted
            var emptyMarble Marble
            tx.Value = emptyMarble   //copy nil marble
        } else {
            json.Unmarshal(historyData.Value, &marble) //un stringify it aka JSON.parse()
            tx.Value = marble    //copy marble over
        }
        history = append(history, tx)   //add this tx to the list
    }
    fmt.Printf("- getHistoryForMarble returning:\n%s", history)

    //change to array of bytes
    historyAsBytes, _ := json.Marshal(history)     //convert to array of bytes
    return shim.Success(historyAsBytes)
}

// ============================================================================================================================
// Get history of asset - performs a range query based on the start and end keys provided.
//
// Shows Off GetStateByRange() - reading a multiple key/values from the ledger
//
// Inputs - Array of strings
//       0     ,    1
//   startKey  ,  endKey
//  "marbles1" , "marbles5"
// ============================================================================================================================
func getMarblesByRange(stub shim.ChaincodeStubInterface, args []string) pb.Response {
    if len(args) != 2 {
        return shim.Error("Incorrect number of arguments. Expecting 2")
    }

    startKey := args[0]
    endKey := args[1]

    resultsIterator, err := stub.GetStateByRange(startKey, endKey)
    if err != nil {
        return shim.Error(err.Error())
    }
    defer resultsIterator.Close()

    // buffer is a JSON array containing QueryResults
    var buffer bytes.Buffer
    buffer.WriteString("[")

    bArrayMemberAlreadyWritten := false
    for resultsIterator.HasNext() {
        aKeyValue, err := resultsIterator.Next()
        if err != nil {
            return shim.Error(err.Error())
        }
        queryResultKey := aKeyValue.Key
        queryResultValue := aKeyValue.Value

        // Add a comma before array members, suppress it for the first array member
        if bArrayMemberAlreadyWritten == true {
            buffer.WriteString(",")
        }
        buffer.WriteString("{\"Key\":")
        buffer.WriteString("\"")
        buffer.WriteString(queryResultKey)
        buffer.WriteString("\"")

        buffer.WriteString(", \"Record\":")
        // Record is a JSON object, so we write as-is
        buffer.WriteString(string(queryResultValue))
        buffer.WriteString("}")
        bArrayMemberAlreadyWritten = true
    }
    buffer.WriteString("]")

    fmt.Printf("- getMarblesByRange queryResult:\n%s\n", buffer.String())

    return shim.Success(buffer.Bytes())
} 
```

#### 创建 write_ledger.go 文件并编辑（事务操作）

```go
$ vim write_ledger.go 
```

文件完整内容如下

```go
/*
Licensed to the Apache Software Foundation (ASF) under one
or more contributor license agreements.  See the NOTICE file
distributed with this work for additional information
regarding copyright ownership.  The ASF licenses this file
to you under the Apache License, Version 2.0 (the
"License"); you may not use this file except in compliance
with the License.  You may obtain a copy of the License at

  http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing,
software distributed under the License is distributed on an
"AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
KIND, either express or implied.  See the License for the
specific language governing permissions and limitations
under the License.
*/

package main

import (
    "encoding/json"
    "fmt"
    "strconv"
    "strings"

    "github.com/hyperledger/fabric/core/chaincode/shim"
    pb "github.com/hyperledger/fabric/protos/peer"
)

// ============================================================================================================================
// write() - genric write variable into ledger
// 
// Shows Off PutState() - writting a key/value into the ledger
//
// Inputs - Array of strings
//    0   ,    1
//   key  ,  value
//  "abc" , "test"
// ============================================================================================================================
func write(stub shim.ChaincodeStubInterface, args []string) pb.Response {
    var key, value string
    var err error
    fmt.Println("starting write")

    if len(args) != 2 {
        return shim.Error("Incorrect number of arguments. Expecting 2\. key of the variable and value to set")
    }

    // input sanitation
    err = sanitize_arguments(args)
    if err != nil {
        return shim.Error(err.Error())
    }

    key = args[0]      //rename for funsies
    value = args[1]
    err = stub.PutState(key, []byte(value))  //write the variable into the ledger
    if err != nil {
        return shim.Error(err.Error())
    }

    fmt.Println("- end write")
    return shim.Success(nil)
}

// ============================================================================================================================
// delete_marble() - remove a marble from state and from marble index
// 
// Shows Off DelState() - "removing"" a key/value from the ledger
//
// Inputs - Array of strings
//      0      ,         1
//     id      ,  authed_by_company
// "m999999999", "united marbles"
// ============================================================================================================================
func delete_marble(stub shim.ChaincodeStubInterface, args []string) (pb.Response) {
    fmt.Println("starting delete_marble")

    if len(args) != 2 {
        return shim.Error("Incorrect number of arguments. Expecting 2")
    }

    // input sanitation
    err := sanitize_arguments(args)
    if err != nil {
        return shim.Error(err.Error())
    }

    id := args[0]
    authed_by_company := args[1]

    // get the marble
    marble, err := get_marble(stub, id)
    if err != nil{
        fmt.Println("Failed to find marble by id " + id)
        return shim.Error(err.Error())
    }

    // check authorizing company (see note in set_owner() about how this is quirky)
    if marble.Owner.Company != authed_by_company{
        return shim.Error("The company '" + authed_by_company + "' cannot authorize deletion for '" + marble.Owner.Company + "'.")
    }

    // remove the marble
    err = stub.DelState(id)                                                 //remove the key from chaincode state
    if err != nil {
        return shim.Error("Failed to delete state")
    }

    fmt.Println("- end delete_marble")
    return shim.Success(nil)
}

// ============================================================================================================================
// Init Marble - create a new marble, store into chaincode state
//
// Shows off building a key's JSON value manually
//
// Inputs - Array of strings
//      0      ,    1  ,  2  ,      3          ,       4
//     id      ,  color, size,     owner id    ,  authing company
// "m999999999", "blue", "35", "o9999999999999", "united marbles"
// ============================================================================================================================
func init_marble(stub shim.ChaincodeStubInterface, args []string) (pb.Response) {
    var err error
    fmt.Println("starting init_marble")

    if len(args) != 5 {
        return shim.Error("Incorrect number of arguments. Expecting 5")
    }

    //input sanitation
    err = sanitize_arguments(args)
    if err != nil {
        return shim.Error(err.Error())
    }

    id := args[0]
    color := strings.ToLower(args[1])
    owner_id := args[3]
    authed_by_company := args[4]
    size, err := strconv.Atoi(args[2])
    if err != nil {
        return shim.Error("3rd argument must be a numeric string")
    }

    //check if new owner exists
    owner, err := get_owner(stub, owner_id)
    if err != nil {
        fmt.Println("Failed to find owner - " + owner_id)
        return shim.Error(err.Error())
    }

    //check authorizing company (see note in set_owner() about how this is quirky)
    if owner.Company != authed_by_company{
        return shim.Error("The company '" + authed_by_company + "' cannot authorize creation for '" + owner.Company + "'.")
    }

    //check if marble id already exists
    marble, err := get_marble(stub, id)
    if err == nil {
        fmt.Println("This marble already exists - " + id)
        fmt.Println(marble)
        return shim.Error("This marble already exists - " + id)  //all stop a marble by this id exists
    }

    //build the marble json string manually
    str := `{
        "docType":"marble", 
        "id": "` + id + `", 
        "color": "` + color + `", 
        "size": ` + strconv.Itoa(size) + `, 
        "owner": {
            "id": "` + owner_id + `", 
            "username": "` + owner.Username + `", 
            "company": "` + owner.Company + `"
        }
    }`
    err = stub.PutState(id, []byte(str))  //store marble with id as key
    if err != nil {
        return shim.Error(err.Error())
    }

    fmt.Println("- end init_marble")
    return shim.Success(nil)
}

// ============================================================================================================================
// Init Owner - create a new owner aka end user, store into chaincode state
//
// Shows off building key's value from GoLang Structure
//
// Inputs - Array of Strings
//           0     ,     1   ,   2
//      owner id   , username, company
// "o9999999999999",     bob", "united marbles"
// ============================================================================================================================
func init_owner(stub shim.ChaincodeStubInterface, args []string) pb.Response {
    var err error
    fmt.Println("starting init_owner")

    if len(args) != 3 {
        return shim.Error("Incorrect number of arguments. Expecting 3")
    }

    //input sanitation
    err = sanitize_arguments(args)
    if err != nil {
        return shim.Error(err.Error())
    }

    var owner Owner
    owner.ObjectType = "marble_owner"
    owner.Id =  args[0]
    owner.Username = strings.ToLower(args[1])
    owner.Company = args[2]
    owner.Enabled = true
    fmt.Println(owner)

    //check if user already exists
    _, err = get_owner(stub, owner.Id)
    if err == nil {
        fmt.Println("This owner already exists - " + owner.Id)
        return shim.Error("This owner already exists - " + owner.Id)
    }

    //store user
    ownerAsBytes, _ := json.Marshal(owner) //convert to array of bytes
    err = stub.PutState(owner.Id, ownerAsBytes)  //store owner by its Id
    if err != nil {
        fmt.Println("Could not store user")
        return shim.Error(err.Error())
    }

    fmt.Println("- end init_owner marble")
    return shim.Success(nil)
}

// ============================================================================================================================
// Set Owner on Marble
//
// Shows off GetState() and PutState()
//
// Inputs - Array of Strings
//       0     ,        1      ,        2
//  marble id  ,  to owner id  , company that auth the transfer
// "m999999999", "o99999999999", united_mables" 
// ============================================================================================================================
func set_owner(stub shim.ChaincodeStubInterface, args []string) pb.Response {
    var err error
    fmt.Println("starting set_owner")

    // this is quirky
    // todo - get the "company that authed the transfer" from the certificate instead of an argument
    // should be possible since we can now add attributes to the enrollment cert
    // as is.. this is a bit broken (security wise), but it's much much easier to demo! holding off for demos sake

    if len(args) != 3 {
        return shim.Error("Incorrect number of arguments. Expecting 3")
    }

    // input sanitation
    err = sanitize_arguments(args)
    if err != nil {
        return shim.Error(err.Error())
    }

    var marble_id = args[0]
    var new_owner_id = args[1]
    var authed_by_company = args[2]
    fmt.Println(marble_id + "->" + new_owner_id + " - |" + authed_by_company)

    // check if user already exists
    owner, err := get_owner(stub, new_owner_id)
    if err != nil {
        return shim.Error("This owner does not exist - " + new_owner_id)
    }

    // get marble's current state
    marbleAsBytes, err := stub.GetState(marble_id)
    if err != nil {
        return shim.Error("Failed to get marble")
    }
    res := Marble{}
    json.Unmarshal(marbleAsBytes, &res) //un stringify it aka JSON.parse()

    // check authorizing company
    if res.Owner.Company != authed_by_company{
        return shim.Error("The company '" + authed_by_company + "' cannot authorize transfers for '" + res.Owner.Company + "'.")
    }

    // transfer the marble
    res.Owner.Id = new_owner_id    //change the owner
    res.Owner.Username = owner.Username
    res.Owner.Company = owner.Company
    jsonAsBytes, _ := json.Marshal(res)     //convert to array of bytes
    err = stub.PutState(args[0], jsonAsBytes)     //rewrite the marble with id as key
    if err != nil {
        return shim.Error(err.Error())
    }

    fmt.Println("- end set owner")
    return shim.Success(nil)
}

// ============================================================================================================================
// Disable Marble Owner
//
// Shows off PutState()
//
// Inputs - Array of Strings
//       0     ,        1      
//  owner id       , company that auth the transfer
// "o9999999999999", "united_mables"
// ============================================================================================================================
func disable_owner(stub shim.ChaincodeStubInterface, args []string) pb.Response {
    var err error
    fmt.Println("starting disable_owner")

    if len(args) != 2 {
        return shim.Error("Incorrect number of arguments. Expecting 2")
    }

    // input sanitation
    err = sanitize_arguments(args)
    if err != nil {
        return shim.Error(err.Error())
    }

    var owner_id = args[0]
    var authed_by_company = args[1]

    // get the marble owner data
    owner, err := get_owner(stub, owner_id)
    if err != nil {
        return shim.Error("This owner does not exist - " + owner_id)
    }

    // check authorizing company
    if owner.Company != authed_by_company {
        return shim.Error("The company '" + authed_by_company + "' cannot change another companies marble owner")
    }

    // disable the owner
    owner.Enabled = false
    jsonAsBytes, _ := json.Marshal(owner)  //convert to array of bytes
    err = stub.PutState(args[0], jsonAsBytes)  //rewrite the owner
    if err != nil {
        return shim.Error(err.Error())
    }

    fmt.Println("- end disable_owner")
    return shim.Success(nil)
} 
```