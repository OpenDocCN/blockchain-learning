# 第二十六章 【IPFS 一问一答】IPFS 应用层之 Multiformats

# 26 【IPFS 一问一答】IPFS 应用层之 Multiformats

![](img/2e490bce1a9ba74cb9f3cf8ef7b53cf7.jpg)

25 章节讲过 IPFS 的族谱。具体的关系是 Multiformats、LibP2P、IPLD 服务于 IPFS 的，而 Filecoin 是 IPFS 的升级，加了一层激励层。本章节将讲解 Multiformats 项目。

## 26.1 什么是 Multiformats

Multiformats 项目是为 IPFS 协议专门打造的，允许协议相互操作，可以保持协议灵活度、可扩展、可升级，即打造一个永不过时的系统。目前应用在 IPFS 和 libp2p 模块上，在 IPFS 体系中主要负责身份的加密和数据的自我描述，是未来安全系统的协议集合，通过增强自我描述的格式值来实现，自描述格式可以让系统可互相协作和升级。

该协议的自我描述方面有以下规定：

*   它们必须指定的是某个特定的值，而不是从上下文判断的值。
*   它们必须避免只是一种结构，提高它们的可扩展性
*   它们必须紧凑并且能用二进制格式表示。
*   它们必须具有人类可读的表现形式。

## 26.2 Multiformats 的组成内容

目前，Multiformats 的组成部分如下：

*   multihash - 自描述的 hash 值
*   multiaddr - 自描述的网络地址
*   multibase - 自描述的编码值
*   multicodec - 自描述的序列化值
*   multistream - 自描述网络传输流
*   multigram (WIP) - 自描述分组网络协议