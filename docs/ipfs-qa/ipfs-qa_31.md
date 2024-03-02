# 第三十一章 【IPFS 一问一答】解析 IPFS Multiformat 之 Multistream

# 31 【IPFS 一问一答】解析 IPFS Multiformat 之 Multistream

multistream 利用 multicodec，实现了自描述的功能，下面是基于一个 javascript 的例子；先 new 一个 buffer 对象，里面是 json 对象，然后给它加一个前缀 protobuf，这样这个 multistream 就构造好了，可以通过网络传输。在解析时可以先取 codec 前缀，然后移除前缀，得到具体的数据内容。

```go
// encode some json
const buf = new Buffer(JSON.stringify({ hello: 'world' }))

const prefixedBuf = multistream.addPrefix('json', str) // prepends multicodec ('json')
console.log(prefixedBuf)
// <Buffer 06 2f 6a 73 6f 6e 2f 7b 22 68 65 6c 6c 6f 22 3a 22 77 6f 72 6c 64 22 7d>

const.log(prefixedBuf.toString('hex'))
// 062f6a736f6e2f7b2268656c6c6f223a22776f726c64227d

// let's get the Codec and then get the data back

const codec = multicodec.getCodec(prefixedBuf)
console.log(codec)
// json

console.log(multistream.rmPrefix(prefixedBuf).toString())
// "{ \"hello\": \"world\" } 
```

输出结果：

```go
hex:   062f6a736f6e2f7b2268656c6c6f223a22776f726c64227d
ascii: /json\n"{\"hello\":\"world\"}"
```

## 31.1 multistream 源码分析

源码路径：src\github.com\multiformats\go-multistream\multistream.go

安装 go-multistream

`go get github.com/multiformats/go-multistream`

我们通过多路复用器让用户添加不同的处理程序来处理不同的“协议”:

```go
package main

import (
    "fmt"
    "io"
    "io/ioutil"
    "net"
    ms "github.com/multiformats/go-multistream"
)

// 创建多路复用器, 针对不用的协议创建不同的处理函数
// 创建 "/cats" and "/docs" 两种不同的协议
func main() {
    mux := ms.NewMultistreamMuxer()
    mux.AddHandler("/cats", func(proto string, rwc io.ReadWriteCloser) error {
        fmt.Fprintln(rwc, proto, ": HELLO I LIKE CATS")
        return rwc.Close()
    })
    mux.AddHandler("/dogs", func(proto string, rwc io.ReadWriteCloser) error {
        fmt.Fprintln(rwc, proto, ": HELLO I LIKE DOGS")
        return rwc.Close()
    })

    list, err := net.Listen("tcp", ":8765")
    if err != nil {
        panic(err)
    }

    go func() {
        for {
            con, err := list.Accept()
            if err != nil {
                panic(err)
            }

            go mux.Handle(con)
        }
    }()

    // 开始测试
    conn, err := net.Dial("tcp", ":8765")
    if err != nil {
        panic(err)
    }

    // /cats 协议
    mstream := ms.NewMSSelect(conn, "/cats")
    cats, err := ioutil.ReadAll(mstream)
    if err != nil {
        panic(err)
    }
    fmt.Printf("%s", cats)
    mstream.Close()

    conn, err = net.Dial("tcp", ":8765")
    if err != nil {
        panic(err)
    }
    defer conn.Close()
    // /dogs 协议
    err = ms.SelectProtoOrFail("/dogs", conn)
    if err != nil {
        panic(err)
    }
    dogs, err := ioutil.ReadAll(conn)
    if err != nil {
        panic(err)
    }
    fmt.Printf("%s", dogs)
    conn.Close()
}
```

操作：

```go
> go run multistream.go
/cats :HELLO I LIKE CATS
/dogs :HELLO I LIKE DOGS
```