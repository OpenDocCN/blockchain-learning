# 5.3 链码实现的 Hello World

## 目标

1.  使用链码相关的 API 实现一个简单的 Hello World 入门应用
2.  使用开发测试模式测试 Hello World 应用

## 任务实现

> 前面我们已经接触了与链码相关的内容，下面我们根据已掌握的链码知识实现一个简单的链码应用。该应用需求较为简单：链码在实例化时向账本保存一个初始数据，key 为 Hello， value 为 World，然后用户发出查询请求，可以根据 key 查询到相应的 value。

### 5.3.1 链码开发

1.  **创建文件夹**

    进入 `fabric-samples/chaincode/` 目录下并创建一个名为 `hello` 的文件夹

    ```go
    $ cd hyfa/fabric-samples/chaincode
    $ sudo mkdir hello
    $ cd hello 
    ```

2.  **创建并编辑链码文件**

    ```go
    $ sudo vim hello.go 
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

4.  **编写主函数**

    ```go
    func main()  {
       err := shim.Start(new(HelloChaincode))
       if err != nil {
          fmt.Printf("链码启动失败: %v", err)
       }
    } 
    ```

5.  **自定义结构体**

    ```go
    type HelloChaincode struct {

    } 
    ```

6.  **实现 Chaincode 接口**

    实现 `Chaincode` 接口必须重写 **Init** 与 **Invoke** 两个方法。

    **Init** 函数：初始化数据状态

    *   获取参数并判断参数长度是否为 2
        *   参数: Key, Value
    *   调用 PutState 函数将状态写入账本中
    *   如果有错误, 则返回
    *   打印输出提示信息
    *   返回成功

    具体实现代码如下：

    ```go
    // 实例化/升级链码时被自动调用
    // -c '{"Args":["Hello","World"]'
    func (t *HelloChaincode) Init(stub shim.ChaincodeStubInterface) peer.Response  {
       fmt.Println("开始实例化链码....")

       // 获取参数
       //args := stub.GetStringArgs()
       _, args := stub.GetFunctionAndParameters()
       // 判断参数长度是否为 2 个
       if len(args) != 2 {
          return shim.Error("指定了错误的参数个数")
       }

       fmt.Println("保存数据......")

       // 通过调用 PutState 方法将数据保存在账本中
       err := stub.PutState(args[0], []byte(args[1]))
       if err != nil {
          return shim.Error("保存数据时发生错误...")
       }

       fmt.Println("实例化链码成功")

       return shim.Success(nil)

    } 
    ```

**Invoke** 函数

*   获取参数并判断长度是否为 1

*   利用第 1 个参数获取对应状态 GetState(key)

*   如果有错误则返回

*   如果返回值为空则返回错误

*   返回成功状态

    具体实现代码如下：

```go
 // 对账本数据进行操作时被自动调用(query, invoke)
 func (t *HelloChaincode)  Invoke(stub shim.ChaincodeStubInterface) peer.Response  {
     // 获取调用链码时传递的参数内容(包括要调用的函数名及参数)
     fun, args := stub.GetFunctionAndParameters()

     // 客户意图
     if fun == "query"{
         return query(stub, args)
     }

          return shim.Error("非法操作, 指定功能不能实现")
 } 
```

1.  **实现查询函数**

    函数名称为 query，具体实现如下：

    ```go
    func query(stub shim.ChaincodeStubInterface, args []string) peer.Response {
       // 检查传递的参数个数是否为 1
       if len(args) != 1{
          return shim.Error("指定的参数错误，必须且只能指定相应的 Key")
       }

       // 根据指定的 Key 调用 GetState 方法查询数据
       result, err := stub.GetState(args[0])
       if err != nil {
          return shim.Error("根据指定的 " + args[0] + " 查询数据时发生错误")
       }
       if result == nil {
          return shim.Error("根据指定的 " + args[0] + " 没有查询到相应的数据")
       }

       // 返回查询结果
       return shim.Success(result)
    } 
    ```

### 5.3.2 链码测试

1.  **启动网络**

    进入 `fabric-samples/chaincode-docker-devmode/` 目录

    ```go
    $ cd ../chaincode-docker-devmode/ 
    ```

2.  **构建并启动链码**

    **2.1 打开一个新的终端 2，进入 chaincode 容器：**

    ```go
    $ sudo docker exec -it chaincode bash 
    ```

    **2.2 编译链码**

    ```go
    # cd hello
    # go build 
    ```

    **2.3 启动链码**

    ```go
    # CORE_PEER_ADDRESS=peer:7052 CORE_CHAINCODE_ID_NAME=hellocc:0 ./hello 
    ```

    命令执行后输出如下：

    ```go
    [shim] SetupChaincodeLogging -> INFO 001 Chaincode log level not provided; defaulting to: INFO
    [shim] SetupChaincodeLogging -> INFO 002 Chaincode (build level: ) starting up ... 
    ```

3.  **测试：**

    **3.1 打开一个新的终端 3，进入 cli 容器**

    ```go
    $ sudo docker exec -it cli bash 
    ```

    **3.2 安装链码**

    ```go
    # peer chaincode install -p chaincodedev/chaincode/hello -n hellocc -v 0 
    ```

    **3.3 实例化链码**

    ```go
    # peer chaincode instantiate -n hellocc -v 0 -c '{"Args":["init", "Hello","World"]}' -C myc 
    ```

    **3.4 调用链码**

    根据指定的 key （"Hello"）查询对应的状态数据

    ```go
    # peer chaincode query -n hellocc  -c '{"Args":["query","Hello"]}' -C myc 
    ```

    返回查询结果： `World`

## FAQ

1.  在调用链码时将 query 换为 invoke 可以吗？

    可以将 query 替换为 invoke 操作，但是两个命令的执行流程也不同，而且执行后可以从终端的输出中看出，返回的查询结果显示的内容是一串数字，无法确定其正确性。