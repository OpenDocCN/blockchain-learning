# 第五章 IPFS Weekly 5

# IPFS Weekly 5

作者：Richard Littauer，Andrew Chin，2016-03-04

IPFS（[`ipfs.io/`](https://ipfs.io/) ） 是一种新的超媒体分发协议，通过内容和身份进行寻址，旨在使网络更快，更安全，更开放。在这些帖子中，我们将尽量强调过去一周发生的一些发展。对于任何想要参与的人，请打开本文嵌入的超链接，在 github（[`github.com/ipfs`](https://github.com/ipfs) ） 上搜索足够的信息或在 IRC（[`webchat.freenode.net/?channels=ipfs`](https://webchat.freenode.net/?channels=ipfs) ）上加入我们。

本期周刊内容涵盖了上个月发生的一些事情。

## 精彩呈现

我们的朋友和一些用户在`ConsenSys`上发表了一篇文章“Introduction to IPFS”（链接[`medium.com/@ConsenSys/an-introduction-to-ipfs-9bba4860abd0`](https://medium.com/@ConsenSys/an-introduction-to-ipfs-9bba4860abd0) ）。文章从不太专业的序言开始，之后深入全面的讲解 IPFS 模型是如何工作的。内容涉及了多个实例，包括目录结构、版本控制系统和区块链。这是个非常好的文章，它使读者熟悉了 IPFS 对象的简单工作过程和图形可视化，并且进一步深入理解了 IPFS。这篇文章还有一个特点，就是封面图片非常神奇。感谢@ChrisLundkvist 和@ConsenSysAndrew!

## 更新

*   @lgierth 升级了 go-ipfs 的自动化集成的 docker 镜像，该镜像现在命名为`ipfs/go-ipfs`。获取镜像的命令是：`docker run -i --net=host ipfs/go-ipfs`。每次代码提交都会自动化生成一个镜像，并且在最新发布的镜像是那个有个自动生成的标签。镜像大小也就几 MB，安装时不再需要给 IPFS 仓库提供空间。相反，如果没有安装空间，它将生成一个暂时的身份和配置信息，当仓库存在以后这些内容也随之被丢弃。这将是一个在测试或测试某项工作时最完美的方案。关于`go-ipfs 0.3.x`的镜像没有提供，因为它对应的 Dockerfile 文件版本不适合 Docker Hub 的自动化构建。
*   @noffle, @whyrusleeping 和 @chriscool 一直在改进测试套件和片状测试（过去常常出现不准确）。这样，我们将整合 PRs 更快，同时 CI 对成功和失败实现更准确的分类。
*   @noffle 也在`go-ipfs`与@vanilla 的`go get`安装兼容方面取得了进展。希望月底能够出现更多的有意义的结果。
*   @RichardLitt 在处理`IPFS HTTP API`的同时，还整理了大量关于 CLI 修复的文档。
*   @chriscool 重新编辑了`the build documentation for Windows`，使 Windows 用户更容易地安装和使用 IPFS。
*   @lgierth 发布了新的博文`migration from 0.3.x to 0.4.0.`
*   @noffle 改进了 FUSE 连接和终止。

## GX

@whyrusleeping 发布了一个 PR，它引入了一个名为`gx`的工具，用于作为我们项目的依赖包。以前我们使用 godeps，当初在 go-ipfs 仓库构建 ipfs 时，总是还间接的保存了构建 ipfs 所需的所有代码。所以我们很难将其应用，首先，它使存储库的大小膨胀超过我们的代码库的原始大小，导致克隆需要更长时间，并使 CI 更慢。其次，更新这些依赖关系是一件麻烦事：这部分是由于糟糕的软件包管理选择的错误，部分是因为我们发现 godeps UX 不友好。要解决这个问题，@ whyrusleeping 创建了`gx`。`Gx`是一个基于 ipfs 的包管理工具。包的引用是在 merkletree 中链接的所有哈希，并且解析给定项目的所有依赖项就像 ipfs fetch 一样简单。现在我们正在使用 gx，主要的 go-ipfs repo 非常小，依赖关系可以很容易地获取和安装（并在项目中共享），我们还将使用 ipfs 索引 ipfs。

从普通用户的角度来看，有一些小变化; `go get`不再是安装 ipfs 的可行方法，用户现在需要用`make install`，就像其他大型 golang 项目（其中包括 docker 和 kubernetes）一样运行。你可以在这里阅读更多关于 gx 的内容：github.com/whyrusleeping/gx 和 gx-go（gx 专用于 go 的子工具）：github.com/whyrusleeping/gx-go 。

@RichardLitt 改进了`gxREADME`文档以更好地解释其目标，并帮助大家启动起来。如果您认为可以改进哪些内容，请查看并通过问题向我们提供反馈。

## ipns-pub

@whyrusleeping 发明了一种工具，叫`ipns-pub`。它允许大家在不实际运行节点的情况下可以向网络发布 IPNS 条目。你可以通过`ipfs-key`产生密钥对，然后通过密钥发布你喜欢的任何 IPFS 路径的内容。请注意，使用此工具发布的条目有效期是 24 小时。所以一定要保证发布的条目处于活跃状态。工具里还有一个`--daemon`选项，它可以每隔 12 小时自动重新发布你的条目，使条目永远处于活跃状态。

## specs（软件包脚本文件）

经过几个月的缜密设计，我们终于将 IPLD 的 spec 文件并入到主分支里。这里面大部分工作是由@mildred 和@jbenet 完成的。他们整合了好多社区人员提供的建议和设计方案。包括
“thin-waist” Merkle DAG、merkle-links、-dags、 -paths、IPLD Data 等内容。简言之：命名为 merkle-links 格式的 JSON 文档可以被遍历。

## http-api-spec

@RichardLitt 完成了所有当前关于`HTTP API Spec`的 ipfs 命令的整理工作。这就意味着，如果有任何关于 HTTP API 如何工作的问题，你可以在主分支上或打开`PRs`得到你想要的答案。如果你对 HTTP API 是如何工作的很感兴趣，或者有任何具体问题，请检查当前版本（[`github.com/ipfs/http-api-spec/blob/master/apiary.apib`](https://github.com/ipfs/http-api-spec/blob/master/apiary.apib) ），或进入[`github.com/ipfs/http-api-spec/pulls`](https://github.com/ipfs/http-api-spec/pulls) 填写你的问题。

## 发布

dist.ipfs.io（[`dist.ipfs.io/`](https://dist.ipfs.io/) ）

## js-ipfs

感谢@diasdavid，使 DAG 对象的操作命令可以正常使用。同样，感谢@dignified 做出的努力。如果没有指定的回调，js-ipfs API 现在可以实现返回 promises。允许 Javascript 社区使用的两种主要方法同样有效。

## registry-mirror 镜像

@diasdavid 通过删除对 registry-static 的依赖来提高 registry-mirror 性能和健壮性。复制了一些必要的部分。通过 registry-mirror 克隆 registry，使可靠性和性能迈出了重要一步。

## station

@dignifiedquire 修复了掉线或草稿文件的上传，以及一些依赖性问题。所以继续尝试吧。

## ipfs-geoip

@dignifiedquire 重新编辑了生成脚本文件，并清理了代码，以便现在数据始终可以重现并完全存储在 IPFS 上。这可确保 IPFS 上的 geoip 查找在所有将来的版本中都能正常工作。

## fs-repo-migrations

@chriscool 改进了 fs-repo-migrations 的测试方法 - 测试通过各种简单工作负载在向前和向后迁移时验证更多边缘情况

## ipfs-hyperlog

@noffle 构建了`ipfs-hyperlog`，和与 ipfs 兼容的`hyperlog`分支，一个基于`scuttlebutt`日志和因果链接复制的 DAG。`ipfs-hyperlog`将直接替代@mafintosh 的`hyperlog`。它的关键区别在于它创建了一个与 IPFS 对象二进制兼容的 Merkle DAG 。这意味着使用 ipfs-hyperlog 构建的任何 DAG 的任何节点都可以复制到 IPFS 网络和从 IPFS 网络复制！

## Logo

@Kubuxu 开发了一个新的 IPFS Logo（[`ipfs.io/ipfs/QmTgtbb4LckHaXh1YhpNcBu48cFY8zgT1Lh49q7q7ksf3M/`](https://ipfs.io/ipfs/QmTgtbb4LckHaXh1YhpNcBu48cFY8zgT1Lh49q7q7ksf3M/) ）。进入看一下吧。

## 社区

*   如果您对会议有任何建议，可以在[`github.com/ipfs/community/issues/105`](https://github.com/ipfs/community/issues/105) 上提交您的提案。IPFS 社区将看看我们是否可以参加那次会议并在那里开展活动。
*   我们现在在`ipfs/community README`([`github.com/ipfs/community#events-calendar`](https://github.com/ipfs/community#events-calendar) )上有一个 IPFS 社区事件列表。你有任何想要添加的内容吗？查看过去的活动。
*   @RichardLitt 建议使用新的 GitHub 模板进行 IPFS。你怎么看待这个想法？[`github.com/ipfs/community/issues/108`](https://github.com/ipfs/community/issues/108)

## IPFS 的落地应用

*   Marmot Image Checker：来自 Twitter（[`twitter.com/asciinema/status/701730719589126146`](https://twitter.com/asciinema/status/701730719589126146) ）
*   asciienema 现在允许在 IPFS 上播放。来自 Twitter([`twitter.com/asciinema/status/701730719589126146`](https://twitter.com/asciinema/status/701730719589126146) )

## 贡献者

在整个 IPFS GitHub 组织中，以下人员在 2 月 1 日（中午，GMT）和 2 月 29 日之间就 GitHub 提交了代码，提出问题或发表评论。我们使用此工具和其他工具自动生成此列表，因此，如果您的姓名不在此处，请告知我们。

*   @abacon (Bacon)
*   @almereyda (jon r)
*   @amstocker (Andrew Stocker)
*   @anacrolix (Matt Joiner)
*   @anarcat (anarcat)
*   @Ape (Lauri Niskanen)
*   @area
*   @ARezaK
*   @Asgraf (Michal Turek)
*   @Balancer (Balancer)
*   @balupton (Benjamin Lupton)
*   @bierlingm
*   @BigBlueHat (BigBlueHat)
*   @boergsen
*   @boxxa (Boxxa)
*   @briantigerchow (Brian Tiger Chow)
*   @brimstone (Matt)
*   @bussiere (bussiere)
*   @bzz (Alexander)
*   @chpio
*   @chris-martin (Christopher Martin)
*   @chriscool (Christian Couder)
*   @christianlundkvist (Christian Lundkvist)
*   @ChristopherA (Christopher Allen)
*   @cjcase (Cj Case)
*   @cleichner (Chas)
*   @codeburd
*   @Crest (Crest)
*   @cryptix (Henry)
*   @daveajones (Dave Jones)
*   @David-Leudolph (David Leudolph)
*   @david415 (David Stainton)
*   @davidar (David A Roberts)
*   @denisnazarov (Denis Nazarov)
*   @dginev (Deyan Ginev)
*   @diasdavid (David Dias)
*   @dignifiedquire (Friedel Ziegelmayer)
*   @djdv (Dominic Della Valle)
*   @dominictarr (Dominic Tarr)
*   @Dumptel
*   @dylanPowers (Dylan Powers)
*   @edent (Terence Eden)
*   @ehd (Stephan Seidt)
*   @EliasGabrielsson (Elias Gabrielsson)
*   @emardee
*   @eminence (Andrew Chin)
*   @faebser (Fabian Frei)
*   @fazo96 (Enrico Fasoli)
*   @Fil (Fil)
*   @frabrunelle (Francis Brunelle)
*   @GitCop
*   @GravisZro (Gravis)
*   @greenkeeperio-bot (Greenkeeper)
*   @halseth
*   @harlantwood (Harlan T Wood)
*   @hitchcott (Chris Hitchcott)
*   @hosh (Ho-Sheng Hsiao)
*   @hutenosa
*   @IanCal (Ian Calvert)
*   @ianopolous (Ian Preston)
*   @ingokeck (Ingo Keck)
*   @insanity54 (Chris Grimmett)
*   @ion1 (Johan Kiviniemi)
*   @j-h-scheufen
*   @j4nu5 (Kushagra Sinha)
*   @jamescarlyle (James Carlyle)
*   @jbenet (Juan Benet)
*   @jbshirk (Joe)
*   @jedahan (Jonathan Dahan)
*   @jeffscottward (Jeff Ward)
*   @jefft0 (Jeff Thompson)
*   @johncant (John Cant)
*   @kalmi (Tarnay Kálmán)
*   @Kolomona (Kolomona Myer)
*   @krl (kristoffer)
*   @Kubuxu (Jakub Sztandera)
*   @kyledrake (Kyle Drake)
*   @kyrias (Johannes Löthberg)
*   @lamarpavel
*   @lernisto (Terrel Shumway)
*   @lgierth (Lars Gierth)
*   @lidel (Marcin Rataj)
*   @lockedshadow
*   @lovelaced
*   @mappum (ᴍᴀᴛᴛ ʙᴇʟʟ)
*   @matshenricson (Mats Henricson)
*   @MichaelMure (Michael Muré)
*   @micxjo (Micxjo Funkcio)
*   @mildred (Mildred Ki’Lya)
*   @mindhog
*   @Mithgol
*   @mnp (Mitchell Perilstein)
*   @montagsoup
*   @MrChrisJ (Chris Ellis)
*   @mseri
*   @NDuma (NDuma)
*   @nginnever (Nathan Ginnever)
*   @NightRa (Ilan Godik)
*   @NodeGuy (David Braun)
*   @noffle (Stephen Whitmore)
*   @odipar (rapido)
*   @olizilla (Oli Evans)
*   @palesz (Palesz)
*   @palkeo (palkeo)
*   @parkan (Arkadiy Kukarkin)
*   @peerchemist
*   @pietsch (Christian Pietsch)
*   @prusnak (Pavol Rusnak)
*   @randomshinichi
*   @reit-c
*   @rethore (Pierre-Elouan Réthoré)
*   @rht
*   @RichardLitt (Richard Littauer)
*   @richardschneider (Richard Schneider)
*   @rsynnest
*   @rubiojr (Sergio Rubio)
*   @rwcarlsen (Robert Carlsen)
*   @se3000 (Steve Ellis)
*   @Shaaah (Shaaah)
*   @shtukas (Pascal Honoré)
*   @sivachandran (Sivachandran)
*   @sleep-walker (Tomáš Čech)
*   @slothbag
*   @Stebalien (Steven Allen)
*   @suisha (David Mai)
*   @tcyrus (Timothy Cyrus)
*   @thelinuxkid (Andres Buritica)
*   @tidux (Todixu)
*   @tinybike (Jack Peterson)
*   @tomgg (tmg)
*   @tv42 (Tv)
*   @void4
*   @wanderer
*   @wasserfuhr (RainerWasserfuhr)
*   @whyrusleeping (Jeromy Johnson)
*   @xicombd (Francisco Baio Dias)
*   @yangwao (Matej Nemček)
*   @zignig (Simon Kirkby)
*   @Zogg