# 第二十八章 【IPFS 一问一答】解析 IPFS Multiformat 之 Multiaddr

# 28 【IPFS 一问一答】解析 IPFS Multiformat 之 Multiaddr

Multiaddr 是自我描述的网络地址。它很好的改进了目前的网络地址格式，旨在使网络成为未来的证明，可以组合和高效。

目前的寻址方案存在许多问题：

*   它们阻碍协议之间的协议迁移和互操作性。
*   它们没有很好的构造，但是很少有 X-over-Y 构造，但只能在经典 uri/url 或者 host:port 方案中解决。
*   它们不支持多路复用的：它们是地址端口，而不是进程。
*   它们是隐式的，因为它们假设 out-of-band 值和上下文。
*   它们没有高效的机器可读表示。

Multiaddr 改进了这些问题，它的特性是：

*   Multiaddrs 支持任何网络协议的地址。
*   Multiaddrs 是自我描述的。
*   Multiaddrs 符合简单的语法，使得解析和构造变得微不足道。
*   Multiaddrs 具有人类可读且高效的机器可读表示。
*   Multiaddrs 封装良好，允许琐碎的包装和封装层的展开。

来看一下两者的结构事例：

普通标示：

```go
127.0.0.1:9090   # ip4\. is this TCP? or UDP? or something else?
[::1]:3217       # ip6\. is this TCP? or UDP? or something else?

http://127.0.0.1/baz.jpg
http://foo.com/bar/baz.jpg
//foo.com:1234
 # use DNS, to resolve to either ip4 or ip6, but definitely use
 # tcp after. or maybe quic... >.<
 # these default to TCP port :80. 
```

而用 multiaddr 方法标示：

```go
/ip4/127.0.0.1/udp/9090/quic
/ip6/::1/tcp/3217
/ip4/127.0.0.1/tcp/80/http/baz.jpg
/dns4/foo.com/tcp/80/http/bar/baz.jpg
/dns6/foo.com/tcp/443/https 
```

使用 multiaddr 后，表达的更清晰，而且格式上看着更人性化，可读性高了。

## 28.1 multiaddr 格式

Multiaddr 采用 递归 TLV（type+length+value）的格式进行编码，它有两种形式：

*   人类友好版本（采用 UTF-8 编码）。
*   机器友好版本，用于存储，传输。

人类友好格式定义如下:

`(/<addr-protocol-str-code>/<addr-value>)+`

*   路径符号嵌套协议和地址，例如：/ip4/127.0.0.1/udp/4023/quic
*   类型（addr-protocol-str-code）是标识网络协议的字符串代码。 协议表是可配置的。
*   值是网络地址值，采用字符串形式。

机器友好的格式定义如下：

`(<addr-protocol-code><addr-value>)+`

*   类型（addr-protocol-code）是标识网络协议的变量整数。 协议表是可配置的。
*   长度 是一个无符号变量整数，用于计算地址值的长度（以字节为单位）
*   值（addr-value） 是网络地址长度值

## 28.2 multiaddr 安装并使用

我们采用 Golang 来使用 multiaddr。

安装 multiaddr

`go get github.com/multiformats/go-multiaddr`

代码：

```go
package main

import (
    "fmt"

    ma "github.com/multiformats/go-multiaddr"
)

func main() {
    // construct from a string (err signals parse failure)
    m1, err := ma.NewMultiaddr("/ip4/127.0.0.1/udp/1234")
    if err != nil {
        panic(err)
    }
    // construct from bytes (err signals parse failure)
    m2, err := ma.NewMultiaddrBytes(m1.Bytes())
    if err != nil {
        panic(err)
    }

    // true
    fmt.Println(m2.Equal(m1))
    fmt.Println(m1.Protocols())
    m3, err := ma.NewMultiaddr("/sctp/5678")
    if err != nil {
        panic(err)
    }
    fmt.Println(m1.Encapsulate(m3))
    m4, err := ma.NewMultiaddr("/udp/1234")
    if err != nil {
        panic(err)
    }
    fmt.Println(m1.Decapsulate(m4))
} 
```

测试：

```go
> go run multiaddr.go
true
[{ip4 4 [4] 32 false {0x4cd930 0x4cdd00 <nil>}} {udp 273 [142 2] 16 false {0x4cdd70 0x4cdf80 <nil>}}]
/ip4/127.0.0.1/udp/1234/sctp/5678
/ip4/127.0.0.1 
```