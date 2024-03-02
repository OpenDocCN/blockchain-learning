# 第十四章 【分布式存储系统 etcd】etcd-manage 项目——添加 key

# etcd-manage 项目——添加 key

我们要通过页面上的 Add 按钮，进行添加 key，可以添加 key-value 数值，也可以新建 dir 目录。

打开 v1.go 文件：

首先添加一个函数 postEtcdKey()，用于实现添加一个 key：

```go
// 添加 key
func postEtcdKey(c *gin.Context) {
    saveEtcdKey(c, false)
}
```

并注册路由：

```go
func V1(v1 *gin.RouterGroup){
    v1.GET("/members", getEtcdMembers) // 获取节点列表

    v1.GET("/server", getEtcdServerList) // 获取 etcd 服务列表

    v1.POST("/key", postEtcdKey)       // 添加 key

}
```

然后我们去实现 saveEtcdKey()函数，在这之前，我们先添加一个结构体：

```go
// PostReq 添加和修改时的 body
type PostReq struct {
    *etcdv3.Node
    EtcdName string `json:"etcd_name"`
}
```

并在 model.go 文件中，添加一个 Node 结构体：

```go
 // Node 需要使用到的模型
type Node struct {
    IsDir   bool   `json:"is_dir"`
    Version int64  `json:"version,string"`
    Value   string `json:"value"`
    FullDir string `json:"full_dir"`
}
```

因为在添加的时候，可以是直接添加的 key-value 数据，也可以是创建目录，所以我们要区分开。

然后在 program/etcdv3 目录下新建 go 文件：keys.go，

```go
 // Value 获取一个 key 的值
func (c *Etcd3Client) Value(key string) (val *Node, err error) {
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    resp, err := c.Client.Get(ctx, key)
    if err != nil {
        return
    }
    if resp.Kvs != nil && len(resp.Kvs) > 0 {
        val = &Node{
            Value:   string(resp.Kvs[0].Value),
            FullDir: key,
            Version: resp.Kvs[0].Version,
        }
    } else {
        err = ErrorKeyNotFound
    }
    return
}
```

这个主要就是先从 etcd 中获取 key 对应的 value 值，这里先附上 resp.Kvs 的结构体源代码：

```go
 type KeyValue struct {
    // key is the key in bytes. An empty key is not allowed.
    Key []byte `protobuf:"bytes,1,opt,name=key,proto3" json:"key,omitempty"`
    // create_revision is the revision of last creation on this key.
    CreateRevision int64 `protobuf:"varint,2,opt,name=create_revision,json=createRevision,proto3" json:"create_revision,omitempty"`
    // mod_revision is the revision of last modification on this key.
    ModRevision int64 `protobuf:"varint,3,opt,name=mod_revision,json=modRevision,proto3" json:"mod_revision,omitempty"`
    // version is the version of the key. A deletion resets
    // the version to zero and any modification of the key
    // increases its version.
    Version int64 `protobuf:"varint,4,opt,name=version,proto3" json:"version,omitempty"`
    // value is the value held by the key, in bytes.
    Value []byte `protobuf:"bytes,5,opt,name=value,proto3" json:"value,omitempty"`
    // lease is the ID of the lease that attached to key.
    // When the attached lease expires, the key will be deleted.
    // If lease is 0, then no lease is attached to the key.
    Lease int64 `protobuf:"varint,6,opt,name=lease,proto3" json:"lease,omitempty"`
} 
```

然后我们在添加一个函数用来返回 key 以及父路径，因为我们添加的 key 中可能包含上一级的父目录：比如 key 为 bb/cc，value 值为 haha，那我们是希望先创建 bb 目录，然后里面存储 key 为 cc，对应 value 值为 haha：

```go
 func (c *Etcd3Client) ensureKey(key string) (string, string) {
    key = strings.TrimRight(key, "/")
    if key == "" {
        return "/", ""
    }
    if strings.Contains(key, "/") == true {
        return key, path.Clean(key + "/../")
    } else {
        return key, ""
    }
} 
```

接下来在 etcdv3.go 中，添加一个常量值：

```go
const (
    DEFAULT_DIR_VALUE = "etcdv3_dir_$2H#%gRe3*t"
)
```

