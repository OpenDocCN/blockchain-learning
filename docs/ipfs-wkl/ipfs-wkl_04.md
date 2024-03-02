# 第四章 IPFS Weekly 4

# IPFS Weekly 4

作者：Richard Littauer，Andrew Chin，2016-02-05

IPFS（[`ipfs.io/）`](https://ipfs.io/）) 是一种新的超媒体分发协议，通过内容和身份进行寻址，旨在使网络更快，更安全，更开放。在这些帖子中，我们将尽量强调过去一周发生的一些发展。对于任何想要参与的人，请打开本文嵌入的超链接，在 github（[`github.com/ipfs）`](https://github.com/ipfs）) 上搜索足够的信息或在 IRC（[`webchat.freenode.net/?channels=ipfs）`](https://webchat.freenode.net/?channels=ipfs）) 上加入我们。

以下是 1 月 25 日内容的一些亮点：

## 更新

*   js-ipfs： @diasdavid 和 @vijayee 创建了`js-ipfs-merkle-dag`和 `js-ipfs-blocks`，这将有助于 IPFS 处理`MerkleDAGs`，并且可扩展性足以允许其他人构建自己的 MerkleDAG 结构。此外，`js-ipfs`现在能够将文件导入到`MerkleDAG`，这是一个重要的里程碑。链接这里（[`github.com/ipfs/js-ipfs#ipfs-core-implementation-architecture`](https://github.com/ipfs/js-ipfs#ipfs-core-implementation-architecture) ）， 新手可以进入了解 js-ipfs 的工作原理。最后，许多新问题和简单完成地工作被标记为`贡献者解决内容`。 `idb-plus-blob-store`： `substack/idb-blob-store`有关于发布`finish`事件(因为在`streams`和方法 `.end`的问题导致的在`createWritableStream`出现的结果)的问题得到解决。所以@dignifiedquire 编写了这个模块修复了这个问题。现在我们可以使用 IndexedDB 作为存储去测试浏览器中 js-ipfs 的所有部分。
*   go-ipfs：感谢 @Kubuxu。`dnslink`通过专门的子域名（_dnslink.）进行了扩展。用户可以用命令（使用 CNAME）给域名起别名为 gateway.ipfs.io，同时仍然能够将`dnslink`设置为他们希望的内容。另外，@ whyrusleeping 有一个开放的`PR`，用来从 go-ipfs 中提取 libp2p，并将其作为一个模块放入到 gx（IPFS 本地包管理器）上。在将 go-ipfs 压缩成更小的可扩展性模块方便，它起到了重要的作用。最后， @whyrusleeping 对`go-ipfs`拉取的请求进行了清理。关闭了所有请求通道，或根据需要 ping 了请求的发送者，对其进行更新。
*   notes: @noffle 组织一个研讨，针对`ipfs mount`的接口更新问题。
*   community: @diasdavid 编辑了`guide to writing captain.logs`，内容主要是由社区维护者关于 IPFS 项目，状态，以及如何去帮助方面编写的。我们计划在不同的项目中增加这个板块。

## Community

*   里斯本市: @diasdavid 本周在里斯本发起了 IPFS 周日黑科技交流日。不少人前来参与了此次活动。其中@xicombd 制作了一个`Chrome Extension`，它允许您从本地 IPFS 守护进程（源代码）访问 IPFS url。
*   茶话会: @whyrusleeping 组织了另一个茶话会。内容是`gx`，一个 IPFS 包的控制中心。
*   NYC 见面会: 计划 2 月底举办一个见面会。感兴趣的可以联系我们，[`github.com/ipfs/community/issues/102`](https://github.com/ipfs/community/issues/102) 。
*   Desert Blockchain: 在最新一期的 Desert Blockchain 小型会议上，IPFS 概念出现在了会议议题上。

## 工具

*   GitHub 服务器存在宕机的可能性。所以很多人已经开始使用 IPFS 转移他们的数据仓库。利用@whyrusleeping 简单的`git-ipfs-rehost`（在 IPFS 上查看）和@cryptix 优秀`git-remote-ipfs`（在 IPFS 上查看）。
*   @pipermerriam 创建了`ipfs-persistence-consortium`，用来构建 IPFS 节点之间互相保护彼此内容的网络生态。这有点类似 @victorbjelkholm 的`pincoop`。或许他们之间可以有某些合作？😎
*   全节点项目 – 包括 IPFS、Tor、Bitcoin、OpenVPN 等。 感谢@MrChrisJ!

## 其它

*   2 月 2 日是 Ralph Merkle 的生日！他是 Merkle 树的创造者，该树是 IPFS 如何运作的关键部分。祝拉尔夫生日快乐！ 🎂
*   `Mine`（链接[`www.mediachainlabs.com/`](http://www.mediachainlabs.com/) 进入 ）在媒体链接协议上正在使用 IPFS，该协议是一种跟踪媒体创作，媒体源头等的协议。他们发表了几篇关于它的精彩文章。本周@denisnazarov 编写了`The GIF That Fell To Earth`；@parkan 递交了`Developer Update`，其内容是讨论 pHash，IPLD，等等。
*   @lexansoft 创建了一个以太坊命名的注册中心，名为 EtherID（repo here），它使用 IPFS 存储用户的内容。然后@btsfav 写了一篇关于 EtherID 与 IPFS 的个人网站的文章。

## 贡献者

在整个 IPFS GitHub 组织中，以下人员在 1 月 25 日（格林威治标准时间中午）和 2 月 1 日之间在 GitHub 上提交了代码、提出问题或发表评论。我们正在使用此工具自动生成此列表，因此，如果您的姓名不在此处，请告知我们。

*   @alexAubin (Alexandre Aubin)
*   @andreiamatuni (Andrei Amatuni)
*   @area
*   @AtnNn (Etienne Laurin)
*   @bdunlay (Brian Dunlay)
*   @BigBlueHat (BigBlueHat)
*   @chriscool (Christian Couder)
*   @ConsciousCode (Conscious Code)
*   @cryptix (Henry)
*   @davidar (David A Roberts)
*   @diasdavid (David Dias)
*   @dignifiedquire (Friedel Ziegelmayer)
*   @dysbulic (Will Holcomb)
*   @eminence (Andrew Chin)
*   @fazo96 (Enrico Fasoli)
*   @GitCop
*   @greenkeeperio-bot (Greenkeeper)
*   @harlantwood (Harlan T Wood)
*   @Hexagon6
*   @IanCal (Ian Calvert)
*   @ion1 (Johan Kiviniemi)
*   @JAremko
*   @jbenet (Juan Benet)
*   @jedahan (Jonathan Dahan)
*   @kazarena
*   @kpcyrd
*   @Kubuxu (Jakub Sztandera)
*   @lgierth (Lars Gierth)
*   @lidel (Marcin Rataj)
*   @lockedshadow
*   @Luzifer (Knut Ahlers)
*   @MartinThoma (Martin Thoma)
*   @MichaelMure (Michael Muré)
*   @mildred (Mildred Ki’Lya)
*   @mindhog
*   @Mithgol
*   @mortonfox (Morton Fox)
*   @MrChrisJ (Chris Ellis)
*   @NDuma (NDuma)
*   @NeoTeo (Teo Sartori)
*   @nikhilshekhawat
*   @noffle (Stephen Whitmore)
*   @palesz (Palesz)
*   @Patagonicus (Philipp Adolf)
*   @ralphbean (Ralph Bean)
*   @randomshinichi
*   @rht
*   @RichardLitt (Richard Littauer)
*   @Shaaah (Shaaah)
*   @sivachandran (Sivachandran)
*   @slothbag
*   @thelinuxkid (Andres Buritica)
*   @tilgovi (Randall Leeds)
*   @tommg (Thomas Gardner)
*   @VertigoRay (Raymond Piller)
*   @w33tmaricich (Alexander Maricich)
*   @whyrusleeping (Jeromy Johnson)
*   @whyun7892
*   @willeponken (William Wennerström)
*   @willglynn (Will Glynn)
*   @xicombd (Francisco Baio Dias)

谢谢，下周见！如果你有下周的好东西可以分享，请在下一周的 sprint 问题上给我们留言！
发送网址：[`github.com/ipfs/newsletter/issues/7`](https://github.com/ipfs/newsletter/issues/7)