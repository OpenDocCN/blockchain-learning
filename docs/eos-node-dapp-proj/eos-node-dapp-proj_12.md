# 6.3 TOKEN 管理--代码实现

## 交易所智能合约之 TOKEN 管理代码实现

在\<\<TOKEN 管理>>章节中，我们介绍了 TOKEN 管理所涉及的数据模型、业务逻辑、约束条件以及合约方法的权限要求，那么本章节将会根据这些功能描述进行具体代码实现。

在所有的功能实现中，我们将遵循数据结构定义、索引器定义、接口方法定义以及方法实现四个步骤进行分步讲解实现：

* * *

### 第一步、定义 TOKEN 数据结构

在\<\<TOKEN 管理>>数据结构定义，定义了四个字段。其中使用了一个可能比较陌生的字段类型 即：eosio::extended_symbol。 该数据类型其实是由 eoslib 库所封装的一种存储合约的类型，主要用于记录 TOKEN 名称、精度以及 TOKEN 所归属的合约名称三项信息。

其具体实现代码如下所示：

```js
struct [[eosio::table]] token_struct {
    uint64_t id;
    string symbol_name;
    eosio::extended_symbol ext_symbol;
    uint64_t precision;

    uint64_t get_by_symbol() const { 
        return ext_symbol.get_symbol().raw(); 
    }

    uint64_t get_by_contract() const { 
        return ext_symbol.get_contract().value; 
    }

    uint64_t primary_key() const { return id; }

    EOSLIB_SERIALIZE( tokenstruct, (id)(symbol_name)(ext_symbol)(precision))
};
```

其中, symbol_name 字段用于存储 TOKEN 对应的币名称；ext_symbol 字段主要用于存储合约信息； `precision `主要用于定义币种的数据精度。

该段代码前半部分，定义了业务所必需的字段定义；之后，你会发现三个方法 get_by_symbol、get_by_contrac、primary_key，这些方法主要是为了给索引器提供索引查询使用；而最后 EOSLIB_SERIALIZE，主要用于对合约数据的序列化与反序列化，方便对合约的存储与读取。

需要注意的是如果想让智能合约 abi 文件出现此结构体的数据定义，那就需要在表结构中添加`[[eosio::table]] `注解。

**注意事项：**

**问题一**: 当数据表中存在数据后，对表结构添加新字段后，发现无法读取数据。 **解决办法**： 备份或删除原有数据，修改完数据结构体后，再将数据导入进来。

