# 十一、.2 EOS 智能合约的标准化开发流程

在前一个章节中，我们介绍了如何在本地快速搭建一个 EOS 项目工程以及如何快速使用 vscode 构建一个 EOS 本地链环境。

那么本章节，将会介绍如何在此项目工程结构中新建一个业务合约。在合约开发过程， 如何进行合约设计、代码规范以及在开发中需要规避的安全问题。关于安全问题，我们会在后续的一个章节中集中讲解。

除此之外，还会教大家如何利用现在文档资料。比如：如何选择合适的数据类型，合约中系统方法如何使用， C+ +资料查询等。

* * *

接下来，我们将一步步带大家创建自己的智能合约

**首先**， 在工程目录下新建两个文件`dexchange.hpp`和`dexcange.cpp`.；

```js
.
├── CMakeLists.txt
├── README.md
├── contracts
│   ├── dexchange
│   │   ├── dexchange.cpp   //头文件定义
│   │   └── dexchange.hpp   //源文件定义
├── docker
├── migrations
├── package.json
└── test

8 directories, 21 files 
```

**dexchange.hpp**：主要用于定义合约主体。并在合约中，定义所有交易所业务方法及方法内部需要使用的业务模型。

# 

**dexcange.cpp**：主要针对**dexchange.hpp**定义的接口进行业务扩展实现。

**然后**， 在`dexchange.hpp`文件中定义合约主体内容，分为编写六小段内容

1.  命令空间，主要用于处理相同类或数据类型冲突的问题。比如：在 A 包中有一个数据类型 person；在 B 包下也同样有一个数据类型 person；在这种情况下我们想要使用不同的 person 数据类型，只能通过添加命名空间来进行区分；

    ```js
    namespace chaindesk {
        ... ...
    }
    ```

2.  合约类定义

    ```js
    class [[eosio::contract("dexchange")]] dexchange : public eosio::contract{
        ... ...
    }
    ```

3.  构造函数定义

    ```js
    dexchange( name receiver, name code, datastream<const char*>    ds ):contract( receiver, code, ds ) {}
    ```

    构造函数必须定义，该函数主要用于初始化合约系统成员变量 receiver、code、ds, 以供合约实现中通过 _self, _code 调用

4.  类函数定义

    ```js
    [[eosio::action]]
    void init();
    ```

    `eosio::action`标识当前方法可对外访问。

5.  数据结构体定义

    ```js
    struct balance {
        asset total_asset; 
        asset frozen_asset;

        uint64_t primary_key()const { 
            return total_asset.symbol.raw(); 
        }

        //序列化结构体
        EOSLIB_SERIALIZE(balance, (total_asset)(frozen_asset));
    };
    ```

6.  访问索引器

    ```js
    typedef eosio::multi_index<"balance"_n, balance> BalanceIndex;
    ```

    该索引器为访问业务数据的入口，支持按主键及自定义索引进行数据查询。

    另外，在定义过程中，可根据实际业务需求自由设定成员变量与成员函数的访问级别，是对外可见(public)还是对外不可见(private)。

**其次**， 在`dexcange.cpp`文件具体实现在头文件`dexchange.hpp`中定义的函数方法。 在合约方法实现文件，主要编写以下三部分内容：

1.  命令空间
    在多个命令空间的情况下，用于区分相同方法。
2.  实现具体方法
3.  apply 宏定义
    用于控制智能合约方法调用前的逻辑判断以及路由转发。比如：充值 EOS 币的时候，可以判断是否是从 eosio.token 发送过来的消息通知。
    除此之外，如果在此宏定义中未配置合约方法，外部是无法进行调用的。

```js
namespace chaindesk {

    void dexchange::init(){

    }

    extern "C" void apply(uint64_t receiver, uint64_t code, uint64_t action) { 

        if ((code == "eosio.token"_n.value) && (action == "transfer"_n.value)) {
            transfer data = eosio::unpack_action_data<transfer>();
            if(data.to.value == receiver &&
                (
                code == "eosio.token"_n.value && data.quantity.symbol == symbol("EOS",4))
                ){
                eosio::execute_action(eosio::name(receiver), eosio::name(code), &dexchange::deposit);
            }
            return;
        }

        if (code != receiver)
            return;

        switch (action) {
            EOSIO_DISPATCH_HELPER(dexchange, (init));
        };
        eosio_exit(0);
    }
}
```

**下一步**，尝试编译与发布合约 可通过上一章节工程配置介绍发布合约的方式，进行合约编译及发布；

**最后**，合约测试与调用。 关于合约的测试，有两种方式：

*   使用 cleos push action 的方式直接调用合约方法测试
*   使用 jest 插件编译 js 脚本进行测试 鉴于目前只针对合约本身进行开发，暂时先使用 cleos 命令的方式进行测试。

```js
# 1\. 解锁钱包
cleos wallet unlock -n <钱包名称> --password <钱包密码>;
# 2\. 调用方法
cleos push action <合约名> <合约方法> [<参数 1>，<参数 2>] -p <合约名>@active
```

## 知识点

1.  头文件(.hpp): 用于声明类定义，其中包括成员变量、数据结构体、成员函数的定义，但对于成员函数在此文件中并不进行具体实现。
2.  源文件(.cpp): 主要用于对头文件中已声明的成员函数进行具体实现。
3.  .hpp 和.cpp 文件实际上是可以统一整合成.cpp 文件中的。但为了遵循规范，以及在与第三方进行系统集成时不暴露自己的核心实现而又能让对方调用，所以需要采用封装隔离的方式。

## 参考资料

1.  在线编辑器： [`www.tutorialspoint.com/compile_cpp_online.php`](https://www.tutorialspoint.com/compile_cpp_online.php)
2.  C++参考手册：[`www.cplusplus.com/`](http://www.cplusplus.com/)
3.  The cplusplus.com tutorial： [`people.scs.carleton.ca/~dehne/projects/cpp-doc/tutorial/index.html`](http://people.scs.carleton.ca/~dehne/projects/cpp-doc/tutorial/index.html)