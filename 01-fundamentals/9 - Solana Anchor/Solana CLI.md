# Solana CLI

> 公开版说明：原文中的助记词和私钥示例已移除。请按教程在本地生成测试钱包，并替换代码中的占位内容。

Solana CLI 是和 Solana 网络交互的命令行工具。开发者必备。

```bash
solana config get
```

用于显示当前 Solana 配置文件的设置。通过这个命令，你可以查看当前环境中 Solana CLI 所使用的各种配置选项。

```json
Config File: /Users/paulliu/.config/solana/cli/config.yml
RPC URL: https://api.mainnet-beta.solana.com
WebSocket URL: wss://api.mainnet-beta.solana.com/ (computed)
Keypair Path: /Users/paulliu/.config/solana/id.json
Commitment: confirmed
```

**Config File:** 当前使用的 Solana 配置文件路径。在 Solana CLI 中，所有的配置（如 RPC URL、Keypair 路径等）都保存在该文件中。

**RPC URL:** Solana CLI 当前连接的 RPC 服务器 URL。RPC 是 Solana 的客户端与区块链网络通信的方式。通过 RPC URL，Solana CLI 会与 Solana 网络上的一个特定节点进行通信。

**WebSocket URL:** 与 RPC URL 配对的 **WebSocket** URL。WebSocket 是一种双向通信协议，允许客户端与服务器建立持久连接，以便实时接收事件（如交易确认、区块广播等）。`wss://` 表示 WebSocket 协议的加密版本，它与 `https://` 类似。

**Keypair Path:** Solana CLI 使用的 **密钥对** 文件的路径。用户的 **密钥对** 用于身份验证和签名操作。 Keypair Path 是 Solana CLI 存储该密钥对的文件路径。 **Keypair** 指的就是 **用户的钱包**，在 Solana 中，它代表着用户的 **公钥**（钱包地址）和 **私钥**（用于签名交易的密钥对）。通过这个密钥对，用户能够对自己的账户进行操作，比如发送和接收 Solana 代币（SOL），以及与区块链进行交互。

**Commitment:** Solana 网络查询数据时使用的 **承诺级别（Commitment Level）**。

```bash
solana config set -u mainnet-beta
solana config set -u devnet
solana config set -u localhost
solana config set -u testnet
```

这些命令是用来设置 Solana CLI 所连接的区块链网络。

**`-u`** 参数用于指定要操作的 Solana 集群：

- mainnet-beta（主网）
- devnet（开发网）
- testnet（测试网）
- localhost（本地验证器），指的是通过 **`solana-test-validator`** 启动的一个**持久化的本地验证器网络，不是 `anchor test` 命令启动的临时网络。**

```bash
solana-keygen new
```

用于生成一个新的 **Ed25519 密钥对**（keypair），即用户的 **公钥** 和 **私钥**，通常用于创建新的 Solana 钱包。生成的密钥对存储在上面的默认路径：`~/.config/solana/id.json`。

**`--outfile <path>`**：指定生成的密钥对存储的文件路径。

**`--force`**：如果文件已存在，强制覆盖该文件。

**`--silent`**：该命令会生成一个带有助记词的密钥对，使用 BIP-39 标准格式。但使用 `--silent` 后，助记词不会显示在屏幕上，也不会以任何形式保存。因此，如果你丢失了生成的 `id.json` 文件，**就无法恢复密钥对**。

**`--no-bip39-passphrase`**：**不使用 BIP39 密码短语** 来保护恢复助记词。

**`--word-count <NUMBER>`**：用于 **指定生成的助记词（seed phrase）的单词数**。可选的单词数为 12、15、18、21、24 个。

**BIP39 助记词**：Solana 默认使用 Bitcoin Improvement Proposal 39 标准来生成助记词。这是一组由 12 或 24 个英文单词组成的随机字符串，用于恢复钱包。**默认情况下，Solana 使用 12 个单词的助记词**。

```
[助记词已移除；请在本地生成自己的测试钱包]
```

BIP39 助记词提供了一种通过记忆单词序列来恢复钱包的方式。BIP39 允许你设置一个额外的 **Passphrase（密码短语）**来加密助记词。如果你没有输入密码短语，那么生成的密钥对的安全性依赖于你的助记词本身，任何人拿到你的助记词就能恢复你的钱包。如果你输入了密码短语，那么在恢复钱包时，你不仅需要提供助记词，还需要提供正确的密码短语。即使别人获取了你的助记词，没有正确的密码短语，**他们无法恢复你的钱包**。

