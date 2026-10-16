# 部署到 devnet

1. 修改 Anchor.toml，在其中添加

```toml
[toolchain]
package_manager = "yarn"

[features]
resolution = true
skip-lint = false

[programs.localnet]
voting = "9aeoZT1WyTfT9cXEEDyb32orFgPsXHpQmcthjGs5Z4qf"

**[programs.devnet]
voting = "9aeoZT1WyTfT9cXEEDyb32orFgPsXHpQmcthjGs5Z4qf"**

[registry]
url = "https://api.apr.dev"

[provider]
cluster = "localnet"
wallet = "~/.config/solana/id.json"

[scripts]
test = "cargo test-sbf"
```

1. 确保当前钱包有足够的 SOL，可以通过 `solana airdrop` 获得空投

```bash
solana balance -u devnet
```

1. 执行部署命令

```bash
anchor deploy --provider.cluster devnet
```

如果部署成功，会输出：

```
Program Id: 9aeoZT1WyTfT9cXEEDyb32orFgPsXHpQmcthjGs5Z4qf
Owner: BPFLoaderUpgradeab1e11111111111111111111111
ProgramData Address: EQPvwShYsR8XohDZziARsRetccoFdTQeXAsMPsLdG1ne
Authority: 3ZoojjwWCgArGREegaQgifwwtZF1EQdid5BjEX5weXah
Last Deployed In Slot: 402333188
Data Length: 260240 (0x3f890) bytes
Balance: 1.81247448 SOL
```

链上会创建两个账户：

- **Program Account（程序账户）**，地址就是 **Program ID**。存储这个程序的“元数据”，标记它是一个可执行程序。
- **Program Data Account（程序数据账户）**，存放你编译后的 `.so` ELF 二进制代码，以及和升级机制相关的信息。

1. 验证是否部署成功

```bash
solana program show 9aeoZT1WyTfT9cXEEDyb32orFgPsXHpQmcthjGs5Z4qf -u devnet
```

1. 查看账户余额，确认是否扣费

```bash
solana balance -u devnet
```

---

[返回课程首页](../../README.md)
