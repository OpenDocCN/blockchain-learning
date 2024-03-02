# 十一、.11 快速搞懂 EOS 合约的增删改查以及高级技巧

相信很多开发朋友在学习一项新技术的时候，往往考虑的第一件事可能就是如何保存我们的业务数据。不管是 JavaWeb， 还是 GO，或者 Node 项目，都希望能够搞明白它的数据存储技术。

那么，在智能合约中是如何进行对数据的增删改查功能的呢？是相对复杂还是简单的呢？

本小节将带你从一个简单示例入门，逐一了解在整个操作开发过程所要完成的各个技术点。让你从一个基础的 demo 示例了解，到理解整个完整开发流程，最后掌握合约存储高级应用 API 及技巧。

* * *

**file: userlist.hpp**

```js
#include <eosiolib/eosio.hpp>
#include <eosiolib/print.hpp>

using namespace eosio;

CONTRACT userlist : public contract {
  public:
    using contract::contract;
    userlist(eosio::name receiver, eosio::name code, datastream<const char*> ds):contract(receiver, code, ds) {}

    [[eosio::action]]
    void add(std::string username, uint64_t age);

    [[eosio::action]]
    void modify(uint64_t id, std::string username, uint64_t age);

    [[eosio::action]]
    void del(uint64_t id);

  private:

    struct  [[eosio::table]] tbl_user {
      uint64_t id;
      std::string username;
      uint64_t age;

      uint64_t primary_key() const { return id; }

      uint64_t byage() const { return age; }

    };
    typedef eosio::multi_index<"tbluser"_n, tbl_user,
        indexed_by< "byage"_n, const_mem_fun<tbl_user, uint64_t, &tbl_user::byage>>
    > user_index;
};

EOSIO_DISPATCH(userlist, (add)(modify)(del)) 
```

**filename: userlist.cpp**

```js
#include "userlist.hpp"

void userlist::add(std::string username, uint64_t age) {
  userlist::user_index user_stable(_self, _code.value);

  user_stable.emplace(_self, & {
    s.id = user_stable.available_primary_key();
    s.username = username;
    s.age = age;
  });
}

void userlist::modify(uint64_t id, std::string username, uint64_t age) {
  userlist::user_index user_stable(_self, _code.value);
  auto item =  user_stable.find(id);

  eosio_assert(item != user_stable.end(), "the data isn't exist.");

  user_stable.modify(item, _self, &{
    s.username = username;
    s.age = age;
  });
}

void userlist::del(uint64_t id) {
  userlist::user_index user_stable(_self, _code.value);
  auto item =  user_stable.find(id);

  eosio_assert(item != user_stable.end(), "the data isn't exist.");

  user_stable.erase(item);
} 
```

以上示例完整展示一个具有增、删、改、查操作的用户管理合约示例。而完成这样一个功能，至少需要完成两部分代码定义：

*   表及结构体定义
    结构体是业务数据的存储载体；而表是业务数据所要持续化的数据库
*   合约方法定义
    合约方法为操作合约数据的唯一入口。而操作合约数据的方式也是需要通过一个名为`multi_index`的索引器来完成实现。

接下来，我们将分解介绍每个实现步骤以及中间的注意事项：

## 第一步、定义结构体， 创建用户对象结构

```js
struct [[eosio::table]] tbl_user {
    uint64_t id;
    std::string username;
    uint64_t age;

    uint64_t primary_key() const { return id; }
};
```

定义结构体时，需要根据实际业务模型字段大小选择相应的数据类型，因为在智能合约中存储任何数据都需要消耗资源。

`[[eosio::table]]`注解用于标识在智能合约存储系统中创建与结构体一致的表。

## 第二步、定义主键字段

一定要实现`primary_key()`缺省方法，用于告诉索引器主键字段是哪个字段，而且注意该方法为常量方法。

另外，可能我们有需求需要根据年龄进行数据查询，这样的化，也必须针对年龄字段的实现相应的查询方法。比如：

```js
uint64_t getbyage() const { return age; }
```

## 第三步、定义合约方法

从开头示例中，可以看出对数据的查询及操作，都必须用到`multi_index`类型定义。那么它到底是什么呢？

