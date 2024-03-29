# 10.1 编码规范

该章节主要介绍了在进行合约开发时，需要注意的一些关于帐户名称、合约名、方法及结构体定义时要遵守的编程规范。建立并培养良好的编程习惯，以及防止在正式环境上线时，导致意外问题的发生。比如：在私链环境中是可以随意定义 EOS 帐户名称的，但在线上环境只允许创建长度为 12 的帐户名。

* * *

## 命名规范

### EOS 帐户命名

*   名称仅允许包含.、a-z、1-5 三种类型的字符。即：`abcdefghijklmnopqrstuvwxyz12345.`
*   名称必须以字母开头且长度为 12。

### 合约命名规范

| 类型 | 命名规范 |
| --- | --- |
| Structs | 全部小写，单词之前下划线连接。长度不可超过 12 个字符。 |
| Classes | 全部小写，单词之前下划线连接。长度不可超过 12 个字符。 |
| Methods | 全部小写，单词之前下划线连接。长度不可超过 12 个字符。 |
| Types | 全部小写，单词之前下划线连接。 |
| Template Arguments | 驼峰命令 |
| Macros | 全部大写 |

## 代码注释

*   单行注释 / **Comments** /
*   多行注释 / *Comment Block* /

## 代码缩进

用 3 个空格代替 Tabs

* * *

> 在教程中如出现错误🐛或不易理解的知识点，欢迎加我微信指正! Name: zhangliang | WeChat: rushking2009 | Mail: zhangliang@cldy.org

![Show me your code.](img/9c507c40d372f5692d061c802a44deb2.jpg "加群了解")![](img/aab6c923225b0a35b6580de17534641d.jpg)

注： 有想了解**愿码全思维 IT 工程师加速器**的朋友，可以扫码加群咨询。

* * *

**changelog** 2019-03-06 zhangliang

*   初次发稿