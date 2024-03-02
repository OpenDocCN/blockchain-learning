# 第二十一章 IPFS Weekly 21

# IPFS Weekly 21

作者：Jenn Turner，2018-12-4
翻译：张恒兴 _ 孔壹学院|CHAINDESK
QQ 群：348924182；263270946

欢迎来到 IPFS 周刊。
行星际文件系统（IPFS）是一种新的超媒体分发协议，由内容和身份解决。IPFS（[`ipfs.io/`](https://ipfs.io/) ）支持创建完全分布式的应用程序。它旨在使网络更快，更安全，更开放。由于这是一个非常大的工程，我们将在每周期刊中跟踪整个生态系统的开发。

对于任何想要参与的人，请打开本文嵌入的超链接，在 github（[`github.com/ipfs`](https://github.com/ipfs) ） 上搜索足够的信息或在 IRC（[`webchat.freenode.net/?channels=ipfs`](https://webchat.freenode.net/?channels=ipfs) ）上加入我们。

以下是自上次 IPFS 周刊以来的一些亮点：

## 最新动态

*   我们的 HTTP 客户端库名字更改了！（详细内容查看[`blog.ipfs.io/58-http-client-rename/`](https://blog.ipfs.io/58-http-client-rename/) ）！最重要的消息是，HTTP 客户端库的名字将把`ipfs-api`改为`ipfs-http-client`。没错，它更长，但更好地描述模块是什么！阅读博客了解更多信息。
*   美食。看看@ momack28 度假烘焙冒险中的这些神奇的饼干。[`twitter.com/momack28/status/1069510119132516352`](https://twitter.com/momack28/status/1069510119132516352)
*   不要忘记`Sander Pick`将在下周一下午 5 点的 IPFS 每周会议上发表关于`Textile Photos` 的相关讨论。[`github.com/ipfs/team-mgmt#-ipfs-weekly-call--formerly-known-as-ipfs-all-hands-call`](https://github.com/ipfs/team-mgmt#-ipfs-weekly-call--formerly-known-as-ipfs-all-hands-call)

## 认识`Yurko`工程师

上周，我们提到了 IPFS 上的实时流媒体（[`github.com/tomeshnet/ipfs-live-streaming`](https://github.com/tomeshnet/ipfs-live-streaming) ），恰巧，在上周的 IPFS 每周视频会议上，`Yurko`刚好就这个项目与我们分享了他的经验。`Yurko`是一位拥有二十年经验和积极的 IPFS 贡献者的 IT 行业人员。他是一个名为`Toronto Mesh`组织的成员，这个组织致力于在多伦多组建 mesh 网络。如果你想了解更多关于 IPFS 直播视频的相关内容，你可以看一下上周会议的这个视频，看看 Yurko 是如何解释这个问题的。[`www.youtube.com/watch?v=VFsKl6UMwqM&feature=youtu.be&t=435`](https://www.youtube.com/watch?v=VFsKl6UMwqM&feature=youtu.be&t=435)

## IPFS 在业界

你在推特上关注 IPFS 吗？有关新闻中 IPFS 的最新提及，请查看我们的 Twitter 提要或查看有关 Awesome IPFS 的最新文章。

*   IPFS 教程：在网络上使用 HTTP 网关运行你的 IPFS 节点。媒体，2018 年 12 月 3 日。 [`medium.com/@rossbulat/introduction-to-ipfs-set-up-nodes-on-your-network-with-http-gateways-10e21ea689a4`](https://medium.com/@rossbulat/introduction-to-ipfs-set-up-nodes-on-your-network-with-http-gateways-10e21ea689a4)
*   文章：使用 IPFS 和区块链技术解决互联网中心化的问题。Medium，201 Dec 20 [`medium.com/@odomojuli/curatorial-enclaves-cf595e5d099d`](https://medium.com/@odomojuli/curatorial-enclaves-cf595e5d099d)
*   BCH 支持的比特币文件系统项目增加了 IPFS 支持。比特币新闻，2018 年 11 月 28 日。 [`news.bitcoin.com/bch-powered-bitcoin-files-project-adds-ipfs-support/`](https://news.bitcoin.com/bch-powered-bitcoin-files-project-adds-ipfs-support/)
*   应用：使用 Subby 在 IPFS 网络上实现如 YouTube 一般的视频流。 [`medium.com/@subby/stream-videos-from-ipfs-like-on-youtube-using-subby-32b8df17ad03`](https://medium.com/@subby/stream-videos-from-ipfs-like-on-youtube-using-subby-32b8df17ad03)
*   文章：分布式互联网-类似 HBO 的故事？。数据驱动投资者，2018 年 11 月 27 日 [`medium.com/datadriveninvestor/the-decentralized-internet-an-hbo-tale-1c4247efdb44`](https://medium.com/datadriveninvestor/the-decentralized-internet-an-hbo-tale-1c4247efdb44)
*   教程：如何在几秒钟内托管您自己的分布式网站。Pact，2018 年 11 月 27 日。 [`blog.florence.chat/tutorial-how-to-create-your-own-distributed-website-in-just-a-few-seconds-5100ccf068bc`](https://blog.florence.chat/tutorial-how-to-create-your-own-distributed-website-in-just-a-few-seconds-5100ccf068bc)

## 我们的工具和项目

`Awesome IPFS`([`awesome.ipfs.io/`](https://awesome.ipfs.io/) )是一个维护和更新项目、工具列表、或几乎任何与 IPFS 相关的东西的社区，非常棒。要查看更多信息，或将您的信息添加到列表中，请访问[`github.com/ipfs/awesome-ipfs`](https://github.com/ipfs/awesome-ipfs) 。

*   查看一下`textile-go`和用来与 IPFS 交互的`Files API 组件` 。
*   Neilos（[`medium.com/@marco.castignoli/neilos-bfda3f0137c6`](https://medium.com/@marco.castignoli/neilos-bfda3f0137c6) ）提供了一种更安全的浏览网络的方式，在网络中不存在在设备上运行恶意代码的风险。受第一个万维网概念的启发，如果我们拥有强大的组件，就没有必要运行那些没用的代码！
*   基于在线存储方案和 IPFS 的`Authpaper Delivery`的优势到底在哪儿？[`medium.com/@icoauthpapercoin/advantages-of-authpaper-delivery-over-online-storage-solutions-and-ipfs-de9b251f6ecd`](https://medium.com/@icoauthpapercoin/advantages-of-authpaper-delivery-over-online-storage-solutions-and-ipfs-de9b251f6ecd)
*   案例：去中心化分布式的医疗生态系统。[`medium.com/@loudsunday/use-case-for-a-ddhe-1a94529d2a4d%20(DDHE`](https://medium.com/@loudsunday/use-case-for-a-ddhe-1a94529d2a4d%20(DDHE))
*   `sunjam`带给大家一个”Diffuse 音乐播放器“，享受你自己创造的音乐吧。[`ownyourbits.com/2018/11/16/enjoy-your-self-hosted-music-with-diffuse-music-player/`](https://ownyourbits.com/2018/11/16/enjoy-your-self-hosted-music-with-diffuse-music-player/)
*   `Avado Genesis Edition`（The Avado Genesis Edition ），它是一个预先配置的设备，提供 Web3 访问和基于 DAppNode 的个人家庭服务器。该设备将于圣诞节期间推出。它拥有一个 IPFS 节点，一个以太坊完全节点和一个 VPN 标准。

## 社区

您知道 IPFS 在 discuss.ipfs.io 上有社区论坛吗？[`discuss.ipfs.io/`](https://discuss.ipfs.io/) 注册参与有关编码，教程，查看公告和了解即将举行的社区活动的讨论。

*   OPO.js 于 12 月 7 日推出了`Protocol Labs`的`Alex Potsides`活动以及其他一些非常棒的演讲，所以如果你在波尔图，请查看。[`www.meetup.com/opo-js/events/256434646/`](https://www.meetup.com/opo-js/events/256434646/)
*   12 月 9 号，在柏林 IPFS 发起参与 IPFS 开源项目开发活动。[`www.meetup.com/en-AU/IPFS-Berlin/events/255970865/`](https://www.meetup.com/en-AU/IPFS-Berlin/events/255970865/)
*   互联网档案馆将于 2018 年 12 月 11 日在加利福尼亚州旧金山举办去中心化网络见面会。[`www.eventbrite.com/e/decentralized-web-meet-up-tickets-52509395014`](https://www.eventbrite.com/e/decentralized-web-meet-up-tickets-52509395014)
*   在此注册，接收 IPFS 里斯本会议的通知。[`docs.google.com/forms/d/e/1FAIpQLSfJVVPwvp6RY3MUg1zAVl1g_5y2nGb7WJIMI1Hs6glzm7FLHQ/viewform`](https://docs.google.com/forms/d/e/1FAIpQLSfJVVPwvp6RY3MUg1zAVl1g_5y2nGb7WJIMI1Hs6glzm7FLHQ/viewform)
*   介绍 AraCon, 第一届 Aragon 大会，将于 2019 年 1 月 29 至 30 日在德国柏林召开。 [`blog.aragon.org/announcing-aracon-the-aragon-conference/`](https://blog.aragon.org/announcing-aracon-the-aragon-conference/)
*   `Global Diversity CFP Day 2019`将与 2019 年 3 月 2 日举办。届时将有多个 workshop 同时展开，鼓励新人发言，说出你们的提案，给出你们对于科技任何方面的简介，不要错过哦! [`www.globaldiversitycfpday.com/`](https://www.globaldiversitycfpday.com/)
*   Data Terra Nemo 将在 2019 年 5 月举办, 他们已经决定让 Juan Benet 第一个发言。[`dtn.is/`](https://dtn.is/)

感谢您阅读，下周见