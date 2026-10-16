# 5 - Solana Node & Light Client

在 Solana 网络里，有两大类参与者：Solana Nodes 和 Light Clients。

# Solana Nodes

目前 Solana 网络上几乎所有节点都运行 Agave，是 Solana 官方的验证器客户端（validator client），由 **Anza Labs**（Solana Labs 的核心开发团队）开发和维护的。所谓的不同节点如 Validator、RPC Node、Archive Node、本地测试节点，其实都是 **Agave** 加不同的参数：

- 验证器节点
    
    验证器节点参与共识，赚取奖励，需要质押 SOL。
    
    - 运行 `agave-validator`
    - 投票账户，一个 **链上数据账户**，由 Solana 的 **Vote Program** 管理
    - 默认会开启 **修剪模式 （Ledger Pruning）**，只保留最近 **2 个 epoch** 左右的账本数据（大约 3–4 天），这是为了避免存储无限膨胀，因为 Solana 的吞吐量非常高，历史数据增长极快。
- RPC 节点
    
    不参与共识，维护状态，对外提供 API。
    
    - 运行 `agave-validator`
    - `-full-rpc-api`
    - `--no-voting`
    - 默认情况下，RPC 节点和验证器一样，**也只保留最近 2 个 epoch（大约 3–4 天）的数据**。如果查询的交易已经被修剪（过期了），RPC 节点会返回 Transaction not found。
- 归档节点
    
    不参与共识，保留从创世区块开始的全链数据，并对外提供数据查询。
    
    - 运行 `agave-validator`
    - `-no-prune`
    - `-no-voting`
    - `-full-rpc-api`
- 本地测试节点
    
    运行 `solana-test-validator`（Agave 的简化封装）。
    

通常不会把一个节点配置为即是验证器，也同时是 RPC 节点和 归档节点，RPC 节点和归档节点都是单独的，归档节点通常也是 RPC 节点。RPC 节点、归档节点同验证器节点一样，会收到来自 leader 的 shreds，并在本地重放交易，唯一不同的是不发起投票。

# Light Clients

轻客户端不会运行 `agave-validator` 这种完整节点软件，其运行在手机钱包、浏览器插件、跨链桥、IoT 设备等，如 **Phantom** 等钱包就是轻客户端。

- 只做最小化验证：
    - 下载 **区块头** 或 **部分账户状态**
        
        轻客户端一般只需要下载最新的一部分区块头，用于确认最近的链状态。轻客户端只下载与自己相关或需要验证的那部分账户信息。例如：钱包只下载用户地址相关的账户（用户的 SOL 余额、Token Account、NFT Metadata），跨链桥轻客户端只下载需要验证的目标账户（比如某个 Token Mint 的状态）。
        
    - 通过 **Merkle 证明** 或类似技术验证交易/状态
- 不参与共识。
- 依赖 **RPC 节点** 来获取大部分数据。

---

[返回课程首页](../README.md)
