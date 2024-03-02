# 第八章 IPFS Weekly 8

# IPFS Weekly 8

IPFS（[`ipfs.io/`](https://ipfs.io/) ） 是一种新的超媒体分发协议，通过内容和身份进行寻址，旨在使网络更快，更安全，更开放。在这些帖子中，我们将尽量强调过去一周发生的一些发展。对于任何想要参与的人，请打开本文嵌入的超链接，在 github（[`github.com/ipfs`](https://github.com/ipfs) ） 上搜索足够的信息或在 IRC（[`webchat.freenode.net/?channels=ipfs`](https://webchat.freenode.net/?channels=ipfs) ）上加入我们。

如果您希望将此更新作为电子邮件发送，请注册我们的每周简报（[`tinyletter.com/ipfsweekly`](https://tinyletter.com/ipfsweekly) ）！

以下是 3 月 14 日的一些重点内容：

## 更新

![](img/4dfff10065f9c082de050615ed35c9fa.jpg)

### WebUI ([`github.com/ipfs-shipyard/ipfs-webui`](https://github.com/ipfs-shipyard/ipfs-webui) )

@dignifiedquire 一直在努力开发新的 webui，而且即将推出。您将能够预览图像，观看视频，创建文件夹，拖放等。附图是一个漂亮的 GIF。获取更多标签信息，请链接[`github.com/ipfs-shipyard/ipfs-webui/issues?q=is%3Aopen+is%3Aissue+label%3A%22help+wanted%22`](https://github.com/ipfs-shipyard/ipfs-webui/issues?q=is%3Aopen+is%3Aissue+label%3A%22help+wanted%22) 查询。

### PubSub

上周在露营期间，大家通过视频聊天，在 PubSub 上进行了一些积极的讨论。PubSub 是我们用来谈论一个简单协议的名称，它有助于在 IPFS 之上轻松发布和订阅。我们的要求是它必须易于实现，分层良好，并与其他 IPFS 抽象很好地融合。你如果想加入有关 PubSub API 的讨论，请关注链接中的主题（[`github.com/ipfs/notes/issues/118`](https://github.com/ipfs/notes/issues/118) ）。有关更多讨论，请查看[`github.com/ipfs/notes/issues?utf8=%E2%9C%93&q=is%3Aissue+is%3Aopen+pubsub`](https://github.com/ipfs/notes/issues?utf8=%E2%9C%93&q=is%3Aissue+is%3Aopen+pubsub) 。

### ipfs-log （[`github.com/orbitdb/ipfs-log`](https://github.com/orbitdb/ipfs-log) ）

@haadcode 一直在研究和发布 ipfs-log，这是一个部分有序的 IPFS 哈希链表。日志中的每个条目都指向所有已知的头或叶节点。在需要追踪“动态内容”的应用程序中，如跟踪文件的版本、创建 IPFS 哈希的消息、消息传递或作为 CRDT 的传输，`ipfs-log`可以被用作创建中的区块。它最初是为`orbit-db`创建的，目前用于 IPFS 上的 KV 存储和事件监听。

### ipfs init

针对`js-ipfs`的`ipfs init`工作几乎已经完成，这要归功于@noffle。这将使`go-ipfs`具有兼容性，但前提是使用`JavaScript`运行。如果你喜欢构建测试和断点测试，那么这里有很多机会在`js-ipfs`上让你做出贡献([`github.com/ipfs/js-ipfs#contribute`](https://github.com/ipfs/js-ipfs#contribute) )。

### go-ipfs （[`github.com/ipfs/go-ipfs`](https://github.com/ipfs/go-ipfs) ）

本周关于`go-ipfs`，我们为 0.4.0 的最新版本做了很多准备工作。包括大量的测试，信息编写以及 IPFS 不同方面的验证。@whyrusleeping 为`fs-repo-migrations`编写了一个压力测试，它添加了大量对象（超过 200000），并锁定其中的几千个对象，进行迁移，验证所有内容，运行 gc，然后再次验证所有内容。一旦测试通过，他开始测试碰撞因数为 10 的试运行（超过 200 万个对象！），一切都很好。这种稳健性意味着 0.4.0 将很快就能发布。

### ipfs-firefox-addon （[`github.com/ipfs-shipyard/ipfs-companion`](https://github.com/ipfs-shipyard/ipfs-companion) ）

`ipfs-firefox-addon`之前在第 3 期（v1.4.2）中提到过，我们做了些改变。
`Firefox 插件`是通过本地`HTTP2IPFS`网关提供对 IPFS 资源的透明访问。该插件已经通过了`Mozilla`的全面审查，并更新到 v1.5.6([`addons.mozilla.org/en-US/firefox/addon/ipfs-gateway-redirect/versions/1.5.6`](https://addons.mozilla.org/en-US/firefox/addon/ipfs-gateway-redirect/versions/1.5.6) )。平均每天有超过 350 名用户。

1.5.x 系列带来了各种 UX 的改进，例如实时状态和诊断，以及可在“首选项”屏幕上启用实验性功能。大家可以在 Github 上查看完整列表（[`docs.ipfs.io/guides/concepts/dnslink/`](https://docs.ipfs.io/guides/concepts/dnslink/) ）。欢迎提供功能请求和 bug（[`github.com/ipfs-shipyard/ipfs-companion/issues`](https://github.com/ipfs-shipyard/ipfs-companion/issues) ）！

## Community

### Mediachain（[`blog.mediachain.io/mediachain-483f49cbe37a#.50am8s6cw`](https://blog.mediachain.io/mediachain-483f49cbe37a#.50am8s6cw) ）

在 Mine 实验室（[`www.mediachainlabs.com/`](http://www.mediachainlabs.com/) ）的朋友最近发布了一个`L-SPACE`（[`github.com/mediachain/L-SPACE`](https://github.com/mediachain/L-SPACE) ）的 Mediachain 服务器。他们还为他们的博客写了很多更新： Mediachain 如何运作（[`blog.mediachain.io/how-mediachain-works-5a5ccc1c3210`](https://blog.mediachain.io/how-mediachain-works-5a5ccc1c3210) ）， `Dev Update V`([`blog.mediachain.io/mediachain-developer-update-v-a7f6006ad953`](https://blog.mediachain.io/mediachain-developer-update-v-a7f6006ad953) )， Dev Update VI([`blog.mediachain.io/mediachain-developer-update-vi-94d28cf6bc30`](https://blog.mediachain.io/mediachain-developer-update-vi-94d28cf6bc30) )等等。Mediachain 被媒体密切关注，已经在纳斯达克，比特币杂志，CCN 等杂志上亮相 ！恭喜，感谢那里的 IPFS 呐喊！

### 里斯本

@diasdavid 在里斯本组织了一次 IPFS 研发会议。如果您在该地区，请加入此聚会小组（[`www.meetup.com/lisbon-ipfs/events/229530492/`](https://www.meetup.com/lisbon-ipfs/events/229530492/) ）。

### 新闻界

比特币杂志有来自 Eris Industries 的 Zach Ramsay 的客座文章，关于 Blockchains 如何进一步推动公共科学（[`www.nasdaq.com/article/how-blockchains-can-further-public-science-cm592775`](https://www.nasdaq.com/article/how-blockchains-can-further-public-science-cm592775) ）。Zach 还在 Eris 博客上发表了第二部分：公共科学：一个稍微更实用的指南。两者都非常值得一读，特别是如果你在学术界。

## Contributors

在整个 IPFS GitHub 组织中，以下人员在 3 月 14 日（中午，GMT）和 3 月 21 日之间在 GitHub 提交了代码，提出问题或发表评论。以下是贡献人员列表因此，如果您的姓名不在此处，请告知我们。

@amstocker (Andrew Stocker)
@anarcat (anarcat)
@ansuz (ansuz)
@basile-henry (Basile Henry)
@bdunlay (Brian Dunlay)
@brainframe-me (Cox Davy)
@candeira (Javier Candeira)
@chpio
@chriscool (Christian Couder)
@christianlundkvist (Christian Lundkvist)
@Cleric-K
@clkao (Chia-liang Kao)
@davidar (David A Roberts)
@diasdavid (David Dias)
@dignifiedquire (Friedel Ziegelmayer)
@doesntgolf (Nate Dobbins)
@ehd (Stephan Seidt)
@eminence (Andrew Chin)
@fazo96 (Enrico Fasoli)
@GoogilyBoogily (Derek Mayer)
@greenkeeperio-bot (Greenkeeper)
@haadcode (Haad)
@harlantwood (Harlan T Wood)
@ianopolous (Ian Preston)
@ingokeck (Ingo Keck)
@jbenet (Juan Benet)
@jwsher (Justin Sher)
@kalmi (Tarnay Kálmán)
@kevina (Kevin Atkinson)
@klartext
@kseistrup (Klaus Alexander Seistrup)
@Kubuxu (Jakub Sztandera)
@lgierth (Lars Gierth)
@lhenocque
@lidel (Marcin Rataj)
@luigiplr (Luigi Poole)
@matshenricson (Mats Henricson)
@MaxEntropyy
@Mec-iS (Lorenzo)
@mildred (Mildred Ki’Lya)
@Mithgol
@mlbk0
@mnp (Mitchell Perilstein)
@montagsoup
@nginnever (Nathan Ginnever)
@nicola (Nicola Greco)
@noffle (Stephen Whitmore)
@palkeo (palkeo)
@RichardLitt (Richard Littauer)
@rinpoo
@sahib (Chris Pahl)
@se3000 (Steve Ellis)
@sexybiggetje (Martijn de Boer)
@sivachandran (Sivachandran)
@slothbag
@Stebalien (Steven Allen)
@thelinuxkid (Andres Buritica)
@thomas-gardner
@wanderer
@wasserfuhr (RainerWasserfuhr)
@whyrusleeping (Jeromy Johnson)
@willglynn (Will Glynn)
@wking (W. Trevor King)
@xicombd (Francisco Baio Dias)
这份通讯也是一项社区活动。如果您有下周的好东西要分享，请在下一周的 sprint 问题中发表评论（[`github.com/ipfs/newsletter/issues/31`](https://github.com/ipfs/newsletter/issues/31) ）！越多人提到他们想要在每周看到的项目，就越容易将其发送出去。

谢谢，下周见！