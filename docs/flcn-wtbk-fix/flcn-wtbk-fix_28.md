# 第八章 【FileCoin 白皮书中文校正版】-FileCoin 智能合约

Filecoin 为最终用户提供了两个基本命令：Get 和 Put。 这两个命令允许客户以优惠的价格存储数据并从市场中检索数据。 尽管命令涵盖了 Filecoin 的默认使用案例，但我们通过支持智能合约的部署，允许在 Get 和 Put 之上设计更复杂的操作。用户可以编写新的严谨的存储/检索的请求，我们就像归类一般的智能合约一样将其归类为文件合约。我们整合了一个合约系统（基于[18]）和一个桥系统，目的是将 Filecoin 存储装入其他区块链，反之亦然，将其他区块链的功能带入 Filecoin。

我们期望在 Filecoin 生态系统中存在大量的智能合约，我们期待着一个智能合约开发者社区。