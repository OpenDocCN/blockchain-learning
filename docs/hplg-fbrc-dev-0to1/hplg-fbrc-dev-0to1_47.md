# 11.3 链码实现

## 目标

1.  实现链代码

## 任务实现

为了便于测试及简化代码，我们实现一个简单的链码功能，能够实现对分类账本中的数据进行设置（PutState（k，v））及相应的查询（GetState（k））功能即可。

### 11.3.1 编写链码

在当前项目根目录中创建一个存放链码文件的 `chaincode` 目录，然后在该目录下创建一个 `main.go` 的文件并对其进行编辑

```go
$ mkdir chaincode
$ vim chaincode/main.go 
```

编写链码必须遵守链码开发的相关规定（详见第五章链码开发的相关内容），为此我们在链码文件中声明 5 个函数，如下：

*   **Init(stub shim.ChaincodeStubInterface) ：**初始化链码或升级链码时被调用的函数
*   **Invoke(stub shim.ChaincodeStubInterface)：**查询或更新操作分类账本状态时被调用的函数
*   **set(stub shim.ChaincodeStubInterface, args []string)：**根据用户指定的 Key 与 Value 更新分类账本中的状态
*   **get(stub shim.ChaincodeStubInterface, args []string)：**根据用户指定的 Key 从分类账本中查询状态
*   **main()：**启动链码的主函数

`main.go` 文件内容如下：

```go
/**
  author: hanxiaodong
  QQ 群（专业 Fabric 交流群）：862733552
 */
package main

import (
    "github.com/hyperledger/fabric/core/chaincode/shim"
    "github.com/hyperledger/fabric/protos/peer"
    "fmt"
)

type SimpleChaincode struct {

} 

func (t *SimpleChaincode) Init(stub shim.ChaincodeStubInterface) peer.Response{

    return shim.Success(nil)
}

func (t *SimpleChaincode) Invoke(stub shim.ChaincodeStubInterface) peer.Response{
    fun, args := stub.GetFunctionAndParameters()

    var result string
    var err error
    if fun == "set"{
        result, err = set(stub, args)
    }else{
        result, err = get(stub, args)
    }
    if err != nil{
        return shim.Error(err.Error())
    }
    return shim.Success([]byte(result))
}

func set(stub shim.ChaincodeStubInterface, args []string)(string, error){

    if len(args) != 3{
        return "", fmt.Errorf("给定的参数错误")
    }

    err := stub.PutState(args[0], []byte(args[1]))
    if err != nil{
        return "", fmt.Errorf(err.Error())
    }

    err = stub.SetEvent(args[2], []byte{})
    if err != nil {
        return "", fmt.Errorf(err.Error())
    }

    return string(args[0]), nil

}

func get(stub shim.ChaincodeStubInterface, args []string)(string, error){
    if len(args) != 1{
        return "", fmt.Errorf("给定的参数错误")
    }
    result, err := stub.GetState(args[0])
    if err != nil{
        return "", fmt.Errorf("获取数据发生错误")
    }
    if result == nil{
        return "", fmt.Errorf("根据 %s 没有获取到相应的数据", args[0])
    }
    return string(result), nil

}

func main(){
    err := shim.Start(new(SimpleChaincode))
    if err != nil{
        fmt.Printf("启动 SimpleChaincode 时发生错误: %s", err)
    }
} 
```

链码编写好以后，我们需要使用 Fabric-SDK-Go 提供的相关 API 来实现对链码的安装及实例化操作，而无需在命令提示符中输入烦锁的相关操作命令。