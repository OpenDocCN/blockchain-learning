# 11.8 Multi-Index 完整用法及示例讲解---迭代器篇

在**eosio::multi_index**类中共计提供了 cbegin、begin、crbegin、rbegin、cend、end、crend、rend 八个迭代方法，共计四对对称方法

*   cbegin/cend
*   begin/end
*   crbegin/crend
*   rbegin/rend

虽然以上四对方法名称均有所不同，但从其源码实现上可以归为两种：

*   正向迭代器
    cbegin/cend、begin/end
*   反转迭代器
    crbegin/crend、rbegin/rend

可参考其具体源码：

```js
const_iterator cbegin()const {
    using namespace _multi_index_detail;
    return lower_bound( secondary_key_traits<secondary_key_type>::lowest() );
}
const_iterator begin()const  { return cbegin(); }

const_iterator cend()const { return const_iterator( this ); }
const_iterator end()const  { return cend(); }

const_reverse_iterator crbegin()const { return std::make_reverse_iterator(cend()); }
const_reverse_iterator rbegin()const  { return crbegin(); }

const_reverse_iterator crend()const   { return std::make_reverse_iterator(cbegin()); }
const_reverse_iterator rend()const    { return crend(); } 
```

正向迭代器其实就是按照索引由低到高的顺序进行遍历；而反转迭代器其实就是将正向迭代器进行了反转操作，即按照索引高到低的次序进行遍历。

而所有迭代器 begin 都特指当前排序的首个对象实例；而 end 都特指当前排序末尾结束符，注意并不是最后一个对象实例。

## 正向迭代器（cbegin/cend、begin/end）

**功能** ：按索引由低高到的次序进行数据遍历

**代码示例**

```js
user_index userestable(_self, _self.value); // code, scope

userestable.emplace(_self, &{
    s.account_name = 1;
    s.age = 19;
});
userestable.emplace(_self, &{
    s.account_name = 2;
    s.age = 18;
});
userestable.emplace(_self, &{
    s.account_name = 3;
    s.age = 1;
});

auto agestable = userestable.get_index<"byage"_n>();

//正向迭代，索引由低到高的顺序
auto iter = agestable.begin(); // or agestable.cbegin()

eosio_assert(iter->account_name == 3, "Incorrect First Record ");
iter++;
eosio_assert(iter->account_name == 2, "Incorrect Second Record");
iter++;
eosio_assert(iter->account_name == 1, "Incorrect Third Record");
iter++;

eosio_assert(iter == agestable.end(), "Incorrect End of Iterator"); // or agestable.cend() 
```

## 反转迭代器（crbegin/crend、rbegin/rend）

**功能** ：按索引由高到低的次序进行数据遍历

**代码示例**

```js
user_index userestable(_self, _self.value); // code, scope

userestable.emplace(_self, &{
    s.account_name = 1;
    s.age = 19;
});
userestable.emplace(_self, &{
    s.account_name = 2;
    s.age = 18;
});
userestable.emplace(_self, &{
    s.account_name = 3;
    s.age = 1;
});

auto agestable = userestable.get_index<"byage"_n>();

//反转迭代，索引由高到低的顺序
auto iter = agestable.rbegin(); //or agestable.crbegin()

eosio_assert(iter->account_name == 1, "Incorrect First Record ");
iter++;
eosio_assert(iter->account_name == 2, "Incorrect Second Record");
iter++;
eosio_assert(iter->account_name == 3, "Incorrect Third Record");
iter++;

eosio_assert(iter == agestable.rend(), "Incorrect End of Iterator"); //or agestable.crend() 
```

* * *

最后，让我们回顾一下整篇文章的知识点。首次，讲解了如何针对数据结构体进行主键及二级索引字段的索引器定义；然后，讲解了如何利用索引器对数据进行添加、修改、删除操作；紧接着，又讲解了如何对合约表数据进主键、二级索引字段进行单条记录、区间记录查询；最后，又讲解了如何利用迭代器对数据正向或反向进行遍历。