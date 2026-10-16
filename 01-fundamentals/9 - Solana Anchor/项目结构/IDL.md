# IDL

**Interface Description Language** 文件提供了一个标准化的 JSON 文件，用于描述程序的指令和账户。这个文件简化了将你的链上程序与客户端应用程序的集成。

- **标准化**：提供了一种一致的格式来描述程序的指令和账户。
- **客户端生成**：用于生成与程序交互的客户端代码。

```bash
anchor build
```

该命令生成一个位于 `/target/idl/<program-name>.json` 的 IDL 文件。

```rust
use anchor_lang::prelude::*;
 
declare_id!("BYFW1vhC1ohxwRbYoLbAWs86STa25i9sD5uEusVjTYNd");
 
#[program]
mod hello_anchor {
    use super::*;
    pub fn initialize(ctx: Context<Initialize>, data: u64) -> Result<()> {
        ctx.accounts.new_account.data = data;
        msg!("Changed data to: {}!", data);
        Ok(())
    }
}
 
#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(init, payer = signer, space = 8 + 8)]
    pub new_account: Account<'info, NewAccount>,
    #[account(mut)]
    pub signer: Signer<'info>,
    pub system_program: Program<'info, System>,
}
 

#[account]
pub struct NewAccount {
    data: u64,
}
```

上面代码的 IDL 文件为，

```json
{
  "address": "BYFW1vhC1ohxwRbYoLbAWs86STa25i9sD5uEusVjTYNd",
  "metadata": {
    "name": "hello_anchor",
    "version": "0.1.0",
    "spec": "0.1.0",
    "description": "Created with Anchor"
  },
  "instructions": [
    {
      "name": "initialize",
      "discriminator": [175, 175, 109, 31, 13, 152, 155, 237],
      "accounts": [
        {
          "name": "new_account",
          "writable": true,
          "signer": true
        },
        {
          "name": "signer",
          "writable": true,
          "signer": true
        },
        {
          "name": "system_program",
          "address": "11111111111111111111111111111111"
        }
      ],
      "args": [
        {
          "name": "data",
          "type": "u64"
        }
      ]
    }
  ],
  "accounts": [
    {
      "name": "NewAccount",
      "discriminator": [176, 95, 4, 118, 91, 177, 125, 232]
    }
  ],
  "types": [
    {
      "name": "NewAccount",
      "type": {
        "kind": "struct",
        "fields": [
          {
            "name": "data",
            "type": "u64"
          }
        ]
      }
    }
  ]
}
```

`address` 是程序的公钥，通过`declare_id!`宏在程序代码中定义的，它是该程序在Solana区块链上的唯一标识符。

`instructions` 数组对应于你程序中定义的指令。它指定了每个指令所需的账户和参数。

**name**: 指令的名称，指令在程序代码中的名称。

**discriminator**: 8字节的指令鉴别符。它是指令的唯一标识符。Anchor 为程序中的每个指令和自定义账户类型分配一个唯一的 8 字节鉴别符。这些鉴别符作为标识符，用于区分不同的指令或账户类型。

**accounts**: 指令要求的账户列表。

**name**: 账户的名称，这些账户在程序中对应具体的字段名。

**writable**: 布尔值，表示该账户是否可写。如果为true，表示该账户的内容可以在指令执行时修改。

**signer**: 布尔值，表示该账户是否需要在执行该指令时提供签名。如果为true，表示该账户必须提供签名。

**address**: 指定了某个账户的特定地址。它通常用于 **系统账户（如 `system_program`）**，或者当你知道某个账户的固定地址时，直接在 IDL 中指定该地址。

**args**: 指令所需要的参数列表。每个参数都有一个`name`和`type`：

- **name**: 参数的名称，对应程序代码中定义的变量名。
- **type**: 参数的类型。在这里，`data`的类型是`u64`，表示这是一个无符号64位整数。

`accounts` 数组对应于程序中用 `#[account]` 注解的结构体。这些结构体定义了程序创建的账户中存储的数据。

**name**: 账户类型的名称。

**discriminator**: 账户的8字节鉴别符。

`types` 用于描述程序中涉及到的自定义数据类型。

`kind`表示类型的种类，这里是`struct`（结构体）。

`fields`是结构体的字段，包含以下字段：

- **name**: 字段的名称，这里是`data`。
- **type**: 字段的类型，这里是`u64`，表示该字段是一个无符号64位整数。

---

[返回课程首页](../../../README.md)