`id.json` 文件的内容是一个简单的包含 64 个数字的 JSON 数组，这 64 个数字是 **64 个字节** 的 **二进制数** 的 **十进制表示**。

```json
[/* 私钥已移除：请使用本地测试钱包的 64 字节密钥数组 */]
```

- 前 32 个数字（字节）表示你的 **私钥**。
- 后 32 个数字（字节）表示你的 **公钥**。经过 **Base58 编码** 后，就是我们看到的 **钱包地址**。

该文件中的数字未加密，是以不安全的明文形式存储的。任何拥有该文件的人都可以立即并完全控制与之关联的钱包。因此，保护这个文件就像保护现金一样重要。

💡

可以在 Phantom 钱包中导入 `~/.config/solana/id.json` 。

- 点击 **“添加/连接钱包”** 。
- 在弹出的选项中，选择 **“导入私钥”** 。
- 复制文件中的 JSON 数组，包括方括号 `[]`，并粘贴到 Phantom 的输入框中。
- Phantom 会自动识别并导入你的钱包。

```bash
solana address
```

用于显示当前钱包的 **公钥**（也就是钱包地址）。命令输出的公钥是一个 **Base58 编码** 的字符串，它代表了你的钱包地址。如

```json
CondpyJWjXeLGirpeHJ4g9RtB4UCh79p1dVeFCNSJbiE
```

```bash
solana airdrop -u
```

用于向当前钱包地址免费 **空投（airdrop）SOL** 代币。这是一个用于开发和测试的命令，用于获取一些 **SOL** 代币，以便进行交易、部署程序或进行其他操作。

该命令支持使用 `-u` 参数，但仅在 **localhost** 和 **devnet** 网络上有效，**devnet** 网络每次最多只能空投 5 SOL，在 **localhost** 网络上使用是需要使用 **`solana-test-validator`** 启动本地测试网络。**mainnet** 和 **testnet** 网络无法使用该命令进行空投，你需要通过其他方式获得 SOL。

```bash
solana balance -u
```

用于查看当前钱包地址的 **余额**，即该钱包中的 **SOL** 数量。支持 `-u` 参数，用于指定访问的 Solana 集群，用于显示当前钱包地址在不同网络上的余额。

如果你想查看某个特定钱包地址的余额，而不是默认的地址，可以通过 `--keypair` 参数指定另一个密钥对文件：

```bash
solana balance --keypair /path/to/your/wallet.json
```

```
solana-test-validator
```

启动一个本地的单节点模拟器，它模拟了 Solana 区块链的基本行为，包括区块生产、交易验证等，所有操作都发生在你的本地计算机上，而无需连接到公共网络（如 Devnet 或 Mainnet）。这使得开发者能够在不受外部网络限制的情况下进行快速开发、测试和调试。

**`--ledger`**: 指定一个目录来存储账本数据。如果没有使用 `--ledger` 参数，会自动在当前目录下创建一个名为 `test-ledger` 的文件夹。

**`--reset`**: 如果该选项启用，则每次启动时会清除旧的账本数据，重新初始化。

**`--rpc-port <PORT>`**: 这个参数用于指定启动 **JSON RPC 服务** 的端口。默认情况下，Solana 节点会在 **端口 8899** 上提供 RPC 服务，你可以通过修改这个端口来改变服务的监听端口。

- **RPC 服务端口**：RPC 端口是与外部系统进行交互的主要接口。例如，`solana` CLI 或其他程序可以通过 RPC 端口发起查询、提交交易等操作。
- **WebSocket 端口**：与 RPC 端口关联的下一个端口是用于 **WebSocket** 连接的端口，默认情况下，它是 **端口 8900**。WebSocket 提供了一种更高效的实时数据传输方式，适用于需要持续获取链上更新的应用程序。

# Solana 网络

在 Solana 开发过程中，有多个 **网络环境**（Network Environments）可以选择。

- 本地测试网络（Localnet）
    
    本地网络是完全在你本地机器上运行的，交易和部署速度很快，适合快速迭代和调试。
    
- 开发网（Devnet）
    
    一个公开的测试网络，用于开发和测试。相比本地网络，开发网是一个多人使用的环境，稳定性较高。开发网和主网使用的共识机制和基本架构是一样的，因此可以更真实地模拟实际应用场景。
    
- 测试网（Testnet）
    
    一个接近真实环境的网络，用于更大规模的测试。它类似于主网，接近于主网环境，适合进行最终阶段的验证。
    
- 主网（Mainnet）
    
    Solana 的生产环境，处理所有真实的交易和智能合约部署。

---

[返回课程首页](../../README.md)
