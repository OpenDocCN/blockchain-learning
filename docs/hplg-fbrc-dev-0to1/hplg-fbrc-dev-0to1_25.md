# 五、.5 动手编码二：链码实现转账

## 目标

1.  简单的分析链码的设计与开发
2.  使用链码相关的 API 实现一个简单的资产管理应用
3.  使用开发测试模式测试简单的资产链码应用

## 任务实现

> 下面我们来实现一个使用链码能够实现对账户的查询，转账，删除账户的功能，并且整合完善资产管理应用链码的功能，该链码能够让用户在分类账上创建资产，并通过指定的函数实现对资产的修改与查询。

### 5.5.1 转账链码开发

1.  **创建目录**

    为 chaincode 应用创建一个名为 payment 的目录

    ```go
    $ cd ~/hyfa/fabric-samples/chaincode
    $ sudo mkdir payment 
    $ cd payment 
    ```

2.  **新建并编辑链码文件**

    新建一个文件 payment.go ，用于编写 Go 代码

    ```go
    $ sudo vim payment.go 
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
       "strconv"
    ) 
    ```

1.  **定义结构体**

    ```go
    type PaymentChaincode struct {

    } 
    ```

1.  **编写主函数**

    ```go
    func main(){
        err := shim.Start(new(PaymentChaincode))
        if err != nil{
            fmt.Printf("启动 PaymentChaincode 时发生错误: %s", err)
        }
    } 
    ```

1.  **实现 Chaincode 接口**

    **Init** 函数：初始化两个账户，账户名分别为 a、b，对应的金额为 100、200

    *   判断参数个数是否为 4
    *   获取 args[0] 的值赋给 A
    *   strconv.Atoi（args[1]） 转换为整数, 返回 aval, err
    *   判断 err
    *   获取 args[2] 的值赋给 B
    *   strconv.Atoi（args[3]） 转换为整数, 返回 bval, err
    *   判断 err
    *   将 A 的状态值记录到分布式账本中
    *   判断 err
    *   将 B 的状态值记录到分布式账本中
    *   判断 err
    *   return shim.Success（nil）

    具体实现代码如下：

    ```go
    // 初始化两个账户及相应的余额
    // -c '{"Args":["init", "第一个账户名称", "第一个账户初始余额", "第二个账户名称", "第二个账户初始余额"]}'
    func (t *PaymentChaincode) Init(stub shim.ChaincodeStubInterface) peer.Response {

        // 获取参数并验证
        _, args := stub.GetFunctionAndParameters()
        if len(args) != 4 {
            return shim.Error("必须指定两个账户名称及相应的初始余额")
        }

        // 判断账户名称是否合法
        var a = args[0]
        var avalStr = args[1]
        var b = args[2]
        var bvalStr = args[3]

        if len(a) < 2 {
            return shim.Error(a + " 账户名称不能少于 2 个字符长度")
        }
        if len(b) < 2 {
            return shim.Error(b + " 账户名称不能少于 2 个字符长度")
        }

        _, err := strconv.Atoi(avalStr)
        if err != nil {
            return shim.Error("指定的账户初始余额错误: " + avalStr)
        }
        _, err = strconv.Atoi(bvalStr)
        if err != nil {
            return shim.Error("指定的账户初始余额错误: " + bvalStr)
        }

        // 保存两个账户状态至账本中
        err = stub.PutState(a, []byte(avalStr))
        if err != nil {
            return shim.Error(a + " 保存状态时发生错误")
        }
        err = stub.PutState(b, []byte(bvalStr))
        if err != nil {
            return shim.Error(b + " 保存状态时发生错误")
        }

        return shim.Success([]byte("初始化成功"))

    } 
    ```

**Invoke** 函数：应用程序将具有三个不同的分支功能：`find` 、`payment` 、`delete`分别实现转账、删除、查询的功能, 根据交易参数定位到不同的分支处理逻辑。

*   获取函数名称与参数列表
*   判断函数名称并调用相应的函数

    具体实现代码如下：

    ```go
    // peer chaincode query -n pay -C myc -c '{"Args":["find", "a"]}'
    func (t *PaymentChaincode) Invoke(stub shim.ChaincodeStubInterface) peer.Response {
      // 获取用户意图
      fun, args := stub.GetFunctionAndParameters()

      if fun == "find" {
          return find(stub, args)
      }else if fun == "payment" {
          return payment(stub, args)
      }else if fun == "del" {
          return delAccount(stub, args)
      }else if fun == "set" {
          return t.set(stub, args)
      }else if fun == "get" {
          return t.get(stub, args)
      }

      return shim.Error("非法操作, 指定的功能不能实现")
    } 
    ```

