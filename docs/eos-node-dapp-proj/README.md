# 基于 EOS/Node.js 的 DApp 工程项目实战---去中心化交易所

> 来源：[`www.chaindesk.cn/witbook/38`](https://www.chaindesk.cn/witbook/38)

本项目是使用 EOS、SmartContract、Node.js、React 等技术架构， 采用链上搓合与资产清算的方案实现的去中心化交易所。 本系统核心业务逻辑主要是通过智能合约进行实现的， 其中包括搓合逻辑的处理、关键数据的定义、买卖单的创建以及订单薄的维护；而后端服务主要是以 node.js 技术进行功能实现，一方面用于与区块链的接口交互，比如：查询合约内数据以及链上区块数据；另一方面主要用于对外提供 http 及 socket 接口服务，通过整合业务数据及合约数据，以供前端页面的数据展示；除此之外，后端还有配套的调度服务，实时同步链上数据，并生成不同维度的报表数据。