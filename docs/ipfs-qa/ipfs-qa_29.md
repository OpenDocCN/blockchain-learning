# 第二十九章 【IPFS 一问一答】解析 IPFS Multiformat 之 Multibase

# 29 【IPFS 一问一答】解析 IPFS Multiformat 之 Multibase

multibase 代表的是一种编码格式， 方便把数据编码成不同的格式， 比如这里定义了 2 进制、8 进制、10 进制、16 进制、也有我们熟悉的 base58btc 和 base64 编码。

支持的编码格式如下：

```go
Multibase Table v1.0.0-RC (semver)

encoding      codes   name
identity      0x00    8-bit binary (encoder and decoder keeps data unmodified)
base1         1       unary tends to be 11111
base2         0       binary has 1 and 0
base8         7       highest char in octal
base10        9       highest char in decimal
base16        F, f    highest char in hex
base32        B, b    rfc4648 - no padding - highest letter
base32pad     C, c    rfc4648 - with padding
base32hex     V, v    rfc4648 - no padding - highest char
base32hexpad  T, t    rfc4648 - with padding
base32z       h       z-base-32 - used by Tahoe-LAFS - highest letter
base58flickr  Z       highest char
base58btc     z       highest char
base64        m       rfc4648 - no padding
base64pad     M       rfc4648 - with padding - MIME encoding
base64url     u       rfc4648 - no padding
base64urlpad  U       rfc4648 - with padding 
```

## 29.1 multibase 格式

`<varint-base-encoding-code><base-encoded-data>`

*   varint-base-encoding-code 从上面查表得到
*   base-encoded-data 数据

示例:

```go
4D756C74696261736520697320617765736F6D6521205C6F2F # base16 (hex)
JV2WY5DJMJQXGZJANFZSAYLXMVZW63LFEEQFY3ZP           # base32
YAjKoNbau5KiqmHPmSxYCvn66dA1vLmwbt                 # base58
TXVsdGliYXNlIGlzIGF3ZXNvbWUhIFxvLw==               # base64
F4D756C74696261736520697320617765736F6D6521205C6F2F # base16 F
BJV2WY5DJMJQXGZJANFZSAYLXMVZW63LFEEQFY3ZP           # base32 B
zYAjKoNbau5KiqmHPmSxYCvn66dA1vLmwbt                 # base58 z
MTXVsdGliYXNlIGlzIGF3ZXNvbWUhIFxvLw==               # base64 M 
```

它们的前缀分别是: F、B、z、 M。

## 29.2 multibase 安装

我们采用 Golang 来使用 multibase。

安装 multibase

`go get github.com/multiformats/go-multibase`

因为 multiaddr 包需要依赖包 gx，所以你需要以下操作，才能使用 multiaddr。

```go
go get -u github.com/whyrusleeping/gx
go get -u github.com/whyrusleeping/gx-go
cd <your-project-repository>
gx init
gx import github.com/multiformats/go-multibase
gx install --global
gx-go --rewrite 
```

## 29.3 multibase 源码分析

源码路径：src\github.com\multiformats\go-multibase\multibase.go

默认编码表

```go
// specified in standard are left out
var Encodings = map[string]Encoding{
    "identity":          0x00,
    "base16":            'f',
    "base16upper":       'F',
    "base32":            'b',
    "base32upper":       'B',
    "base32pad":         'c',
    "base32padupper":    'C',
    "base32hex":         'v',
    "base32hexupper":    'V',
    "base32hexpad":      't',
    "base32hexpadupper": 'T',
    "base58flickr":      'Z',
    "base58btc":         'z',
    "base64":            'm',
    "base64url":         'u',
    "base64pad":         'M',
    "base64urlpad":      'U',
}
```

编码：

```go
// Encode encodes a given byte slice with the selected encoding and returns a
// multibase string (<encoding><base-encoded-string>). It will return
// an error if the selected base is not known.
func Encode(base Encoding, data []byte) (string, error) 
```

解码：

```go
// Decode takes a multibase string and decodes into a bytes buffer.
// It will return an error if the selected base is not known.
func Decode(data string) (Encoding, []byte, error) 
```