1.  **实现具体业务功能的函数**

    应用程序实现了三个可以通过 `Invoke` 函数调用的函数（finde、payment、delAccount）

    **7.1 实现 find 函数：根据给定的账户名称查询对应的状态信息**

    *   判断参数是否为 1 个
    *   根据传入的参数调用 GetState 查询状态， aval， err 为接收返回值
    *   如果返回 err 不为空，则返回错误
    *   如果返回的状态为空，则返回错误
    *   如果无错误，返回查询到的值

    具体实现代码如下：

    ```go
    // 根据指定的账户名称查询对应的余额信息
    // -c '{"Args":["find", "账户名称"]}'
    func find(stub shim.ChaincodeStubInterface, args []string) peer.Response {
        if len(args) != 1 {
            return shim.Error("必须且只能指定要查询的账户名称")
        }

        result, err := stub.GetState(args[0])
        if err != nil {
            return shim.Error("查询 " + args[0] + " 账户信息失败" + err.Error())
        }

        if result == nil {
            return shim.Error("根据指定 " + args[0] + " 没有查询到对应的余额")
        }

        return shim.Success(result)

    } 
    ```

**7.2 实现 payment 函数：根据指定的两个账户名称及金额，实现转账**

*   判断参数是否为 3
*   获取两个账户名称（args[0] 与 args[1]）值, 赋给两个变量
*   调用 GetState 获取 a 账户状态，avalsByte， err 为返回值
    *   判断有无错误（err 不为空， avalsByte 为空）
*   类型转换: aval， _ = strconv.Atoi（string(avalsByte)）
*   调用 GetState 获取 b 账户状态， bvalsByte，err 为返回值
    *   判断有无错误（err 不为空，bvalsByte 为空）
*   类型转换: bval， _ = strconv.Atoi（string(bvalsByte)）
*   将要转账的数额进行类型转换： x， err = strconv.Atoi（args[2]）
*   判断 err 是否为空
*   aval， bval 执行转账操作
*   记录状态， err = PutState(a, []byte（strconv.Itoa(aval))）
    *   Itoa： 将整数转换为十进制字符串形式
    *   判断有无错误.
*   记录状态， err = PutState（b, []byte(strconv.Itoa(bval))）
    *   判断有无错误.
*   return shim.Success（nil）

    具体实现代码如下：

```go
// 转账
// -c '{"Args":["payment", "源账户名称", "目标账户名称", "转账金额"]}'
func payment(stub shim.ChaincodeStubInterface, args []string) peer.Response {
       if len(args) != 3 {
           return shim.Error("必须且只能指定源账户及目标账户名称与对应的转账金额")
       }

       var source, target string
       var x string

       source = args[0]
       target = args[1]
       x = args[2]

       // 源账户扣除对应的转账金额
       // 目标账户加上对应的转账金额

       // 查询源账户及目标账户的余额
       sval, err := stub.GetState(source)
       if err != nil {
           return shim.Error("查询源账户信息失败")
       }
       // 如果源账户或目标账户不存在的情况下
       // 不存在的情况下直接 return

       tval, err := stub.GetState(target)
       if err != nil {
           return shim.Error("查询目标账户信息失败")
       }

       // 实现转账
       s, err := strconv.Atoi(x)
       if err != nil {
           return shim.Error("指定的转账金额错误")
       }

       svi, err := strconv.Atoi(string(sval))
       if err != nil {
           return shim.Error("处理源账户余额时发生错误")
       }

       tvi, err := strconv.Atoi(string(tval))
       if err != nil {
           return shim.Error("处理目标账户余额时发生错误")
       }

       if svi < s {
           return shim.Error("指定的源账户余额不足, 无法实现转账")
       }

       svi = svi - s
       tvi = tvi + s

       // 将修改之后的源账户与目标账户的状态保存至账本中
       err = stub.PutState(source, []byte(strconv.Itoa(svi)))
       if err != nil {
           return  shim.Error("保存转账后的源账户状态失败")
       }

       err = stub.PutState(target, []byte(strconv.Itoa(tvi)))
       if err != nil {
           return  shim.Error("保存转账后的目标账户状态失败")
       }

       return shim.Success([]byte("转账成功"))

} 
```

**7.3 实现 delAccount 函数：根据指定的名称删除对应的实体信息**

