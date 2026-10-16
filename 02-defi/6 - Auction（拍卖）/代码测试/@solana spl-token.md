# @solana/spl-token

与 Solana 上的 SPL Token 程序交互的官方 JS/TS SDK。

### 安装

在 Solana Anchor 项目中需要安装：

```bash
yarn add @solana/spl-token
```

### 主要功能

创建新的代币 (Mint)

```tsx
// 创建新的代币类型
const mint = await createMint(
  connection,           // Solana 连接
  payer,               // 付费账户
  mintAuthority,       // 铸币权限
  freezeAuthority,     // 冻结权限
  decimals             // 小数位数
);
```

创建代币账户 (Token Account)

```tsx
// 获取或创建关联代币账户
const tokenAccount = await getOrCreateAssociatedTokenAccount(
  connection,
  payer,              // 付费账户  
  mint,               // 代币 mint 地址
  owner               // 代币账户所有者
);
```

铸造代币

```tsx
// 铸造代币到指定账户
await mintTo(
  connection,
  payer,              // 付费账户
  mint,               // 代币 mint 地址
  destination,        // 目标代币账户
  authority,          // 铸币权限
  amount              // 数量
);
```

转账代币

```tsx
// 转账代币
await transfer(
  connection,
  payer,              // 付费账户
  source,             // 源代币账户
  destination,        // 目标代币账户
  owner,              // 源账户所有者
  amount              // 转账数量
);
```

---

[返回课程首页](../../../README.md)