`multi_index`为 EOS 智能合约的数据库访问接口 API，它提供了对数据的添加、修改、删除、主键查询、多索引查询以及多种循环方式。后续我们将会有一个章节，集中介绍介绍它的多种查询及循环方式。

### 1）添加

```js
void userlist::add(std::string username, uint64_t age) {
  userlist::user_index user_stable(_self, _code.value);

  user_stable.emplace(_self, & {
    s.id = user_stable.available_primary_key();
    s.username = username;
    s.age = age;
  });
}
```

上述代码展示了如何保存一条用户数据到智能合约中。首先，我们需要根据事先定义的索引`userlist::user_index`进行实例化，实例化时需要传入两个参数。 第一个参数即表的归属合约帐户，即这张表属于哪个合约。如果查询的是自己合约的表，就传入自己合约的合约帐户名；如果查询的是其他合约中的表，则需要传入对方合约的合约帐户名；第二参数为数据的存储维度，可以理解为分库或分表逻辑。例如：此参数传用的是用户名称，则代表按用户维度隔离彼此的数据。

`user_stable.emplace(_self, &`该行代码为保存数据的索引方法，第一个参数代表谁将为此次数据存储支付相应的资源消耗。如果为`_self`则代表由合约本身支付，如果为用户自己则代表用户需要为此支付资源消耗。

`user_stable.available_primary_key`该行代码展示了如何获取主键字段自增 id 值。

### 2）修改

```js
void userlist::modify(uint64_t id, std::string username, uint64_t age) {
  userlist::user_index user_stable(_self, _code.value);
  auto item =  user_stable.find(id);

  eosio_assert(item != user_stable.end(), "the data isn't exist.");

  user_stable.modify(item, _self, &{
    s.username = username;
    s.age = age;
  });
} 
```

修改方法与添加方法的不同之处在于：需要使用（查询 find、修改 modify）两个方法才能完成数据的更新操作，即先从库中查询到具体数据记录，然后通过数据的指针引用进行修改操作。

`eosio_assert`方法主要用于对参数或业务逻辑判断的断言处理。

`user_stable.modify(item, _self, & `该行代码第一个参数为需要修改数据的地址引用，第二个参数为谁为此次修改操作花费相应的资源消耗。

### 3）删除

```js
void userlist::del(uint64_t id) {
  userlist::user_index user_stable(_self, _code.value);
  auto item =  user_stable.find(id);

  eosio_assert(item != user_stable.end(), "the data isn't exist.");

  user_stable.erase(item);
}
```

删除方法也是需要先定位到修改对象的地址引用，然后通过`erase`进行合约数据删除操作。

需要注意的是，`multi_index`本身并未提供清空整张表的 API 方法，所以如果要清空数据表只能选择循环迭代的方式一个个进行删除操作。但需要按以下方式进行删除：

```js
//清空表数据
userlist::user_index user_stable(_self, _code.value);
auto itr = user_stable.begin();
while(itr != user_stable.end()){
    itr.earse(itr);
    //注意：不可以添加 itr++
}
```

### 4）查询

在智能合约实现中，往往我们不需要对外提供数据查询的合约方法，因为 EOS 节点服务支持我们通过 RPC 接口的方式直接查询智能合约中的表数据，这样更加适合中心化服务灵活配置自己的查询需求。

**查询单条数据**

```js
userlist::user_index user_stable(_self, _code.value);

user_stable.find(id); //根据主键查询数据 
```

**查询多条数据**

```js
userlist::user_index user_stable(_self, _code.value);
auto ageindex = user_stable.get_index<"byage"_n>();
auto itr = ageindex.find(20);
while(itr != ageindex.end()){
    //print
}
```

以上示例为通过二级索引方式对年龄字段进行数据查询，即查询所有年龄为 20 的用户列表。

* * *

**小结** 本章节我们通过代码示例的形式向大家讲解了一个基础数据功能所需地技术知识点，以及如何分步骤实现一个具备增删改查功能的业务模块。除此之外，还重点讲解了如何对业务数据的多维度查询检索与遍历。

* * *

> 在教程中， 如果出现错误🐛或不易理解的知识点，欢迎加我微信指正! Name: zhangliang | WeChat: rushking2009 | Mail: zhangliang@cldy.org

* * *

**changelog** 2019-03-08 zhangliang(mailto:zhangliang@cldy.org)

*   初次发稿