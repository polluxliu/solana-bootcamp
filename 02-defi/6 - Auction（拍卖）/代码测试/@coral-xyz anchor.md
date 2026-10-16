# @coral-xyz/anchor

TypeScript 客户端（不论是 `@solana/web3.js` 还是 `@coral-xyz/anchor`）和 Solana 网络的交互，**本质上都是通过 RPC 调用**。Solana 的 RPC 节点会开放 **JSON-RPC API**，分为 HTTP API 和 WebSocket API。

`@solana/web3.js` 是 Solana 官方 JavaScript SDK。而 `@coral-xyz/anchor` 是 Anchor 框架中的 TypeScript SDK，基于 `@solana/web3.js` 封装，是专门为 Solana Anchor 程序设计的客户端。

💡

任何 Solana 程序（包括 Anchor 写的）都能使用 `@solana/web3.js`。

`const program = anchor.workspace.auction as Program<Auction>`

代表 `Auction` 程序的一个 `Program` 实例。

在 `@coral-xyz/anchor` 里，`program.methods.<instruction>()` 是调用链上合约的入口。这些调用都需要一个 **Provider**（`AnchorProvider`），Provider 封装了：

- **connection**：要连接的集群 (mainnet / devnet / localnet)
- **wallet**：签名交易的钱包
- **options**：确认级别、preflight 设置等

`anchor.AnchorProvider.env()` 这个方法会从 **环境变量** 或 **Solana CLI 配置文件** 里读取配置信息并生成一个 Provider。注意，这个方法并不会从当前项目的 Anchor.toml 配置文件中读取配置，这个配置文件是给 `anchor build` , `anchor test`, `anchor deploy` 这些 Anchor CLI 使用的，`@coral-xyz/anchor` 不会使用。

- **Environment Variables（环境变量）**
    
    优先级最高。
    
    - **`ANCHOR_PROVIDER_URL`**
        
        这是你的 Anchor 程序要连接的 Solana 集群 RPC 端点。
        
        例如：
        
        - 本地测试 validator: `http://127.0.0.1:8899`
        - Devnet: `https://api.devnet.solana.com`
        - Mainnet: `https://api.mainnet-beta.solana.com`
    - **`ANCHOR_WALLET`**
        
        指向一个本地的 **keypair JSON 文件**，通常是 `solana-keygen` 生成的 `id.json`。
        
        例如：
        
        - `~/.config/solana/id.json`
        - 或者自定义的钱包路径 `~/.config/solana/devnet.json`
    - **其它 `ANCHOR_*` 环境变量**
        
        其它用于配置 Provider 的选项。
        
        例如：
        
        - `ANCHOR_COMMITMENT`
            - `"processed"` → 默认值，最快，但可能回滚
            - `"confirmed"` → 交易已在一个确认的区块里，不太可能回滚
            - `"finalized"` → 交易所在的区块已达到最大确认，几乎不可能回滚
- **Solana CLI 配置**
    
    如果没有设置环境变量，则会读取这个配置。
    
    Solana CLI 的配置文件路径：`~/.config/solana/cli/config.yml`。当你运行 `solana config get` 时就会把这个文件中的内容打印出来。
    
    ```bash
    Config File: /Users/paulliu/.config/solana/cli/config.yml
    RPC URL: https://api.devnet.solana.com 
    WebSocket URL: wss://api.devnet.solana.com/ (computed)
    Keypair Path: /Users/paulliu/.config/solana/id.json 
    Commitment: confirmed 
    ```
    
    如果这个配置文件也不存在，则会使用默认配置。
    
    - **RPC URL**：默认使用 `http://127.0.0.1:8899`
    - **钱包**：默认使用 `~/.config/solana/id.json`
    - **commitment**：默认使用 `processed`

`AnchorProvider.env()` 不接受参数，如果不想修改 Solana CLI 配置文件，也不想使用环境变量，那可以不用 `env()`，而是手动创建 Provider。

一条指令（Instruction）由程序地址、账户列表、参数数据三部分组成的。

- 程序地址 `programId`
    
    要调用的链上程序的地址。必须指定。
    
- 账户列表 `accounts`
    
    指令在执行时需要访问的账户。必须指定。
    
- 参数数据 `data`
    
    要接收的输入参数，是一个序列化的字节数组。如果指令没有参数，可以不指定。
    

调用程序指令的语法如下：

