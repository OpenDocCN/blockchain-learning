# 5.4 动手编码一：链码实现资产管理

## 目标

1.  简单的分析链码的设计与开发
2.  使用链码相关的 API 实现一个简单的资产管理应用
3.  使用开发测试模式测试简单的资产链码应用

## 任务实现

> 下面我们来实现一个简单的资产链码应用，该链码能够让用户在分类账上创建资产，并通过指定的函数实现对资产的修改与查询功能。

### 5.4.1 资产链码开发

1.  **创建目录**

    为 chaincode 应用创建一个名为 test 的目录

    ```go
    $ cd ~/hyfa/fabric-samples/chaincode
    $ sudo mkdir test 
    $ cd test 
    ```

2.  **新建并编辑链码文件**

    新建一个文件 test.go ，用于编写 Go 代码

    ```go
    $ sudo vim test.go 
    ```

3.  **导入链码依赖包**

    ```go
    // hanxiaodong
    // QQ 群（专业 Fabric 交流群）：862733552
    package main

    import (
       "github.com/hyperledger/fabric/core/chaincode/shim"
       "github.com/hyperledger/fabric/protos/peer"
       "fmt"
    ) 
    ```

1.  **定义结构体**

    ```go
    type SimpleChaincode struct {
    } 
    ```

1.  **编写主函数**

    ```go
    func main(){
        err := shim.Start(new(SimpleChaincode))
        if err != nil{
            fmt.Printf("启动 SimpleChaincode 时发生错误: %s", err)
        }
    } 
    ```

1.  **实现 Chaincode 接口**

    **Init** 函数：初始化数据状态

    *   获取参数, 使用 `GetStringArgs` 函数传递给调用链码的所需参数
    *   检查合法性, 检查参数数量是否为 2 个, 如果不是, 则返回错误信息
    *   利用两个参数, 调用 PutState 方法向账本中写入状态, 如果有错误则返回 shim.Error()， 否则返回 nil（shim.Success）

    具体实现代码如下：

    ```go
    func (t *SimpleChaincode) Init(stub shim.ChaincodeStubInterface) peer.Response{
        args := stub.GetStringArgs()
        if len(args) != 2{
            return shim.Error("初始化的参数只能为 2 个， 分别代表名称与状态数据")
        }
        err := stub.PutState(args[0], []byte(args[1]))
        if err != nil{
            return shim.Error("在保存状态时出现错误")
        }
        return shim.Success(nil)
    } 
    ```

**Invok**函数：验证函数名称为 set 或 get，并调用那些链式代码应用程序函数，通过 shim.Success 或 shim.Error 函数返回响应。

*   获取函数名与参数
*   对获取到的参数名称进行判断, 如果为 set, 则调用 set 方法, 反之调用 get
    *   set/get 函数返回两个值（result, err）
*   如果 err 不为空则返回错误
*   err 为空则返回 []byte（result）

    具体实现代码如下：

    ```go
    func (t * SimpleChaincode) Invoke(stub shim.ChaincodeStubInterface) peer.Response{
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
    ```

1.  **实现具体业务功能的函数**

    应用程序实现了两个可以通过 `Invoke` 函数调用的函数 （set/get）

    为了访问分类账的状态，利用 chaincode shim API 的 `ChaincodeStubInterface.PutState` 和`ChaincodeStubInterface.GetState` 函数

    **7.1 实现 set 函数：修改资产**

    *   检查参数个数是否为 2
    *   利用 PutState 方法将状态写入
    *   如果成功,则返回要写入的状态, 失败返回错误: fmt.Errorf("...")

    具体实现代码如下：

```go
 func set(stub shim.ChaincodeStubInterface, args []string)(string, error){

       if len(args) != 2{
           return "", fmt.Errorf("给定的参数个数不符合要求")
       }

       err := stub.PutState(args[0], []byte(args[1]))
       if err != nil{
           return "", fmt.Errorf(err.Error())
       }
       return string(args[0]), nil

   } 
```

**7.2 实现 get 函数：查询资产**

*   接收参数并判断个数 是否为 1 个
*   调用 GetState 方法返回并接收两个返回值（value, err）判断 err 及 value 是否为空 return ""， fmt.Errorf("......")
*   返回值 return string(value)，nil

    具体实现代码如下：

```go
func get(stub shim.ChaincodeStubInterface, args []string)(string, error){
    if len(args) != 1{
        return "", fmt.Errorf("给定的参数个数不符合要求")
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
```

### 5.4.2 链码测试

跳转至 `fabric-samples` 的 `chaincode-docker-devmode` 目录

```go
$ cd ~/hyfa/fabric-samples/chaincode-docker-devmode/ 
```

1.  **终端 1 启动网络**

    ```go
    $ sudo docker-compose -f docker-compose-simple.yaml up -d 
    ```

    > 在执行启动网络的命令之前确保无 Fabric 网络处于运行状态，如果有网络在运行，请先关闭。

2.  **终端 2 建立并启动链码**

    **2.1 打开一个新终端 2，进入 chaincode 容器**

    ```go
    $ sudo docker exec -it chaincode bash 
    ```

    **2.2 编译**

    进入 test 目录编译 chaincode

    ```go
    # cd test
    # go build 
    ```

    **2.3 运行 chaincode**

    ```go
    # CORE_PEER_ADDRESS=peer:7052 CORE_CHAINCODE_ID_NAME=test:0 ./test 
    ```

    命令执行后输出如下:

    ```go
    [shim] SetupChaincodeLogging -> INFO 001 Chaincode log level not provided; defaulting to: INFO
    [shim] SetupChaincodeLogging -> INFO 002 Chaincode (build level: ) starting up ... 
    ```

3.  **终端 3 测试**

    **3.1 打开一个新的终端 3，进入 cli 容器**

    ```go
    $ sudo docker exec -it cli bash 
    ```

    **3.2 安装链码**

    ```go
    # peer chaincode install -p chaincodedev/chaincode/test -n test -v 0 
    ```

    **3.3 实例化链码**

    ```go
    # peer chaincode instantiate -n test -v 0 -c '{"Args":["a","10"]}' -C myc 
    ```

    **3.4 调用链码**

    指定调用 set 函数，将`a`的值更改为`20`

    ```go
    # peer chaincode invoke -n test -c '{"Args":["set", "a", "20"]}' -C myc 
    ```

    执行成功，输出如下内容：

    ```go
    ......
    [chaincodeCmd] chaincodeInvokeOrQuery -> INFO 0a8 Chaincode invoke successful. result: status:200 payload:"a" 
    ```

    **3.5 查询**

    指定调用 get 函数，查询 `a` 的值

    ```go
    # peer chaincode query -n test -c '{"Args":["query","a"]}' -C myc 
    ```

    执行成功, 输出: `20`