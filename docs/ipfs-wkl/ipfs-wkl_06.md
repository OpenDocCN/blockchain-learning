# 第六章 IPFS Weekly 6

# IPFS Weekly 6

作者：Richard Littauer，Andrew Chin，2016-03-09

IPFS（[`ipfs.io/`](https://ipfs.io/) ） 是一种新的超媒体分发协议，通过内容和身份进行寻址，旨在使网络更快，更安全，更开放。在这些帖子中，我们将尽量强调过去一周发生的一些发展。对于任何想要参与的人，请打开本文嵌入的超链接，在 github（[`github.com/ipfs`](https://github.com/ipfs) ） 上搜索足够的信息或在 IRC（[`webchat.freenode.net/?channels=ipfs`](https://webchat.freenode.net/?channels=ipfs) ）上加入我们。

如果您希望将此更新作为电子邮件发送，请注册我们的每周简报（[`tinyletter.com/ipfsweekly`](https://tinyletter.com/ipfsweekly) ）！

以下是三月份第一周的一些亮点：

## 更新

### station([`github.com/ipfs-shipyard/ipfs-desktop`](https://github.com/ipfs-shipyard/ipfs-desktop) )

![](img/b33dcf0647fd22219037959df1a18b0d.jpg)

新版本的`station`已准备好供开发人员预览！`station`是在您的计算机上运行 IPFS 守护程序的最简单方法之一。它充当服务，您可以获得许多方便的功能，例如通过 GUI 打开 IPFS 节点和通过 IPFS 共享你的文件。要使用它，您需要安装 Node.js 4（此处为安装说明 [`nodejs.org/en/`](https://nodejs.org/en/) ）和 npm 3（随 Node 附带）。然后，执行以下操作：

```go
> git clone https://github.com/ipfs/station.git && cd station
> npm install
> npm start 
```

### website（网站 [`github.com/ipfs/website`](https://github.com/ipfs/website) ）

网站上的 API 命令列表已更新 ([`docs.ipfs.io/reference/api/cli/`](https://docs.ipfs.io/reference/api/cli/) )。这提供了一个单独的位置来同时查看 go-ipfs 的所有 CLI 命令; 如果您不确定接下来要使用哪个命令，并且知道的 ipfs commands 比较零散，那么这是一个很好的参考点。

### AUR （[`aur.archlinux.org/`](https://aur.archlinux.org/) ）

@Kubuxu 确保了可以在 AUR 上获取 gx，gx-go 和 ipget。AUR 是一个 arch-linux 的用户仓库。这意味着在某些 Linux 平台上获取和安装它们会更容易。

### karma-peer （[`github.com/dignifiedquire/karma-peer`](https://github.com/dignifiedquire/karma-peer) ）

@dignifiedquire 专注于 karma-peer 的工作。karma-peer 现在能够动态启动浏览器，这将有助于@diasdavid（希望更多人！）为 P2P 浏览器应用程序编写更好的测试。请参阅此处的一些测试示例（[`github.com/dignifiedquire/karma-peer/blob/master/test/index.spec.js`](https://github.com/dignifiedquire/karma-peer/blob/master/test/index.spec.js) ）。您还可以阅读针对此模块的讨论和用于测试 P2P 浏览器应用程序的原始工具。

### randor

@dignifiedquire 一直在研究 randor，这是一个测试框架，能够将大量文件和大量请求发送到 IPFS，以测试它如何适用于边缘情况以及它如何扩展。Randor 现在能够基于存储的数据进行可预见的回归测试，因此很容易找到并修复 bug。@whyrusleeping 已经开始着手处理 randor 检测到的第一个 bug。

### WebRTC 资源管理器（[`github.com/daviddias/webrtc-explorer`](https://github.com/daviddias/webrtc-explorer) ）

WebRTC Explorer 2.0.0 已经发布了 alpha 版！WebRTC Explorer 是一个 P2P 路由覆盖网络，使用 WebRTC 数据通道作为节点之间的传输层。WebRTC Explorer 支持浏览器之间的通信，无需调解器（服务器），使用户只需使用 Web 技术即可在机器之间传输数据包。WebRTC Explorer 的灵感来自`Chord DHT`，即创建一个带指纹列表的路由方案，其中节点之间是保持均衡的。@diasdavid 录制了详解视频（[`www.youtube.com/watch?v=fNQGGGE__zI&feature=youtu.be`](https://www.youtube.com/watch?v=fNQGGGE__zI&feature=youtu.be) ），大家可以链接查看。

### libp2p （[`github.com/libp2p/js-libp2p`](https://github.com/libp2p/js-libp2p) ）

@diasdavid 发布了两个用于流复用的模块版本：libp2p-spdy 和 libp2p-multiplex。还有，libp2p-swarm 加了一个新的 API，和更多的测试用例。大家可以链接[`github.com/libp2p/js-libp2p-switch/pull/20`](https://github.com/libp2p/js-libp2p-switch/pull/20) 查阅内容的变化。

### js-ipfs （[`github.com/ipfs/js-ipfs`](https://github.com/ipfs/js-ipfs) ）

有一些人一直在询问如何为 IPFS 的 JavaScript 实现做出贡献：好吧，不要再等了！现在，您可以阅读最新的 captain.log 条目（[`github.com/ipfs/js-ipfs/issues/30#issuecomment-187950929`](https://github.com/ipfs/js-ipfs/issues/30#issuecomment-187950929) ），了解项目的状态，并列出您可以贡献的事项。我们感谢您的帮助。

### ipfs-pad

我们希望在 IPFS 之上构建类似 Etherpad 的产品。要做到这一点需要很多的技术支持：我们如何知道如何对编辑进行排序？我们如何处理高延迟（几天/几周）或同时编辑？数据是如何传输的？@noffle 一直在推动构建这个过程，而且本周制作了许多模块来解决它的一些依赖：

*   bisecting-numbers - 类似整数的数字系统，其中任何数都可以被平分以形成无限的整数子系统。
*   bisecting-between - 生成一个给两个其他给定值之间进行排序的唯一值。
*   hyperswarm -创建围绕一个 hyperlog 形成的 P2PWebRTC 集群。

### pubsub

@noffle 还发布了一些早期的疯狂科学模块，这些模块支持流言消息在点对点覆盖网络中进行传播。这些并不是为 IPFS 严格构建的，而是一个实验性的垫片，使像@haad 的 orbit-db 这样的项目，能够在没有中央服务器的情况下运行，以便在对等体之间进行消息交换。

*   secure-gossip - 一个安全、传输不可知、流言的协议。网络中的任何对等体都可以发布消息，这些消息最终将通过每个节点的对等体之间的流言传播到整个网络。
*   pubsub-swarm - 使用传统的 pubsub API，在主题周围形成 p2p 集群，并且实现节点并交换消息。

### go-ipfs（[`github.com/ipfs/go-ipfs`](https://github.com/ipfs/go-ipfs) ）

@whyrusleeping 成立了 Teamcity （[`www.jetbrains.com/teamcity/`](https://www.jetbrains.com/teamcity/) ）。这减少了 Travis 运行的漫长等待，并且有望实现更快的 CI 测试。Teamcity 还为我们的测试提供了很棒的指标，并为故障和失败率提供了很好的统计数据。Teamcity 与大量的测试者很好地融合，从 go 测试到 karma 和 sharness。它将为我们提供有关我们测试运行的更详细的反馈。

### FC00

![](img/e24145bc022241af301f1737a87eecb3.jpg)

@lgierth 在巴黎度过了一个富有成效的一周，并和@xwiki 上与@cjdelisle 以及@ansuz 讨论了关于`cjdns / fc00`的状态和未来，提出了路由改进的想法，并为交换机和 cryptoauth 层起草了规范文档。你可以在这里([`github.com/fc00/spec/pulls`](https://github.com/fc00/spec/pulls) )找到这些规格（它们将很快更新）。3 月剩余时间将继续开展工作。fc00 的交换机和路由层可能是 IPFS / libp2p 智能群的基础，所以这一切都非常令人兴奋。

## 社区

### name-your-contributors （[`github.com/mntnr/name-your-contributors）`](https://github.com/mntnr/name-your-contributors）)

@RichardLitt 周四在 BostonJS 上向大约五十人发表了关于如何使用 name-your-contributors 的演讲。聚会的主题是社区建设。

### dignifed hacks（编码直播）

@dignifiedquire 启动了他编码的直播，他称之为“dignifed hacks”。上周一他记录了自己在第一集中为 WebUI 做了一个新功能。他本周会再做一次。其中一位观众@nginnever 表示，“有助于快速查看我们在 webui 中的组件和数据流。”他将在 Twitter 上宣布定期放映时间，您可以在 YouTube 上订阅 IPFS，[`www.youtube.com/channel/UCdjsUXJ3QawK4O5L1kqqsew`](https://www.youtube.com/channel/UCdjsUXJ3QawK4O5L1kqqsew) 。

### 里斯本区块链研讨会

@diasdavid 参加了由 Kwamecorp 主办的 3 月 5 日里斯本区块链研讨会。研讨会聚集了许多 Blockchain，IPFS，ethereum 和 zerocash 爱好者，他们在工作中将使用这些分布式技术来解决部署问题。

### IPFS Dead drop （[`github.com/c-base/ipfs-deaddrop`](https://github.com/c-base/ipfs-deaddrop) ）

![](img/7de6b4516bed77de1109d57871d66d0e.jpg)

一些 c-base 组织的成员编写了一个类似死机的系统，可以自动将文件从 U 盘上传到 IPFS。将 U 盘插入设备时，它将自动访问 U 盘并将文件发布到 Web 上。感谢 IPFS，这些文件可立即供全世界使用。查看[`deaddrops.com/`](https://deaddrops.com/) 了解更多信息。

使用说明：节点在上述设备中运行。当你插上电源时，打开（[`www.flickr.com/photos/bergie/25018249550/in/datetaken-public/`](https://www.flickr.com/photos/bergie/25018249550/in/datetaken-public/) ）你会看到信息。如果您在柏林或即将访问，请务必前往。

### 星际回路（IPWB）

由 `Archives Unleashed`和`ODU WSDL Research Group`两家组织开发的一个基于 IPFS 的分布式和文件永久存储可提取的系统，该系统叫星际回路（Interplanetary Wayback）。链接查看 [`github.com/oduwsdl/ipwb`](https://github.com/oduwsdl/ipwb)

### 科学数据

@jbenet 访问了 Janelia 研究园区，了解科学数据的工具和用例。他谈到了 IPFS，数据版本控制，包管理等等（即将发布的视频）。他了解了 DVID，以及惊人的 FlyEM 脑成像工作的要求。他看到了 FlyEM Team（github），Freeman Lab（github）和其他团队制作的许多出色的开源研究工具。去看看他们的 GitHub 回购，并帮助他们改善大脑研究！

### Press

Sitepoint 的杰夫史密斯发表了一篇关于 IPFS 的精彩文章：“HTTP 与 IPFS：点对点分享网络的未来？”。[`www.sitepoint.com/http-vs-ipfs-is-peer-to-peer-sharing-the-future-of-the-web/`](https://www.sitepoint.com/http-vs-ipfs-is-peer-to-peer-sharing-the-future-of-the-web/)

## 贡献者

在整个 IPFS GitHub 组织中，以下人员在 2 月 29 日（中午，GMT）和 3 月 7 日之间就 GitHub 提交了代码，提出问题或发表评论。我们使用此工具和其他工具自动生成此列表，因此，如果您的姓名不在此处，请告知我们。

@adrian-bl (Adrian Ulrich)
@amstocker (Andrew Stocker)
@anarcat (anarcat)
@Beligertint
@bergie (Henri Bergius)
@cevin (cevin)
@chriscool (Christian Couder)
@cinderblock (Cameron Tacklind)
@clkao (Chia-liang Kao)
@daveajones (Dave Jones)
@davidar (David A Roberts)
@diasdavid (David Dias)
@dignifiedquire (Friedel Ziegelmayer)
@greenkeeperio-bot (Greenkeeper)
@hjoest (Holger Joest)
@jbenet (Juan Benet)
@jedahan (Jonathan Dahan)
@knocte (Andres G. Aragoneses)
@Kolomona (Kolomona Myer)
@Kubuxu (Jakub Sztandera)
@lgierth (Lars Gierth)
@mappum (ᴍᴀᴛᴛ ʙᴇʟʟ)
@mattseh
@MichaelMure (Michael Muré)
@micxjo (Micxjo Funkcio)
@mildred (Mildred Ki’Lya)
@moritz121
@NeoTeo (Teo Sartori)
@Neurone (Giuseppe Bertone)
@nginnever (Nathan Ginnever)
@noffle (Stephen Whitmore)
@peteygao (Peter Gao)
@randomshinichi
@RichardLitt (Richard Littauer)
@richardschneider (Richard Schneider)
@sivachandran (Sivachandran)
@suisha (David Mai)
@thelinuxkid (Andres Buritica)
@tinybike (Jack Peterson)
@whyrusleeping (Jeromy Johnson)
@xicombd (Francisco Baio Dias)
@yangwao (Matej Nemček)
@yncyrydybyl (Yan Minagawa)
@zignig (Simon Kirkby)
@Zogg

谢谢，下周见！