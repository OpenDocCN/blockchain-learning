# 第六章 【IPFS 一问一答】IPFS 的版本控制系统是怎样的？

# 6 IPFS 的版本控制系统是怎样的？

IPFS 的文件版本控制系统是基于 Git 创建的。我们先分析下 Git。

## 6.1 IPFS 之技术背景 Git 版本控制系统

参考视频 [`ihower.tw/git/index.html`](https://ihower.tw/git/index.html)

git 学习的概念：

1.  工作区：当前你编辑文件的区域就是工作区，比如你的目录是./gitworkshop，那么这个目录下的内容就是你的工作区。
2.  版本库：记录了你工程某次提交的完整状态和内容，这意味着你的数据永远不会丢失。
3.  暂存区：暂存区在版本库中，git add 之后的数据会先存储到这里，等 git commit 之后，数据移到分支下，并清空暂存区。
4.  git 有四大物件，分别是 blob，tree，commit 和 refs（或者叫 tag）。 4.1 blob blob 是对文件内容的保存。当 git add 某个文件时，数据会保存到版本库中的 objects 中，如下图中的 61，就是我刚添加的文件之后产生的 blob。 ![](img/8bf85c4f255e986813e5d8ada2d772e6.jpg)

4.2 tree git 的数据存储都是以 tree 的形式存储的。如下图： ![](img/15c34ed5bc621cfd0c34177f6543c9fd.jpg) 4.3 commit 当执行 commit 后，暂存区的内容会被提交到 git 版本库，形成最新版本，并进行快照。如下图，最终 stage 中的数据被提交到 master 分支下。 ![](img/3ad2a98a9aea35e25065e20b9aa7ea78.jpg)

4.4 refs refs 是 References 的缩写。引用是变量名（名字），值指向 commit 对象，方便追踪代码的变化。如下图，每个版本都有自己的根 hash，refs 也就是 HEAD 指向 master 中的根 hash 所对应的内容。 ![](img/d8e8f13dbea8207cac753c255804039a.jpg)

git 只会对内容有变化的数据进行再存储，未改变的数据不作处理。如下图 ![](img/ad30aaba0f07eaa9bc36647419099d3f.jpg) 分析上图：文件总共包含 3 个，其中 my_dir 下的 my_file.txt 和 hello.txt 中的内容一样，所以只存储一份。接下来我们修改了 my_dir 文件夹下的 my_file.txt 文件。由于内容发生了变化，当 commit 后，就会产生新的根 hash，也就是新的版本内容。

## 6.2 IPFS 版本控制系统

IPFS 版本控制几乎和 git 一样。下图是它的版本库内容。 ![](img/33634f9e96ce1aa8230f6126ace236d4.jpg)

我们在执行 ipfs init 时会产生这个./ipfs 的隐藏文件，里面的内容就是上图显示的。当添加一个全新的文件内容时，datastore 中的有些内容将会发生变化，如版本，ldb，这里主要是树结构的存储发生了变化，除此，新的数据内容将会以碎片的形式被存储在 blocks 中。在 blocks 中的数据是没有重复的。

每个 block 数据对应一个唯一的 hash 值。

一个目录文件，可能包含多个文件和其他目录文件，被提交后，返回很多 hash 值，其中最后那个是整个目录文件的 hash 值，也就是根 hash，这个根 hash 下有树 hash 和叶子 hash。

ipfs 修改数据或检索数据都和 git 一样。