*   判断参数个数是否为 1
*   调用 DelState 方法，err 接收返回值
*   如果 err 不为空, 返回错误
*   返回成功 shim.Success(nil)

    具体实现代码如下：

    ```go
    // 根据指定的账户名称删除相应信息
    // -c '{"Args":["del", "账户名称"]}'
    func delAccount(stub shim.ChaincodeStubInterface, args []string) peer.Response {
      if len(args) != 1 {
          return shim.Error("必须且只能指定要删除的账户名称")
      }

      result, err := stub.GetState(args[0])
      if err != nil {
          return shim.Error("查询 " + args[0] + " 账户信息失败" + err.Error())
      }

      if result == nil {
          return shim.Error("根据指定 " + args[0] + " 没有查询到对应的余额")
      }

      err = stub.DelState(args[0])
      if err != nil {
          return shim.Error("删除指定的账户失败: " + args[0] + ", " + err.Error())
      }

      return shim.Success([]byte("删除指定的账户成功" + args[0]))
    } 
    ```

**7.4 实现 set 函数，设置指定账户的值**

在简单资产管理链码的的 set 函数的功能并不完善，因为我们没有考虑用户存入资产之后需要对该账户的资产进行修改，现在我们来添加这一功能。

具体实现代码如下：

```go
// 向指定的账户存入对应的金额
// -c '{"Args":["set", "账户名称", "要存入的金额"]}'
func (t *PaymentChaincode) set(stub shim.ChaincodeStubInterface, args []string) peer.Response {
       if len(args) != 2 {
           return shim.Error("必须且只能指定账户名称及要存入的金额")
       }

       result, err := stub.GetState(args[0])
       if err != nil {
           return shim.Error("根据指定的账户查询信息失败")
       }

       if result == nil {
           return shim.Error("指定的账户不存在")
       }

       // 存入账户
       val, err := strconv.Atoi(string(result))
       if err != nil {
           return shim.Error("处理指定的账户金额时发生错误")
       }
       x, err := strconv.Atoi(args[1])
       if err != nil {
           return shim.Error("指定要存入的金额错误")
       }

       val = val + x

       // 保存信息
       err = stub.PutState(args[0], []byte(strconv.Itoa(val)))
       if err != nil {
           return shim.Error("存入账户金额时发生错误")
       }
       return shim.Success([]byte("存入操作成功"))

} 
```

**7.5 实现 get 函数，从指定的账户中提取指定的金额**

同理，用户从账户中提取从指定金额的资产之后，也需要对该账户的资产进行修改。

具体实现代码如下：

```go
// 从账户中提取指定的金额
// -c '{"Args":["get", "账户名称", "要提取的金额"]}'
func (t *PaymentChaincode) get(stub shim.ChaincodeStubInterface, args []string) peer.Response  {
       if len(args) != 2 {
           return shim.Error("必须且只能指定要提取的账户名称及金额")
       }

       x, err := strconv.Atoi(args[1])
       if err != nil {
           return shim.Error("指定要提取的金额错误, 请重新输入")
       }

       // 从指定的账户中查询出现有金额
       result, err := stub.GetState(args[0])
       if err != nil {
           return shim.Error("查询指定账户金额时发生错误")
       }
       if result == nil {
           return shim.Error("要查询的账户不存在或已被注销")
       }

       val, err := strconv.Atoi(string(result))
       if err != nil {
           return shim.Error("处理账户金额时发生错误")
       }

       if val < x {
           return shim.Error("要提取的金额不足")
       }

       val = val - x
       err = stub.PutState(args[0], []byte(strconv.Itoa(val)))
       if err != nil {
           return shim.Error("提取失败, 保存数据时发生错误")
       }
       return shim.Success([]byte("提取成功"))

} 
```

### 5.5.2 链码测试

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
    # cd payment
    # go build 
    ```

    **2.3 运行 chaincode**

    ```go
    # CORE_PEER_ADDRESS=peer:7052 CORE_CHAINCODE_ID_NAME=paycc:0 ./payment 
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
    # peer chaincode install -p chaincodedev/chaincode/payment -n paycc -v 0 
    ```

    **3.3 实例化链码**

    ```go
    # peer chaincode instantiate -n paycc -v 0 -c '{"Args":["init","aaa", "100", "bbb","200"]}' -C myc 
    ```

    **3.4 调用链码**

    指定调用 payment 函数，从 `aaa` 账户向 `bbb` 账户转账 `20`

    ```go
    # peer chaincode invoke -n paycc -c '{"Args":["payment", "aaa","bbb","20"]}' -C myc 
    ```

    执行成功，输出如下内容：

    ```go
    ......
    [chaincodeCmd] chaincodeInvokeOrQuery -> INFO 0a8 Chaincode invoke successful. result: status:200 payload:"\350\275\254\350\264\246\346\210\220\345\212\237" 
    ```

    **3.5 查询**

    指定调用 find 函数，查询 `a` 账户的值

    ```go
    # peer chaincode query -n paycc -c '{"Args":["find","aaa"]}' -C myc 
    ```

    执行成功, 输出: `80`