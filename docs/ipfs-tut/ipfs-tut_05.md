# 2.1 教程 1: 将文件内容添加到本地 IPFS 存储库中，并对它进行读取

## 教程目标

通过本小节的学习，你会：

*   将文件里的内容添加到本地 IPFS 存储库中
*   使用 hash 值检索到对应的数据
*   明白 IPFS 的 hash 值和你上传文件内容的关系

## 学习步骤

### 第一步: 创建你要添加的文件

你可以上传任何类型的内容到 IPFS。本教程我们使用的后缀为“.txt”的文件，但是你可以替换成其他任何文件类型，当然操作的步骤都是一样的。

我们在桌面上创建一个文件夹，命令如下：

```go
$ cd ~/Desktop
$ mkdir ipfs-tutorial
$ cd ipfs-tutorial 
```

然后在 ipfs-tutorial 目录下，创建一个 mytextfile.txt 文件，内容是“chaindesk is a good service Community”。在终端用命令很容易的就能创建这个文件，命令如下：

```go
echo "chaindesk is a good IT service Community" > mytextfile.txt 
```

使用 cat 命令，查看文件的信息，命令如下：

```go
$ cat mytextfile.txt
chaindesk is a good IT service Community 
```

### 第二步：添加文件内容到本地 IPFS 存储库

命令很简单，ipfs add 文件名，操作和显示如下：

```go
$ ipfs add mytextfile.txt 

added QmWKGV27xJZdZ9DzpFZMrU2RyFuVE6XEhUHRU7HMhvApYF mytextfile.txt
 41 B / 41 B [=========================================================] 100.00%
```

保存`QmWKGV27xJZdZ9DzpFZMrU2RyFuVE6XEhUHRU7HMhvApYF`。这个很长的字符串就是文件内容加密后的 hash 值。如果文件内容发生变化，那么这个 hash 值也会变，但是文件内容一样，返回的 hash 值都是一样的。

### 第三步：从本地 IPFS 存储库中读取内容

读取的命令很简单，和普通的`cat`命令类似，只不过这里用的是`ipfs cat`。而且 cat 后边跟的是 hash 值，而不是文件名。命令如下：

```go
$ ipfs cat QmWKGV27xJZdZ9DzpFZMrU2RyFuVE6XEhUHRU7HMhvApYF

chaindesk is a good IT service Community 
```

需要注意的是执行`ipfs add`命令后返回的 hash 值，与文件名没有关系，只和文件里面的内容有关联。

### 第四步：确认：IPFS 的 hash 指向的是文件内容，而不是文件本身

当我们使用`ipfs cat`命令时，返回的是文件的内容，而不是文件本身。这是因为 hash`QmWKGV...`是文件内容的 hash 值。我们可以测试一下，直接将内容上传到 IPFS，看下效果，命令如下：

```go
$ echo "chaindesk is a good IT service Community" | ipfs add

added QmWKGV27xJZdZ9DzpFZMrU2RyFuVE6XEhUHRU7HMhvApYF QmWKGV27xJZdZ9DzpFZMrU2RyFuVE6XEhUHRU7HMhvApYF
```

你会发现，返回的 hash 和之前上传 mytextfile.txt 文件返回的 hash 值一样。

你可以在保证文件内容一致的情况下，不断更改文件名字，然后上传到 IPFS，最终返回的 hash 值都是一样的。自己感兴趣的，可以测试下。

### 第五步：修改文件的内容，将会得到不一样的 hash 值

更改文件 mytextfile.txt 里的内容，并再次上传到 IPFS，将会返回不同的 hash 值，命令如下：

```go
$ vi  mytextfile.txt
$ ipfs add  mytextfile.txt
added QmWjoU6Nm3ofbiZWsZrX898uCTFEdgJJijv2CfL5dJ5C6o mytextfile.txt
```

注：vi 是打开并编辑文件内容。我在内容 chaindesk 前边加了个 The，这样文件就发生变化了，然后按 esc，输入：wq，退出保存。

从结果上看，返回的 hash 和之前的完全不一样。

### 第六步：从本地 IPFS 存储库读取内容，并使用`pipe`将其处理成一个文件

我们可以通过`ipfs cat`+hash 从 IPFS 读取 hash 对应的内容，并指定将内容存储到某个文件中。我们已经有了一个文件，即 mytextfile.txt。接下来就使用这个文件来存储读取到的内容，命令如下：

```go
$ ipfs cat QmWKGV27xJZdZ9DzpFZMrU2RyFuVE6XEhUHRU7HMhvApYF > mytextfile.txt
$ cat mytextfile.txt 
chaindesk is a good IT service Community
$ ipfs cat QmWjoU6Nm3ofbiZWsZrX898uCTFEdgJJijv2CfL5dJ5C6o > mytextfile.txt
$ cat mytextfile.txt 
The chaindesk is a good IT service Community 
```

当然你也可以创建新的文件，来存储内容：

```go
$ ipfs cat QmWKGV27xJZdZ9DzpFZMrU2RyFuVE6XEhUHRU7HMhvApYF > new
```

## 注解

IPFS 根据其加密 hash 跟踪内容。此 hash 唯一地标识了内容。只要内容保持不变，hash 值就保持不变，但是如果内容发生变化，您将得到不同的 hash 值。

如果有两个包含相同内容的不同文件，IPFS 将使用一个 hash 跟踪该内容。文件名是不同的，但内容是相同的，所以内容的 hash 值是相同的。

那么将会引出一个问题：怎么才能让 IPFS 跟踪文件名？这将是下一教程的主题。