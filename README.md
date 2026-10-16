# Solana - 开发营

Web3 现有的学习资源相当匮乏，对于个人学习者来说，获取高质量且有深度的 Solana 学习材料是一项不小的挑战。网络上的内容零零散散，Solana 官方的内容晦涩难懂，市面上有一些看似专业的高价付费课程，也都是网上免费资源的拼凑大合集。

在学习一个知识点的时候，总会出现更多不懂的知识点，这应该是每一个学习者在进入 Web3 时遇到的困难。**为此，我们决定来写出一些系统性的课程，帮助学习者以极低的成本学习到 TOP 级的知识和技能**。我们对呈现的内容要求极其严苛，这使得内容撰写十分耗时，课程的更新需要时间，感谢大家的理解。

## 资源目录

Solana 是一个高性能的区块链平台，旨在提供快速、低成本的去中心化应用（dApps）和加密货币交易。Solana 的目标是成为区块链技术中最快、最具扩展性的网络之一，通过创新的共识机制和技术架构，实现每秒数千甚至数万笔交易（TPS），为开发者和用户提供无与伦比的体验。Solana 的生态系统包括去中心化金融（DeFi）、游戏、NFT、DAO（去中心化自治组织）等。很多项目和开发者已经开始在 Solana 上构建应用程序，这使得 Solana 成为一个充满活力且持续发展的区块链平台。

[第一期：扎实的 Solana 开发入门课](01-fundamentals/README.md)

[第二期：从0到1深入 Solana DeFi](02-defi/README.md)

**第一期：扎实的 Solana 开发入门课**

从 Solana 区块链的知识到 Solana Anchor 开发，扎扎实实的入门课程，我们十分确认，学习完之后就已经入了 Solana 的门，此后可以根据自己的喜好去遨游了。

- 知识部分：Solana 的一些基础知识，包括账户、交易、共识机制、POH 等。
- 代码部分：Solana CLI、Anchor CLI、Mollusk 等内容，介绍 Anchor 脚手架项目，从设计思路到逐行代码讲解投票等经典应用的开发和测试。

**课程目录**

1 - 椭圆曲线密码学

2 - Account

3 - Transaction

4 - Consensus

5 - Solana Node & Light Client

6 - 交易元数据

7 - 交易从创建到广播

8 - 区块从构建到确认

9 - Solana Anchor

- 本地开发环境
- Solana CLI
- Anchor CLI
- 项目结构
- mollusk

10 - Vote program

- 账户上下文
- 账户的数据类型
- 代码 - lib.rs
- 代码 - test_voting.rs
- 代码 - Cargo.toml
- 部署到 devnet

**第二期：从零到一深入 Solana DeFi**

Decentralized Finance（去中心化金融）是一种基于区块链的金融体系，把传统金融服务搬到区块链上，由智能合约自动运行，人人都能参与，没有银行等中介。DeFi 领域的应用类型涵盖了去中心化交易、借贷、稳定币、衍生品以及保险等方向。这些应用共同构建了一个开放、透明且高度可组合的金融生态系统，使传统金融的核心功能能够在区块链上以去中心化的方式实现。

本期从大家使用较多的钱包讲起，分析了 Solana 上的同质化代币和非同质化代币的底层实现，然后通过 Solana Anchor 实现了拍卖、质押、借贷、众筹、AMM 等常见应用的代码，最后系统性地提供了一个 DeFi 项目的分析过程，帮助大家在押注某一个项目时有据可依而不盲从。

**课程目录**

1 - 钱包账户

2 - SPL Token

3 - NFT

4 - 搭建项目开发框架

5 - 代码组织方式

6 - Auction（拍卖）

- 链上拍卖平台的设计思路
- 平台相关的账户模型和指令设计
- 拍卖行相关的账户模型和指令设计
- 拍卖相关的账户模型和指令设计
- 竞拍相关的指令设计
- 结算相关的指令设计
- 代码测试
    - @coral-xyz/anchor
    - @solana/spl-token
    - Mocha & Chai
    - 测试代码分析
        - 准备测试环境
        - 初始化代码
        - initialize_platform
        - add_supported_token
        - create_auction_house
        - create_auction
        - place_bid
        - buy_now
        - settle_auction

~~Staking（质押）~~

~~Lending & Borrowing（借贷）~~

~~Crowdfunding（众筹）~~

~~AMM（自动做市商）~~

~~DeFi 项目分析实战~~

## 更新说明

课程内容将持续完善，后续会陆续补充新章节、实战案例，并更新已有内容。欢迎关注本仓库，获取最新课程。

- [第一期：扎实的 Solana 开发入门课](01-fundamentals/README.md)
- [第二期：从0到1深入 Solana DeFi](02-defi/README.md)

质押、借贷、众筹、AMM 和 DeFi 项目分析实战等章节将陆续更新，敬请期待。
