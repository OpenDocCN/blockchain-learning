# 第三章 IPFS Weekly 3

# IPFS Weekly 3

作者：Richard Littauer，Andrew Chin，2016-02-01

IPFS（[`ipfs.io/）`](https://ipfs.io/）) 是一种新的超媒体分发协议，通过内容和身份进行寻址，旨在使网络更快，更安全，更开放。在这些帖子中，我们将尽量强调过去一周发生的一些发展。对于任何想要参与的人，请打开本文嵌入的超链接，在 github（[`github.com/ipfs）`](https://github.com/ipfs）) 上搜索足够的信息或在 IRC（[`webchat.freenode.net/?channels=ipfs）`](https://webchat.freenode.net/?channels=ipfs）) 上加入我们。

如果您希望将此更新作为电子邮件发送，请注册我们的每周简报[`tinyletter.com/ipfsweekly！`](https://tinyletter.com/ipfsweekly！)

这是双周刊：以下是 1 月 12 日和 1 月 19 日内容的一些亮点：

## 更新

*   dist.ipfs.io 分发页面系统已经推出！这项工作主要是由@dignifiedquire 带头完成的。分发页面系统是查找和下载 IPFS 生成的所有官方二进制文件的新一站式商城。每个项目都有：
    *   分发名称和简短描述;
    *   当前版本号和发布日期;
    *   一个下载按钮，可以检测您的平台并自动为您建议适当的分发;
    *   包含所有支持平台（操作系统和体系结构）下载链接的网格;
    *   A Changelog，指向所有版本更改摘要的链接;
    *   An All Versions，查看和下载以前版本的链接。
        该站点也托管在 IPFS 上，用于`ipfs-update`更新 IPFS。在[`dist.ipfs.io`](http://dist.ipfs.io) 上查看。
*   go-ipfs 0.3.11 版已经推出！而且，我们将 0.4.0 合并为 master。@jbenet，@ Dignifiedquire 和其他人负责`fixed appveyor`。最后@Dignifiedquire 的最新 webui 被推到了 0.3.11。查看更改日志。
*   js-ipfs-merkle-dag @diasdavid 致力于 merkle-dag 实现的互操作性，使您能够通过 go-ipfs 在 JavaScript 中成功读取（和写入）存储在 IPFS Repo 中的 MerkleDAG 节点。
*   py-ipfs 感谢@candeira 和@ivilata，关于 IPFS Python 实现，我们现在有一个非常详细的计划。如果你是一名 python 工程师，请加入我们。
*   JS-mafmt @whyrusleeping 正在 js 方面对 multiaddr 做验证工作，关于 mafmt 的 Go 工作已经完成。
*   infrastructure ipfs.io 公共网关现在同时由 v0.4.0-dev 和 v0.3.11 支持。两者都会代理请求，第一个成功的响应获胜。ipfs.io 最终将由 v0.4.x 支持。这意味着如果您要在一个特定版本上定位请求，请分别使用 v04x.ipfs.io 或 v03x.ipfs.io。当公共网关不支持 v03x 时，后者将停止返回 DNS 记录。此外，在我们发布 v0.4.0 之后的一段时间内，默认的引导程序节点在 v03x 和 v04x 之间进行了划分。大多数可以由 v04x 节点使用，少数可以由 v03x 节点使用。随着时间的推移，后者将被淘汰。
*   ipfs-firefox-addon 通过本地 HTTP2IPFS 网关提供对 IPFS 资源的透明访问的 Firefox 插件已更新至 1.4.2 版。这是我们第一次提到这一点; 去看看吧。插件使您可以通过 IPFS 而不是 HTTP 加载内容。例如，如果您从公共网关打开资源（例如[`ipfs.io/ipfs/QmW2WQi7j6c7UgJTarActp7tDNikE4B2qXtFCfLPdsgaTQ/cat.jpg`](https://ipfs.io/ipfs/QmW2WQi7j6c7UgJTarActp7tDNikE4B2qXtFCfLPdsgaTQ/cat.jpg) ） 并启用了插件，请求将不会访问公共网关，但数据将从 IPFS 群，以分散和分布的方式。它现在也支持 IPFS 协议方案，这意味着可以通过 fs:/ipfs/<hash>直接查找哈希。</hash>
*   go-multiaddr go-multiaddr 现在支持动态添加新协议。这允许我们构建新协议，而不必强迫每个人更新 multiaddr 库本身。
*   community 每个 IPFS 存储库都添加了标签'help wanted'和'difficulty：easy'，'：hard'和'：moderate'，以帮助贡献者知道那些最容易得手的工作。同样，@ RichardLitt 创建了一个文档样式指南，以帮助标准化我们在所有工具中使用英语的方式。它会随着时间的推移而增长。
*   ipfs-dht @whyrusleeping 写了一个'just a dht'二进制文件。这意味着您可以在不运行完整 ipfs 节点的情况下帮助网络。而且它只需要更少的内存和带宽，但拥有更多 dht 节点，实现加快查找。
*   js-ipfsd-ctl 能够控制 IPFS 守护程序的 Node 模块已更新为 0.3.11。
*   webui webui 不再依赖于 jQuery，感谢@luigiplr，@ travisperson 和@dignifiedquire 的大量工作。

## 工作正在进行中

*   js-ipfs @diasdavid 已将 js-ipfs 和 libp2p 合并到路线图中，他将不断更新细节内容，变得整体变得清晰。可以关注一下。
*   weekly @RichardLitt 为 IPFS 的贡献者提供了大量的工作，目前`name-your-contributors`还可用。这几乎已经完成，但还有一些调整要做。

## 聚会和会议

*   ArcticJS @ diasdavid， @ whyrusleeping， @ RichardLitt 和 @noffle 能够一起第一次作为一个小组参加聚会，并在斯瓦尔巴群岛的北极圈上举办的第一次 JavaScript 会议，即 ArcticJS 会议，一起参与讨论。大约 15 人参加。那里有很多的砌砖和雪。他们谈论缓冲技术，并在午夜时分在海洋中游泳（是，真的）。会议大部分内容都是对话，但也包括 @diasdavid 和 @RichardLitt 一起研究 API。@whyrusleeping 学习如何使用 javascript，以及 @diasdavid 和 @xicombd 一起致力于 IPFS。总而言之，这是一次令人难以置信的旅行。

## Shoutouts

*   Erlang 的发明者乔·阿姆斯特朗（Joe Armstrong）在一篇关于组织软件的最新一系列帖子中提到了持久性和 IPFS 的重要性。谢谢你的好意，乔！

## 贡献者

在整个 IPFS GitHub 组织中，以下人员已于 1 月 11 日至 1 月 25 日期间在 GitHub 上提交了代码，创建问题或发表评论。（我们正在使用此工具自动生成此列表，因此，如果您的名字不是，请告诉我们那里。）

*   @adrian-bl (Adrian Ulrich)
*   @alexeicolin
*   @ali (Ali Ukani)
*   @amstocker (Andrew Stocker)
*   @anarcat (anarcat)
*   @ansuz (ansuz)
*   @Ape (Lauri Niskanen)
*   @area
*   @AtnNn (Etienne Laurin)
*   @Azulan (Frank Flores)
*   @Balancer (Balancer)
*   @bcg-didi
*   @benjaminbollen (Benjamin Bollen)
*   @brailateo (Constantin Teodorescu)
*   @btrask (Ben Trask)
*   @CaioAlonso (Caio Alonso)
*   @chpio
*   @chriscool (Christian Couder)
*   @ConsciousCode (Conscious Code)
*   @CrowdHailer (Peter Saxton)
*   @dandart (Dan Dart)
*   @david415 (David Stainton)
*   @davidar (David A Roberts)
*   @diasdavid (David Dias)
*   @digital-dreamer
*   @dignifiedquire (Friedel Ziegelmayer)
*   @djdv (Dominic Della Valle)
*   @dylanPowers (Dylan Powers)
*   @eminence (Andrew Chin)
*   @Faleidel
*   @fazo96 (Enrico Fasoli)
*   @findkiko
*   @Foxcool (Alexander Babenko)
*   @GitCop
*   @goog (Jay Cheng)
*   @greenkeeperio-bot (Greenkeeper)
*   @harlantwood (Harlan T Wood)
*   @ikreymer (Ilya Kreymer)
*   @jbenet (Juan Benet)
*   @jbshirk (Joe)
*   @jedahan (Jonathan Dahan)
*   @Kubuxu (Jakub Sztandera)
*   @kyledrake (Kyle Drake)
*   @lgierth (Lars Gierth)
*   @lidel (Marcin Rataj)
*   @lovelaced
*   @luigiplr (Luigi Poole)
*   @Luzifer (Knut Ahlers)
*   @matshenricson (Mats Henricson)
*   @MChabez
*   @Mec-iS (Lorenzo)
*   @MichaelMure (Michael Muré)
*   @mildred (Mildred Ki’Lya)
*   @Mithgol
*   @NDuma (NDuma)
*   @NeoTeo (Teo Sartori)
*   @nginnever (Nathan Ginnever)
*   @nicola (Nicola Greco)
*   @NightRa (Ilan Godik)
*   @noffle (Stephen Whitmore)
*   @Patagonicus (Philipp Adolf)
*   @peerchemist
*   @peteygao (Peter Gao)
*   @PlanetPlan
*   @pra85 (Prayag Verma)
*   @prusnak (Pavol Rusnak)
*   @ralphtheninja (Lars-Magnus Skog)
*   @ReadmeCritic
*   @Red5d
*   @reit-c
*   @rht
*   @RichardLitt (Richard Littauer)
*   @richardschneider (Richard Schneider)
*   @rugk (rugk)
*   @SCBuergel (Sebastian C. Bürgel)
*   @seclorum (seclorum)
*   @thelinuxkid (Andres Buritica)Thomas Gardner
*   @travisperson (Travis Person)
*   @tv42 (Tommi Virtanen)
*   @void4
*   @WeMeetAgain (Cayman)
*   @whyrusleeping (Jeromy Johnson)
*   @windemut
*   @wking (W. Trevor King)
*   @xicombd (Francisco Baio Dias)
*   @zignig (Simon Kirkby)

谢谢，下周见！如果你有下周的好东西可以分享，请在下一周的 sprint 问题上给我们留言！
发送网址：[`github.com/ipfs/newsletter/issues/7`](https://github.com/ipfs/newsletter/issues/7)