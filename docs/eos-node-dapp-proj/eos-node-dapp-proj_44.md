# 十一、.6 Multi-Index 完整用法及示例讲解---数据存储篇

上一小节介绍了如何针对一个数据结构定义对应的索引器，那么本小节将介绍如何利用 eosio::multi_index 所提供数据操作方法对数据进行添加、修改、删除。

### 1\. 添加方法

**eosio::multi_index::emplace**为 eosio::multi_index 所提供的添加方法，其方法参数为： **payer**、**Lambda function**

```js
const_iterator eosio::multi_index::emplace (
    name payer,
    Lambda && constructor
)
```

**emplace 参数说明**：

*   **payer**: 支付资源帐户。
    即：存储数据所需消耗资源的支付帐户
*   **Lambda && constructor**： Lambda 函数。
    例如: `& { //TODO do some stuff } ` 另外，在传统数据库表设计中，通常我们都会将主键设置成自增类型。那么，同样智能合约中也可以实现自增 ID，可以通过索引器提供的 eosio::multi_index::available_primary_key ()方法获取下一个 ID 并对主键进行赋值。

**代码示例** ：新建一条用户数据

```js
void createuser(name username, uint64_t age) {
    user_index userestable(_self, _self.value); // code, scope

    userestable.emplace(username, & {
        user.account_name = username;
        user.age = age;
    });
}
```

### 2\. 修改方法

** eosio::multi_index::index::modify**为**eosio::multi_index**索引器提供的修改数据方法。

```js
void eosio::multi_index::index::modify (
    const_iterator itr,
    eosio::name payer,
    Lambda && updater
)
```

参数说明：

*   **itr** 该参数主要用于接受所要修改数据实例对应的 const_iterator 对象，而非对象实例本身。**const_iterator 对象**或者可以理解为数据实例的引用地址。
*   **payer** 资源消耗付费帐户
    即：修改数据所需消耗资源的支付帐户
*   **Lambda && updater**
    例如: `& { //TODO do some stuff } `

**代码示例**: 修改一个用户的年龄 age 属性

```js
void modifyuser(name username, uint64_t age) {
    user_index userestable(_self, _self.value); // code, scope

    auto iter = userestable.find(username);

    userestable.modify(iter, username, & {
        user.age = age;
    });
}
```

### 3\. 删除方法

**eosio::multi_index::erase**同样也是由**eosio::multi_index**提供的数据删除方法。

```js
const_iterator eosio::multi_index::erase (
    const_iterator itr
)
```

**参数说明**：

*   itr 该参数主要用于接收**const_iterator**类型实例。而该类型实例其实就是对表数据实例的地址引用封装。

与添加、修改不同之处在于：删除数据时不需要指定资源支付帐户。因为删除数据相当于释放资源。在开发过程中，是提倡对无用资源的及时清理工作的。

**代码示例** ：删除一个用户

```js
void modifyuser(name username, uint64_t age) {
    user_index userestable(_self, _self.value); // code, scope

    auto iter = userestable.find(username);

    userestable.erase(iter);
}
```

* * *

通过本小节的学习，我们学会了如何使用 multi_index 索引的 emplace、modify、erase 方法来完成对合约数据的添加、修改、删除操作。

* * *

> 在教程中如出现错误🐛或不易理解的知识点，欢迎加我微信指正! Name: zhangliang | WeChat: rushking2009 | Mail: zhangliang@cldy.org

![](img/9c507c40d372f5692d061c802a44deb2.jpg)![](img/aab6c923225b0a35b6580de17534641d.jpg)

* * *

### **changelog**

2019-03-21 zhangliang(mailto:zhangliang@cldy.org)

*   初次发稿