**问题二**: 定义数据结构体时，如何选择使用什么数据类型 ？ **解决办法：** 智能合约所可以使用的数据类型主要分为两种：第一种为 C++语言本身所支持的基础数据类型，大家可能过[`www.cplusplus.com`](http://www.cplusplus.com)网站进行查询；另外一种，便是 EOS 所封装的数据类型，可能过[`eosio.github.io/eosio.cdt/1.3.2modules.html`](https://eosio.github.io/eosio.cdt/1.5.0/modules.html)网站进行查询，当然也可以通过 cdt 所提供的[eoslib 源码库](https://github.com/EOSIO/eosio.cdt/blob/v1.3.2/libraries/eosiolib/action.hpp)查看其源码实现，只不过在阅读源码时一定要注意是查看的哪个版本的实现。比如：1.3.x vs 1.2.x 版本之前是有很大差异的，在 1.3.x 版中就已经弃用了很多数据类型。

* * *

### 第二步、定义索引器

在实现完 TOKEN 结构体后，接下来需要思考并实现的便是：如何定义对数据的多索引数据访问？

比如：

*   1）如何通过**主键**进行数据查询？
*   2）如何通过**合约帐户**进行数据查询？
*   3）如何通过**币种合称**进行数据查询？

基于以上功能需求点， 我们将采用 multi_index 实现主键及二级索引定义的代码实现：

```js
typedef eosio::multi_index< "tokenstruct"_n, tokenstruct, 
    indexed_by< "bysymbol"_n, const_mem_fun<tokenstruct, uint64_t, &tokenstruct::get_by_symbol>>,
    indexed_by< "bycontract"_n, const_mem_fun<tokenstruct, uint64_t, &tokenstruct::get_by_contract>>
> TokenstructIndex;
```

上述代码实现了三种查询方式：

*   基于主键
*   基于 TOKEN 名称
*   基于合约名称

在定义主键或二级索引时，需要注意索引字段的数据类型，目前主键数据类型只支持 uint64_t， 而二级索引字段只支持 uint64_t、uint128_t、uint256_t、double、long double 五种。

**注意事项** **问题一**：如何实现对字符串类型的字段进行索引查询？ **解决办法** ：可在索引字段 get 方法中进行数据类型转换，比如将字符串转为其他支持的数据类型

* * *

### 第三步、定义接口方法定义

在交易所合约 hpp 文件的 public 代码块中，定义 TOKEN 功能的添加、修改、删除方法

```js
 [[eosio::action]]
void createtoken(uint64_t token_id, string symbol_name, eosio::extended_symbol ext_symbol, uint64_t precision);

[[eosio::action]]
void updatetoken(uint64_t token_id, string symbol_name, eosio::extended_symbol ext_symbol, uint64_t precision);

[[eosio::action]]
void deletetoken(uint64_t token_id, string symbol_name);
```

同样，在定义合约接口方法时，需要注意添加注解[[eosio::action]]， 以便生成合约方法在 ABI 文件中的定义。

* * *

### 第四步、对接口方法进行扩展实现

上一个步骤完成了对合约方法的接口定义，接下来进行具体的代码实现：

**A、**实现 **创建 TOKEN** 合约方法

```js
void desert::createtoken(uint64_t id, string symbol_name, eosio::extended_symbol ext_symbol, uint64_t precision){

    require_auth(_self);    

    //TODO 1\. 参数数据校验

    //TODO 2\. 业务数据校验

    //TODO 3\. 合约数据存储
}
```

整个数据处理过程，分为三个步骤：

1.  参数数据校验， 通过断言对参数进行检验；

    ```js
    eosio_assert( id > 0, "Id not allow less than or equal to 0");
    eosio_assert( symbol_name != "", "symbol_name not allow null or blank");
    eosio_assert( ext_symbol.get_symbol().is_valid(), "symbol valid error");
    eosio_assert( ext_symbol.get_contract() != ""_n , "contract not allow null or blank");
    eosio_assert( precision > 0 , "precision not allow less than or equals to 0");
    ```

    主要对于一些必填字段，以及符合一定规范的参数进行基础数据校验。比如：判断字段值是否为空，判断合约信息是否正确

2.  业务数据校验 该部分代码主要用于根据当前功能业务验证其逻辑正确性。比如：所要保存的 TOKEN 名称是否已经存在、所要保存的合约信息是否已经存在等等。

    ```js
    TokenstructIndex tokenstable(_self, _self.value);
    auto token_entry = tokenstable.find(id);
    eosio_assert(token_entry == tokenstable.end(), "id already existed");

    auto symbol_index = tokenstable.get_index<"bysymbol"_n>();
    auto symbol_entry = symbol_index.find(ext_symbol.get_symbol().raw());
    eosio_assert(symbol_entry == symbol_index.end(), "symbol already existed");
    ```

3.  合约数据存储 当所有数据校验验证通过之后，便可以对数据进行数据保存。

    ```js
    tokenstable.emplace( _self, & {
        s.id = id;
        s.symbol_name = symbol_name;
        s.ext_symbol = ext_symbol;
        s.precision = precision;
    });
    ```

**完整示例代码**

```js
void desert::createtoken(uint64_t id, string symbol_name, eosio::extended_symbol ext_symbol, uint64_t precision){
    require_auth(_self);

    eosio_assert( id > 0, "Id not allow less than or equal to 0");
    eosio_assert( symbol_name != "", "symbol_name not allow null or blank");
    eosio_assert( ext_symbol.get_symbol().is_valid(), "symbol valid error");
    eosio_assert( ext_symbol.get_contract() != ""_n , "contract not allow null or blank");
    eosio_assert( precision > 0 , "precision not allow less than or equals to 0");

    TokenstructIndex tokenstable(_self, _self.value);
    auto token_entry = tokenstable.find(id);
    eosio_assert(token_entry == tokenstable.end(), "id already existed");

    auto symbol_index = tokenstable.get_index<"bysymbol"_n>();
    auto symbol_entry = symbol_index.find(ext_symbol.get_symbol().raw());
    eosio_assert(symbol_entry == symbol_index.end(), "symbol already existed");

    tokenstable.emplace( _self, & {
        s.id = id;
        s.symbol_name = symbol_name;
        s.ext_symbol = ext_symbol;
        s.precision = precision;
    });
}
```

B、实现 **修改 TOKEN** 合约方法 该方法在具体的实现逻辑上与**创建 TOKEN 方法**相仿。

```js
void desert::updatetoken(uint64_t id, string symbol_name, eosio::extended_symbol ext_symbol, uint64_t precision){
    require_auth(_self);

    //TODO 1\. 参数校验
    //TODO 2\. 业务校验
    //TODO 3\. 数据更新
}
```

整个实现环节，分为三个步骤：

1.  参数校验

    ```js
    eosio_assert( id > 0, "Id not allow less than or equal to 0");
    eosio_assert( symbol_name != "", "symbol_name not allow null or blank");
    eosio_assert( ext_symbol.get_symbol().is_valid(), "symbol valid error");
    eosio_assert( ext_symbol.get_contract() != ""_n , "contract not allow null or blank");
    eosio_assert( precision > 0 , "precision not allow less than or equals to 0");
    ```

2.  业务校验

    ```js
    TokenstructIndex tokenstable(_self, _self.value);
    auto token_entry = tokenstable.find(id);
    eosio_assert(token_entry != tokenstable.end(), "token does not exist");

    auto symbol_index = tokenstable.get_index<"bysymbol"_n>();
    auto symbol_entry = symbol_index.find(ext_symbol.get_symbol().raw());
    eosio_assert(symbol_entry == symbol_index.end() || token_entry->id == symbol_entry->id , "symbol already existed");
    ```

3.  数据更新

    ```js
    tokenstable.modify(token_entry, _self, & {
        s.symbol_name = symbol_name;
        s.ext_symbol = ext_symbol;    
        s.precision = precision;
    });
    ```

* * *

**完整示例代码**

```js
void desert::updatetoken(uint64_t id, string symbol_name, eosio::extended_symbol ext_symbol, uint64_t precision){
    require_auth(_self);

    eosio_assert( id > 0, "Id not allow less than or equal to 0");
    eosio_assert( symbol_name != "", "symbol_name not allow null or blank");
    eosio_assert( ext_symbol.get_symbol().is_valid(), "symbol valid error");
    eosio_assert( ext_symbol.get_contract() != ""_n , "contract not allow null or blank");
    eosio_assert( precision > 0 , "precision not allow less than or equals to 0");

    TokenstructIndex tokenstable(_self, _self.value);
    auto token_entry = tokenstable.find(id);
    eosio_assert(token_entry != tokenstable.end(), "token does not exist");

    auto symbol_index = tokenstable.get_index<"bysymbol"_n>();
    auto symbol_entry = symbol_index.find(ext_symbol.get_symbol().raw());
    eosio_assert(symbol_entry == symbol_index.end() || token_entry->id == symbol_entry->id , "symbol already existed");

    tokenstable.modify(token_entry, _self, & {
        s.symbol_name = symbol_name;
        s.ext_symbol = ext_symbol;    
        s.precision = precision;
    });
}
```

C、实现 **删除 TOKEN** 合约方法

```js
void desert::deletetoken(uint64_t id, string symbol_name){
    require_auth(_self);

    //TODO 1\. 参数校检
    //TODO 2\. 业务逻辑
    //TODO 3\. 数据存储
}
```

整个实现环节，具体可分为三个步骤：

1.  参数校检

    ```js
    eosio_assert( id > 0, "Id not allow less than or equal to 0");
    eosio_assert( symbol_name.length() > 0, "symbol_name not allow null or blank");
    ```

2.  业务逻辑

    ```js
    TokenstructIndex tokenstable(_self, _self.value);
    auto token_entry = tokenstable.find(id);
    eosio_assert(token_entry != tokenstable.end(), "token does not exist");
    eosio_assert(token_entry->symbol_name == symbol_name, "synbol_name is not correct");
    ```

此方法中 TOKEN 名称参数，主要用于让用户做数据确认，防止对数据的误删操作。

3.  数据存储

    ```js
    tokenstable.erase(token_entry);
    ```

**完整示例代码**

```js
void desert::deletetoken(uint64_t id, string symbol_name){
    require_auth(_self);

    eosio_assert( id > 0, "Id not allow less than or equal to 0");
    eosio_assert( symbol_name.length() > 0, "symbol_name not allow null or blank");

    TokenstructIndex tokenstable(_self, _self.value);
    auto token_entry = tokenstable.find(id);
    eosio_assert(token_entry != tokenstable.end(), "token does not exist");
    eosio_assert(token_entry->symbol_name == symbol_name, "synbol_name is not correct");

    tokenstable.erase(token_entry);
}
```