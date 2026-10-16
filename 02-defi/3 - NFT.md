# 3 - NFT

Ethereum 生态中 **ERC-721** 是最早、最主流的 **NFT 标准**，在 2018 年被提出。其中定义了：

- NFT 合约必须提供哪些函数接口；
- 这些函数的输入、输出、返回值要是什么；
- 事件（Events）要怎么发；
- 语义上如何保证 token 的唯一性。

**ERC-721** 是一份 **标准规范**，**OpenZeppelin** 提供了 ERC-721 的 **标准实现代码**（用 Solidity 写好的合约库），我们只需要继承这些现成的实现，加上自己的逻辑，就能快速写出 NFT 合约。

而在 Solana 上，我们依然不需要额外自定义开发这样的智能合约，由于 **SPL Token Program** 覆盖了同质化与非同质化两类代币，SPL Token Program 直接支持 NFT 的创建与管理。

创建 NFT 只需通过 `spl-token-cli`，将 `decimals` 设置为 **0**，并限制 `supply` 为 **1**，另外，`mint_authority` 需要设置为 **None**，意味着再也不能增发了，`supply` 永远固定。

同时可配合常用的 **Metaplex Token Metadata Program** 实现图片、属性等 NFT 需求。该程序定义了一个 **Metadata Account**，用于存储这些扩展信息。Metadata Account 是一个 PDA，使用的种子为：

```json
["metadata", token_metadata_program_id, mint_pubkey]
```

其中 `mint_pubkey` 是 Mint Account 的地址。该账户的 `data` 字段中存储的 Struct 大致如下：

```rust
pub struct Metadata {
    pub key: Key,              // 标识这个账户是 metadata 类型
    pub update_authority: Pubkey, 
    pub mint: Pubkey,          // 绑定的 mint address
    pub data: Data,            // NFT 数据（name, symbol, uri）
    pub primary_sale_happened: bool,
    pub is_mutable: bool,
    ...
}
```

其中 `data` 里有：

```rust
pub struct Data {
    pub name: String,          // 名字
    pub symbol: String,        // 符号
    pub uri: String,           // 指向 JSON 文件的 URI
    pub seller_fee_basis_points: u16,
    pub creators: Option<Vec<Creator>>,
}
```

**当前 Solana 上的 FT 和 NFT 依然主要采用 Metaplex Metadata Program 来附加和管理标准化的链上元数据**。然而随着 Token-2022 的快速推进，在未来可能会成为主要实现方式之一。

---

[返回课程首页](../README.md)
