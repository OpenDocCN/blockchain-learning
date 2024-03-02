# 第二章 IPFS Weekly 2

# IPFS Weekly 2

作者：Richard Littauer，Andrew Chin，2016-01-11

IPFS（[`ipfs.io/）`](https://ipfs.io/）) 是一种新的超媒体分发协议，通过内容和身份进行寻址，旨在使网络更快，更安全，更开放。在这些帖子中，我们将尽量强调过去一周发生的一些发展。对于任何想要参与的人，请打开本文嵌入的超链接，在 github（[`github.com/ipfs）`](https://github.com/ipfs）) 上搜索足够的信息或在 IRC（[`webchat.freenode.net/?channels=ipfs）`](https://webchat.freenode.net/?channels=ipfs）) 上加入我们。

以下是截止 1 月 5 日的一些亮点：

## 更新

*   (specs) 在`IPLD spec`中关于路径的表示方法存在很多争议。@mildred 特别做了很多有效工作，。
*   （go-ipfs）由于 @ChristianKniep， @ whyrusleeping 和 @chriscool，现在可以在 Travis 上自动测试 IPFS Docker 镜像。
*   （js-ipfs）现在你可以使用命令`npm i -g js-ipfs`，通过`bootstrap + id + version`命令来执行和使用 jsipfs（IPFS 的 javascript impl），与`go-ipfs repo`完全兼容，感谢 @diasdavid！
*   （project-repos）推出了一个非常酷的组织范围仪表板，可以全览所有启动起来的 IPFS 仓库的状态。 @harlantwood 创作。
*   （社区） @RichardLitt 合并了针对`IPFS repos`的`JavaScript guidelines`请求。这是迈向 IPFS 中标准`JavaScript`存储库的重要一步。
    -（社区） @ NeoTeo， @ whyrusleeping 和 @diasdavid 在哥本哈根举办了一次小型聚会！共有 8 人参加。包含 IPFS 简介，问答环节以及一些很好的对话！

## 工作正在进行中

*   (distributions) @Dignifiedquire 进一步开发了发行版页面。 点击这里预览[`v04x.ipfs.io/ipfs/QmZyvWokPYGg6DrjE6o2V7qhThzZQZ8QCWqdd2U3S75HXC/index.html！`](https://v04x.ipfs.io/ipfs/QmZyvWokPYGg6DrjE6o2V7qhThzZQZ8QCWqdd2U3S75HXC/index.html！)
*   （go-ipfs） @lgierth 继续对`dev040`迁移工作。值得注意的是，我们有两个新的网关来帮助过渡： `http： //v04x.ipfs.io`和 `http://v03x.ipfs.io`
*   (archives) @Dignifiedquire 已经向 IPFS 添加了大量 Stackexchange 的成果！详细信息在档案库中。我们一直在寻找关于存档重要数据集的更多帮助，所以请随时加入我们的档案仓库[`github.com/ipfs/archives/`](https://github.com/ipfs/archives/) ！

## 贡献者

在整个 IPFS GitHub 组织中，以下人员自 1 月 4 日起提交了代码。（我们使用此工具自动生成此列表，因此，如果您的姓名不在此，请告诉我们。）将来，我们还会包括评论的人，因为他们也非常重要。在我们开发该技术的同时，和我们共同努力。

*   @chriscool (Christian Couder)
*   @ChristianKniep (Christian Kniep)
*   @diasdavid (David Dias)
*   @Dignifiedquire (Friedel Ziegelmayer)
*   @eminence (Andrew Chin)
*   @greenkeeperio-bot
*   @ianopolous (Ian Preston)
*   @jbenet (Juan Benet)
*   @Kubuxu (Jakub Sztandera)
*   @lgierth (Lars Gierth)
*   @noffle (Stephen Whitmore)
*   @ralphtheninja (Lars-Magnus Skog)
*   @ReadmeCritic
*   @RichardLitt (Richard Littauer)
*   @whyrusleeping (Jeromy Johnson)
*   @yuvallanger (Yuval Langer)

谢谢，下周见！如果你有下周的好东西可以分享，请在下一周的 sprint 问题上给我们留言！
发送网址：[`github.com/ipfs/newsletter/issues/7`](https://github.com/ipfs/newsletter/issues/7)