```tsx
await program.methods
  .<instructionName>(...args)   // ① 指令参数
  .accounts({...})              // ② 账户列表
  .signers([...])               // ③ 额外签名者（可选）
  .preInstructions([...])       // ④ 前置指令（可选）
  .postInstructions([...])      // ⑤ 后置指令（可选）
  .rpc();                       // ⑥ 发送交易
```

背后发生的事情是：

1. **指令构造（Instruction building）**
    
    构造 `TransactionInstruction`：pre → main → post，它们将被 **打包成一笔 Transaction**。
    
2. **交易打包与签名（Transaction assembly & signing）**
    
    把以上的 `TransactionInstruction` 打包进 **Transaction**，并收集钱包签名和所有额外签名。
    
    默认会用 `anchor.getProvider().wallet` 作为 fee-payer，这个 provider 是 Anchor 初始化时设置的，通常情况下它对应的就是你本地 **Solana CLI 配置里的钱包** (`solana config get`)，比如 `~/.config/solana/id.json`。同时收集下面的签名：
    
    - 用 provider 的 wallet（通常是 `ANCHOR_WALLET` 或 `id.json`）签名，它即是 fee-payer 也是默认签名人。
    - 如果 `.signers([...])` 指定了额外签名者，也会签上。注意，即便这里指定了额外的签名者，fee-payer 还是 provider 的 wallet。如果要改变 fee-payer，需要显式设置。
    
    此时得到了签过名的（Versioned）Transaction 的 raw bytes。

    💡
    
    `.rpc()` 是 Anchor 的快捷调用
    
    - 它会 **始终用 `provider.wallet` 作为 fee-payer**。
    - `.signers([...])` 是给 `.rpc()`（或 `provider.sendAndConfirm`）用的 **签名者收集器**。需要把非 provider 的签名者放到 `.signers([...])`，Anchor 会自动把这些 keypair 拿去签名。
    - 即便你 `.signers([auctionHouseAdminKeypair])`，Anchor 也只是额外加签，fee-payer 还是 `provider.wallet`。
    - 如果要指定 fee-payer，需要 **手动构造交易**。

3. **发送与确认（send & confirm）**
    
    把上面的原始交易字节送到 RPC 节点，RPC 节点立即返回交易签名，客户端会根据你配置的确认级别（commitment level）轮询链上的状态，只有当交易状态达到这个确认级别时，`.rpc()` 方法才会返回这个签名。
    
    客户端不断查询这笔交易的状态：
    
    - 如果 commitment = `processed` → 一旦交易在本节点被执行就返回（最快）
    - 如果 commitment = `confirmed` → 等到区块获超多数投票再返回
    - 如果 commitment = `finalized` → 等到区块 root 化才返回（最慢最稳）
4. **返回交易签名 (txSig)**
    
    交易签名是一个 base58 字符串，唯一标识这笔交易。
    
    Solana Rust SDK 里对 `Transaction` 的定义：
    
    ```rust
    pub struct Transaction {
        #[wasm_bindgen(skip)]
        #[serde(with = "short_vec")]
        pub signatures: Vec<Signature>,
    
        #[wasm_bindgen(skip)]
        pub message: Message,
    }
    ```
    
    - **`signatures: Vec<Signature>`**
        - 存储所有签名者对 `Message` 的签名。
        - 顺序与 `Message.account_keys` 中声明的 signer 顺序一致。
        - 第一个签名 **通常** 是 **fee payer** 的签名，也就是我们平时说的“交易签名（transaction signature / txSig）”。
    - **`message: Message`**
        - 交易的“主体”，包含账户列表、指令、recentBlockhash 等。
        - 是所有 signer 用来签名的数据来源。
    
    签名的生成过程如下：
    
    1. 将 Message 序列化为字节数组
    2. 对序列化后的字节数组计算 SHA-256 哈希
        
        `message_hash = SHA256(serialize(Message))`
        
    3. 使用私钥对消息哈希进行签名
        
        每个被标记为 isSigner = true 的账户，都必须对 `message_hash` 签名。
        
        `signature = ed25519_sign(private_key, message_hash)`
        
        签名结果会依次写入 `Transaction.signatures`，顺序对应 `Message.account_keys` 里要求签名的账户顺序。
        
    
    Solana 规定，上面所有签名中的 **第一个签名**（一般是 fee payer 的签名）就是这笔交易的 ID，这个签名被编码为 Base58 字符串，作为 **交易签名**。

---

[返回课程首页](../../../README.md)
