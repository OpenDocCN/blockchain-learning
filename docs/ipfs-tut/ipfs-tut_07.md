# 2.3 教程 3: 将整个文件夹添加到本地 IPFS 存储库中

## 教程目标

通过学习本小节，使你能够：

*   添加一个文件夹到本地的 IPFS 存储库，包括了文件夹的的名称、文件夹下的所有内容等等
*   理解添加文件夹和文件操作的不同
*   通过路径读取文件内容

## 学习步骤

### 第一步：创建一个你即将上传的文件夹

之前我们已经在桌面上创建了一个文件夹 ipfs-tutorial。ok，我们接着用这个文件夹。在这个文件夹下边，我们已经有了一个文件 mytextfile.txt。我们还可以在文件夹下边继续创建新的文件或创建新的文件夹。总之文件夹下的内容不限。

接下来，我们在 ipfs-tutorial 下，创建一个新的文件夹 new，然后在 new 下创建一个文件 newfile，文件中的内容是“QQ 群：348924182；263270946”，命令如下：

```go
$ cd ipfs-tutorial/
$ ls
mytextfile.txt
$ mkdir new
$ ls
mytextfile.txt   new
$ cd new
$ vi newfile
$ ls
newfile
$ cd ..
$ ls
mytextfile.txt   new
$ cat new/newfile 
QQ 群：348924182；263270946 
```

### 第二步：将文件夹下的数据添加到本地 IPFS 存储库

命令很简单，就是在命令中加上个`-r`，如下：

```go
$ ipfs add -r ipfs-tutorial
added QmWKGV27xJZdZ9DzpFZMrU2RyFuVE6XEhUHRU7HMhvApYF ipfs-tutorial/mytextfile.txt
added QmaTbcssmxUTB7na2ggMDTBXaUbSJzJsT5xQEnj3KJx9VL ipfs-tutorial/new/newfile
added QmYFVyeUjycJ5pFPFqqebhiWP7TZQxdxw86SgMAVZGnQnm ipfs-tutorial/new
added QmNpRuzmmucd5sDoC7fjfZRgo2mnaXb35hpsLQf77A96xu ipfs-tutorial 
```

这个 r 的单词应该是 referring，特制文件夹下的所有数据。这样整个文件夹下的数据就添加到本地的 IPFS 存储库了，返回信息的最后部分，即 hash`QmNpRuzmmucd5sDoC7fjfZRgo2mnaXb35hpsLQf77A96xu`，就是访问的入口，该值和文件夹下的文件、文件内容、文件名称、目录结构都有关系。

你可以测试一下，如改变某个文件的名称或修改文件的内容或把文件夹下的数据位置更改下，然后添加到 IPFS，看返回的最后 hash 是都不一样。

### 第三步：使用`-w`标签，打包添加文件夹到 IPFS 存储库。

标签`-w`之前我们已经学习过。这里我们可以把文件夹看成一个整体，就好比是一个文件。执行命令如下：

```go
$ ipfs add -r -w ipfs-tutorial
added QmWKGV27xJZdZ9DzpFZMrU2RyFuVE6XEhUHRU7HMhvApYF ipfs-tutorial/mytextfile.txt
added QmaTbcssmxUTB7na2ggMDTBXaUbSJzJsT5xQEnj3KJx9VL ipfs-tutorial/new/newfile
added QmYFVyeUjycJ5pFPFqqebhiWP7TZQxdxw86SgMAVZGnQnm ipfs-tutorial/new
added QmNpRuzmmucd5sDoC7fjfZRgo2mnaXb35hpsLQf77A96xu ipfs-tutorial
added QmNmGoyiFpaHx1tra4zNL2iZHAfj8JM7gETjk2hZWVGWXi
```

列出目录下文件的信息，使用的命令是`ipfs ls`。为了显示信息对应的指什么，我们将使用`-v`标签，以方便更好的阅读信息，操作如下：

1.  列出文件夹

    ```go
    $ ipfs ls -v QmNmGoyiFpaHx1tra4zNL2iZHAfj8JM7gETjk2hZWVGWXi
    Hash                                           Size Name
    QmNpRuzmmucd5sDoC7fjfZRgo2mnaXb35hpsLQf77A96xu 245  ipfs-tutorial/
    ```

2.  列出文件夹中的数据

```go
$ ipfs ls -v QmNpRuzmmucd5sDoC7fjfZRgo2mnaXb35hpsLQf77A96xu
Hash                                           Size Name
QmWKGV27xJZdZ9DzpFZMrU2RyFuVE6XEhUHRU7HMhvApYF 49   mytextfile.txt
QmYFVyeUjycJ5pFPFqqebhiWP7TZQxdxw86SgMAVZGnQnm 91   new/
```

### 第四步：使用目录 hash 来读取目录下的文件内容

使用`ipfs cat`命令，而后边跟的是文件路径，操作如下：

```go
$ ipfs cat QmNmGoyiFpaHx1tra4zNL2iZHAfj8JM7gETjk2hZWVGWXi/ipfs-tutorial/new/newfile
QQ 群：348924182；263270946
```

如果你不想保留文件夹的名称，那么在上传文件时，不需要使用标签`-w`。那么检索文件时，操作如下：

```go
$ ipfs cat QmNpRuzmmucd5sDoC7fjfZRgo2mnaXb35hpsLQf77A96xu/new/newfile
QQ 群：348924182；263270946
```

### 注解

关于标签`-w`，大家可以根据自己的喜好使用。现在大部分教程都没有提这个标签，而仅仅是使用`ipfs add -r`。