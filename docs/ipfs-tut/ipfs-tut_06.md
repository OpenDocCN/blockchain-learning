# 二、.2 教程 2: 将文件添加到本地 IPFS 存储库中

## 教程目标

通过学习本小节，使你能够：

*   添加一个文件到本地的 IPFS 存储库，包括了文件内容、文件名称、产生的目录
*   添加目录文件到本地 IPFS 存储库
*   明白为什么 IPFS 中，两个不同的文件，内容却是相同的
*   通过使用目录的 hash 来读取该目录下文件的内容

## 学习步骤

### 第一步：创建一个你即将添加的文件

之前的课程中，我们已经有了文件 mytextfile.txt。但是最好还是按下边的 pipe 命令，确保你的文件内容和教程中的一样。

```go
$ echo "chaindesk is a good IT service Community" > mytextfile.txt
```

### 第二步：添加文件到本地 IPFS 存储库中

命令很简单，就是在命令中加上个`-w`，如下：

```go
$ ipfs add -w mytextfile.txt 
added QmWKGV27xJZdZ9DzpFZMrU2RyFuVE6XEhUHRU7HMhvApYF mytextfile.txt
added QmV2HGWGF9XY4svFxyzsjawKEnW7R65dCi6Kx9vxbowCKo 
```

之前的教程中，我们上传文件的命令是`ipfs add mytextfile1.txt`，并没有`-w`的标签，最后返回了只是一个 hash 值。而这次操作，我们发现，返回了 2 个 hash 值。第一个 hash 值指的是文件内容对应的 hash 值；而第二个是上传文件后返回的目录 hash，该目录下的文件就是 mytextfile.txt。

### 第三步：列出目录下的信息

命令中`-w`标签是让 ipfs 产生一个目录，该目录下的文件就是上传的文件。想要进一步了解这一块的，可以使用`ipfs add --help`命令获取更多的信息。

列出目录下文件的信息，使用的命令是`ipfs ls`。为了显示信息对应的指什么，我们将使用`-v`标签，以方便更好的阅读信息，操作如下：

```go
$ ipfs ls QmV2HGWGF9XY4svFxyzsjawKEnW7R65dCi6Kx9vxbowCKo
QmWKGV27xJZdZ9DzpFZMrU2RyFuVE6XEhUHRU7HMhvApYF 49 mytextfile.txt
$ ipfs ls -v QmV2HGWGF9XY4svFxyzsjawKEnW7R65dCi6Kx9vxbowCKo
Hash                                           Size Name
QmWKGV27xJZdZ9DzpFZMrU2RyFuVE6XEhUHRU7HMhvApYF 49   mytextfile1.txt
```

注：获取目录信息，我们必须用`ipfs ls`命令，而不是`ipfs cat`，后者是直接获取文件内容的。如果`ipfs cat 目录 hash`，返回的将是报错，如下：

```go
$ ipfs cat QmV2HGWGF9XY4svFxyzsjawKEnW7R65dCi6Kx9vxbowCKo
Error: this dag node is a directory
```

### 第四步：使用目录 hash 来读取目录下的文件内容

使用`ipfs cat`命令，而后边跟的是文件路径，操作如下：

```go
$ ipfs cat QmV2HGWGF9XY4svFxyzsjawKEnW7R65dCi6Kx9vxbowCKo/mytextfile.txt
chaindesk is a good IT service Community
```

### 注解

将文件内容添加到 IPFS 存储库时，IPFS 将计算文件内容的加密 hash，并将该 hash 返回给您。然后，可以使用这个 hash 去引用文件的内容，并将内容从 IPFS 存储库中读取出来。

为了跟踪文件名和路径等信息，IPFS 允许您在添加文件时“包装”目录和文件名信息。这样将会通过路径来检索 IPFS 存储库中的文件内容。