然后在 keys.go 文件中创建 Put()方法：

```go
 // Put 添加一个 key
func (c *Etcd3Client) Put(key string, value string, mustEmpty bool) error {
    // log.Println(key)
    key, parentKey := c.ensureKey(key)
    //  需要判断的条件
    cmp := make([]clientv3.Cmp, 0)

    if parentKey != "" {
        cmp = append(cmp, clientv3.Compare(
            clientv3.Value(parentKey),
            "=",
            DEFAULT_DIR_VALUE,
        ))
    }

    if mustEmpty {
        cmp = append(
            cmp,
            clientv3.Compare(
                clientv3.Version(key),
                "=",
                0,
            ),
        )
    } else {
        cmp = append(
            cmp,
            clientv3.Compare(
                clientv3.Value(key),
                "!=",
                DEFAULT_DIR_VALUE,
            ),
        )
    }

    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    txn := c.Client.Txn(ctx)
    // make sure the parentKey is a directory
    txn.If(
        cmp...,
    ).Then(
        clientv3.OpPut(key, value),
    )

    txnResp, err := txn.Commit()
    if err != nil {
        return err
    }

    if !txnResp.Succeeded {
        return ErrorPutKey
    }
    return nil
}
```

最后我们就可以开始调用函数添加 key 了，在 v1.go 中继续添加函数 saveEtcdKey()，因为后面要实现的修改 key 和添加 key 的本质操作差不多，我们设计为一个方法即可。但是还是需要传入个 isPut 参数来区分：通过 Post 操作是添加，通过 Put 操作是修改：

```go
 // 保存 key
func saveEtcdKey(c *gin.Context, isPut bool) {

    var err error
    defer func() {
        if err != nil {
            c.JSON(http.StatusBadRequest, gin.H{
                "msg": err.Error(),
            })
        }
    }()

    req := new(PostReq)
    err = c.Bind(req)
    if err != nil {
        return
    }
    if req.FullDir == "" {
        err = errors.New("参数错误")
        return
    }

    etcdCli, exists := c.Get("EtcdServer")
    if exists == false {
        err = errors.New("Etcd client is empty")
        return
    }
    cli := etcdCli.(*etcdv3.Etcd3Client)

    // 判断根目录是否存在
    rootDir := ""
    dirs := strings.Split(req.FullDir, "/")
    if len(dirs) > 1 {
        // 兼容/开头的 key
        if req.FullDir[:1] == "/" {
            _, err = cli.Value("/")
            if err != nil {
                err = cli.Put("/", etcdv3.DEFAULT_DIR_VALUE, true)
                if err != nil {
                    return
                }
            }
        }
        rootDir = strings.Join(dirs[:len(dirs)-1], "/")
    }
    if rootDir != "" {
        // 用/分割
        rootDirs := strings.Split(rootDir, "/")
        if len(rootDirs) > 1 {
            rootDir1 := ""
            for _, vDir := range rootDirs {
                if vDir == "" {
                    vDir = "/"
                }
                if rootDir1 != "" && rootDir1 != "/" {
                    rootDir1 += "/"
                }
                rootDir1 += vDir
                _, err = cli.Value(rootDir1)
                if err != nil {
                    err = cli.Put(rootDir1, etcdv3.DEFAULT_DIR_VALUE, true)
                    if err != nil {
                        return
                    }
                }
            }
        } else {
            _, err = cli.Value(rootDir)
            if err != nil {
                err = cli.Put(rootDir, etcdv3.DEFAULT_DIR_VALUE, true)
                if err != nil {
                    return
                }
            }
        }
    }

    // 保存 key
    if req.IsDir == true {
        if isPut == true {
            err = errors.New("目录不能修改")
        } else {
            err = cli.Put(req.FullDir, etcdv3.DEFAULT_DIR_VALUE, !isPut)
        }
    } else {
        err = cli.Put(req.FullDir, req.Value, !isPut)
    }

    if err != nil {
        return
    }

    c.JSON(http.StatusOK, "ok")
} 
```

到此我们可以实现添加 key 的功能了，只是目前还无法查看到效果。

[源代码](https://github.com/rubyhan1314/myetcd-manage)