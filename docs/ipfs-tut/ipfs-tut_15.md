# 四、.2 教程 2: 创建新的 hash，将其作为发布网页的 IPNS 值

教程 1 中的 IPNS 值是默认的节点 IDhash，如果我们上传多个不同文件，那就没法用了，因为只能给一个用，所以引出本节课程。

## 课程目标

*   通过本小节的学习，使你能够创建自己的 IPNS 值
*   用 IPNS 发布自己的文件

## 学习步骤

### 第一步：创建一个自己的 IPNS 值

```go
 > ipfs key gen --type=rsa --size=2048 mykey
    > ipfs name publish --key=mykey QmSomeHash 
```

以上是系统生成 IDhash 和发布 IPNS 的命令。因为我们在使用时，没有加后缀，所以走的都是默认值。

接下来，我们同样使用上边的命令，创建自己的 IPNS 值。 mykey 是个文件夹名字，里面的内容决定了产生的 hash。所以我们创建一个新的文件夹，这里我创建 mykey1，里面文件是 info.txt。内容“我的第一个 IPNS 值”。然后使用`ipfs key gen --type=rsa --size=2048 mykey1`，生成一个 hash 值，也就是我说的 IPNS 值。

操作：

```go
$ mkdir mykey1
$ echo "我的第一个 IPNS 值" > mykey1/info.txt
localhost:IPFS zhanghengxing$ ipfs key gen --type=rsa --size=2048 mykey1
QmX4KM1J82p895JveV3xfL4aNoEPqaiUmMEkTiaHMgj8Ru 
```

### 第二步：发布

和之前讲过的一样，不做重复，命令： `ipfs name publish --key=mykey1 QmSomeHash`

注：QmSomeHash 是你上传文件的 hash 值。