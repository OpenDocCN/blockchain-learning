# 第十章 【IPFS 一问一答】IPFS 是怎么挖矿的？与传统的区块链挖矿有什么不同？

# 10\. IPFS 是怎么挖矿的？与传统的区块链挖矿有什么不同？

IPFS 挖矿，在目前来说指的是 Filecoin 的挖矿。它的挖矿原理与传统的区块链挖矿决然不同。

## 10.1 IPFS 挖矿的收益有哪些？

IPFS 矿工分为存储矿工和检索矿工，存储矿工负责给客户存储数据，并获得客户支付的酬金，检索矿工负责给客户提供数据检索，并获得客户提供的酬金。存储矿工可以兼职做检索矿工。存储矿工根据其在网络存储数据的占比，概率性的成为 Filecoin 链的出块节点，通过出块获得激励。

所以存储矿工收益来自：1.提供存储服务获取 token；2.可以兼职做检索矿工，提供检索服务获取 token；3.概率性的成为 leader，为区块链出新块来获取 token；4.获取部分在区块中的交易费用。而检索矿工只能提供检索服务来获取 token。

那么存储矿工这么多收益，大家都会选择做存储矿工吗？根据情况而定。要想成为存储矿工，首先要有足够的硬盘存储空间，而且必须要对自己的存储做抵押。关于用什么抵押，现在确定，白皮书和明面上的资料只是说用抵押品抵押，具体抵押品是什么，可能是 token，但更可能是抵押自己的扇区（未存储数据碎片的硬盘存储空间）。而检索矿工不需要提供硬盘空间，也不需要做抵押，只是提供检索服务，类似中介，帮用户找到存数据的存储矿工并发送数据到用户。

前期 IPFS 挖矿的主要收益是出块奖励，为了能提高自己获得出块权的概率，矿工们会尽量多的在市场上存储数据，并 24 小时保证数据的安全性和矿机的良好运行。

## 10.2 IPFS 挖矿过程

### 10.2.1 存储矿工存储数据过程

矿机通过抵押和提供存储空间，成为存储矿工后，就会进入 Filecoin 的存储市场。与客户之间的交易都是在存储市场上进行的。

![](img/29fdb377aa4e0bb9b86dc287ca4a2bb0.jpg)

根据上图，我们可以看出，存储矿工会在存储市场中发出询价订单 ASK order，即提供存储的空间以及存储要价；客户发出出价订单 BID order，即请求存储的数据以及存储出价。 ![](img/62e1f64ba4bacb36dee0d87383737f5f.jpg) 参照上图， 市场中的所有矿工和客户发出的订单都会存储在链上分配表 inchain-AllocTable 上，该表数据上链。网络会最优匹配 ask 和 bid 订单，使两者达成交易，生成 deal 订单，同时将分配表 AllocTable 上的刚刚完成的订单清除掉。deal 订单翻译为成交订单。客户与矿工在 deal 订单上签字，并由客户提交到区块链上。矿工收取费用并存储客户的数据，客户支付费用并发送要存储的数据。

下图是网络 network 最优匹配价格的过程 ![](img/671fad168710da90cc5dd677c658e401.jpg)

### 10.2.2 存储矿工出块过程

在 Filecoin 区块链中，出块是由存储矿工 leader 负责的，每次出块都会出现新的 leader，而且这个 leader 的产生是随机概率性选举出来的。选举 leader 的概率性公式如下图：

![](img/22ac0acdb1c5382cd74fe112a55a9ef5.jpg)

公式中的左半部分，属于随机函数，输出的值在 0-1 之间，右边部分是代表该矿工在网络中已存储数据的占比。存储矿工成为 leader 的条件就是该不等式成立，所以不等式右边的值越大，该矿工成为 leader 的概率性越大。这正是 Filecoin 激励存储矿工们尽量多的存储数据的策略。关于 leader 选举的具体细节，其他章节分析。

理想状态下，是每次选举会产生一个 leader，，但实际上可能会出现 0 个，1 个或多个 leader 的情况。所以在出块时间（好像是 3 秒）系统给定的情况下： 1）未产生 leader 时，出空块。不产币。 2）1 个 leader，出一个块。全部产币属于该 leader 矿工。 3）多个 leader，出多个块。产币总量与 1 个 leader 产币数量一样，多个 leader 根据系统设定比例分配产币。

### 10.2.3 检索矿工挖矿过程

检索矿工整个挖矿过程，也就是检索过程都是在检索市场上进行的。为了使检索快速，检索是在链下发生的。如下图

![](img/3bc7d6df80e12539e6f5eb102d010d9f.jpg) 检索矿工和客户在检索市场中，类似在存储市场中，各自将对应的 ask 订单和 bid 订单提交到网络的链下分配表 offchain-AllocTable 上。网络会根据最优匹配原则将 ask 订单和 bid 订单配对，使其达成成交订单 deal order。双方都会在 deal 订单上签字，最后由检索矿工提交到区块链。

注意：我们通过要求检索矿工将数据分割成多个碎片，并将每个碎片发送给客户，矿工们将收到付款。在这种方式中，如果客户停止付款，或者矿工停止发送数据，任何一方都可以终止这个交易。

检索市场链下支付模式。Filecoin 区块链必须支持快速的支付通道，实现快速和有效的交易。仅在出现纠纷的情况下才使用区块链进行支付。通过这种方式，检索矿工和客户端可以快速发送 Filecoin 协议所要求的小额支付。这个支付通道是由客户创建的。

## 10.3 IPFS 挖矿与传统区块链挖矿的区别？

Filecoin 挖矿不同于 BTC 挖矿，后者要求挖矿设备具有很高的计算能力，而 Filecoin 挖矿则是为了贡献一个人自己的存储空间。因此，对 Filecoin 挖矿重要的因素是存储容量和上传带宽。

比特币挖矿矿机相对昂贵且耗能（电）。它会产生很大的噪音并在运行时产生大量的热量。比特币挖矿设备运行的更合适的环境是在电费低得多的偏远地区。它们不适合家庭使用。

然而，Filecoin 挖矿的本质是共享一自己的存储空间。这是一个非常环保的过程，能耗和发热量很小，几乎不会产生任何噪音。