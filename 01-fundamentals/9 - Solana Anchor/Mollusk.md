# Mollusk

`mollusk` 是一个测试工具，用于模拟 Solana 运行时环境，让你可以在本地快速测试程序。它在你的测试进程内部运行，完全在内存中模拟 Solana 环境。

要使用 Mollusk，在创建一个新的 Anchor 项目时，使用 `--test-template mollusk` 标志初始化项目。这会将项目设置为使用 Mollusk 进行测试。

Mollusk 不依赖于 **`solana-test-validator`**，而是直接调用 Solana 内部的底层 SVM 组件，自己构建一个最小的执行管道。`solana-test-validator` 和 `mollusk` 都可以用于在本地模拟 Solana 环境，但它们的功能和应用场景有所不同。

|  | `solana-test-validator` | `mollusk` |
| --- | --- | --- |
| **模拟级别** | **进程级**，一个完整的节点模拟器 | **库级**，一个内存中的执行器 |
| **测试类型** | 模拟真实的链上环境，适合**端到端测试** | 模拟程序执行逻辑，适合**单元/集成测试** |
| **运行速度** | 较慢，需要启动一个独立进程 | 极快，在内存中直接运行 |
| **真实度** | 高，可以测试与 SPL 等其他程序的交互 | 稍低，专注于程序本身的逻辑 |

运行 Mollusk 测试可以使用 **`anchor run test`** 或 **`cargo test-sbf`**，也可以使用 **`anchor test`**，但注意，**`anchor test`** 命令默认会启动一个本地 validator，即它会运行 **`solana-test-validator`。**

Mollusk 绕过了完整验证器环境，直接使用 Solana 虚拟机（SVM, Solana Virtual Machine）底层组件来执行程序。Mollusk 不会用 `AccountsDB`（validator 存储账户的数据库）、`Bank`（状态管理）、或其他 validator 的重量级模块，这使得 Mollusk 测试速度非常快，但代价是 **你必须手动提供账户列表**，因为它没有地方能读取链上账户，不像 validator 会维护全链的状态。

# 测试代码的组织

- **一个程序，一个测试文件**：对于一个 Solana 程序（比如你的 `my_test_program`），有一个对应的测试文件，通常命名为 `tests/my_program.rs` 或 `tests/my_program_test.rs`。这个文件中包含了该程序的所有测试用例。
- **一个函数，一个测试用例**：在测试文件中，每个 `#[test]` 函数都代表一个独立的测试用例。这些函数会测试程序中的一个特定指令，或者一系列指令的组合。

# 示例

lib.rs,

```rust
use anchor_lang::prelude::*;

declare_id!("14NyxRRT6pme5p4hNLTKeesXDN5Ucu9je3JJwGPi8yzs");

#[program]
pub mod my_test_program {
    use super::*;

    pub fn initialize(ctx: Context<Initialize>) -> Result<()> {
        msg!("Greetings from: {:?}", ctx.program_id);
        Ok(())
    }
}

#[derive(Accounts)]
pub struct Initialize {}
```

test_initialize.rs,

```rust
#![cfg(feature = "test-sbf")]

use {
    anchor_lang::{solana_program::instruction::Instruction, InstructionData, ToAccountMetas},
    mollusk_svm::{result::Check, Mollusk},
};

#[test]
fn test_initialize() {
    let program_id = my_test_program::id();

    let mollusk = Mollusk::new(&program_id, "my_test_program");

    let instruction = Instruction::new_with_bytes(
        program_id,
        &my_test_program::instruction::Initialize {}.data(),
        my_test_program::accounts::Initialize {}.to_account_metas(None),
    );

    mollusk.process_and_validate_instruction(&instruction, &[], &[Check::success()]);
}
```

`#![cfg(feature = "test-sbf")]`

这是一个 Rust 条件编译属性。它告诉编译器：只有当 test-sbf 这个功能标志被启用时，才编译这段代码。 这确保了这段测试代码只在测试环境下（通常是本地模拟器）被编译和运行，而不会被包含在最终部署到 Solana 链上的程序二进制文件中。

- `#!`: 表示一个**文件级别的属性**。
- `cfg`: 这是 `conditional compilation`（条件编译）的缩写，是 Rust 的一个内置宏，用来控制代码的编译。
- `feature = "test-sbf"`: 这是条件编译的**判断条件**。它检查名为 `"test-sbf"` 的 feature 是否被启用。在 Solana 开发中，`test-sbf` 是一个约定俗成的功能名称，它表示 **Solana BPF (Berkeley Packet Filter) 测试**。SBF 是 Solana 新的虚拟机指令集，BPF 是其旧称。

当你运行 `cargo test` 时，你可以通过 `--features` 标志来启用某个功能。例如：

```bash
cargo test --features test-sbf
```

执行这个命令时，所有被 `#![cfg(feature = "test-sbf")]` 标记的代码都会被编译和执行。当你在构建用于部署到链上的程序时，你不会启用 `test-sbf` 功能，因此所有的测试代码都会被忽略，从而保持你的程序二进制文件干净、精简。

`use { ... };`

导入必要的依赖库。最外层的花括号 `{}` 将所有 `use` 语句都包裹在一个代码块中，使得代码看起来更整洁。

`#[test]`

这是一个 Rust 属性，标记了 test_initialize 函数是一个测试函数。当运行 cargo test 时，Rust 会自动找到并执行这个函数。

`let program_id = my_test_program::id();`

my_test_program::id() 是一个由 Anchor 自动生成的函数，它返回你的程序的 Program ID（公钥）。这个 ID 是一个 Pubkey 类型，它在测试中用来指定指令的目标程序。

`let mollusk = Mollusk::new(&program_id, "my_test_program");`

Mollusk::new() 是 Mollusk 库提供的构造函数，这行代码创建了一个 Mollusk 实例，它是一个独立的、内存中的 Solana 运行时模拟器。

- &program_id：传入程序的 ID，告诉 mollusk 应该模拟哪个程序。
- "my_test_program"：传入程序的名称，用于在 mollusk 的内部日志或状态中进行标识。

`let instruction = Instruction::new_with_bytes( ... );`

这部分代码是构建一个 Solana 指令。一个指令由三部分组成：程序 ID、指令数据和账户元数据。

- 程序 ID (Program ID) 是指令的**收件人**，是一个 32 字节的公钥，用于唯一标识你要与之交互的智能合约。
- 指令数据 (Instruction Data) 是指令的**内容**，是一个字节数组，包含了你告诉智能合约具体要做什么的信息。
- 账户元数据 (Account Metas) 是指令的**操作对象**，是一个 `AccountMeta` 对象列表，包含了指令需要读写的所有账户的公钥以及它们的权限信息。

💡

`program ID` 用于标识你的程序，`Instruction Data` 包含鉴别符和参数，`Account Metas` 则指定了涉及的账户及其权限（is_signer, is_writable）。一个交易指令需要明确列出它将访问的所有账户，并为每个账户提供元数据（`AccountMeta`）。这个元数据就包含了账户的权限信息。

一个指令就像这样一句话：**“嘿，`[程序 ID]`，请`[指令数据]`地操作这些账户`[账户元数据]`。”** 这三部分共同构成了一个完整的、可执行的指令。

`program_id`

传入程序的 ID，指定指令应该发往哪个程序。

`&my_test_program::instruction::Initialize {}.data()` 

将指令的类型信息和参数打包成 Solana 程序能够理解的字节数组。

- Anchor 会为你的每一个公共函数（`pub fn`）生成一个同名的、用于序列化指令数据的结构体。因为你的函数名是 `initialize`，所以生成的结构体就叫做 `Initialize`，并将其放在 `instruction` 模块下。Anchor 会根据你的指令函数签名，动态地生成指令结构体。如果函数没有参数，结构体就是空的；如果有参数，结构体就会包含这些参数作为字段。
- `my_test_program::instruction::Initialize {}`：初始化结构体。比如，
    
    ```rust
    my_test_program::instruction::Initialize {} // 无参数
    
    my_test_program::instruction::Initialize { value: 500 } // 有参数
    ```
    
- `.data()`：将**指令鉴别器（Instruction Discriminator）和指令参数**一起打包成一个字节数组。其结构大致如下：
    
    ```json
    [8字节的指令鉴别器] + [指令参数]
    ```
    
    - **指令鉴别器：**是一个 8 字节的哈希值，由指令函数的签名生成。这个哈希值是唯一的，并且在编译时就确定了。生成鉴别器的过程是这样的：
        1. 函数签名：Anchor 会获取指令函数的完整签名，包括函数名和所有参数的类型。
            
            对于 `initialize` 指令：
            
            - 没有参数时，函数签名是 `global:initialize`。
            - 有一个 `u64` 参数时，函数签名是 `global:initialize:u64`。
        2. SHA-256 哈希：Anchor 对这个完整的函数签名字符串进行 SHA-256 哈希。
        3. 前 8 字节：最后，Anchor 取这个哈希值的前 8 个字节作为指令的鉴别器。
    - **指令参数：**获取为 `Initialize` 结构体提供的**具体值**。这部分长度取决于指令函数签名中参数的数量和类型。如上面代码例子中的 `value: 500`，`.data()` 方法会把这个 `u64` 整数 `500` 序列化成 8 个字节，并将其附加在鉴别器后面，总共 16 字节。
    
    当这个字节数组发送到链上后，Solana 程序会先读取开头的 8 个字节，通过它来识别出这是 `initialize` 指令。然后，程序会根据 `initialize` 指令的预期参数（`u64`），读取接下来的 8 个字节，并将其反序列化为 `500` 这个 `u64` 值。
    

`my_test_program::accounts::Initialize {}.to_account_metas(None)`

这行代码将你用 `#[derive(Accounts)]` 宏定义的、易于理解的账户结构，转换成 Solana 运行时所需的、更底层的 `Vec<AccountMeta>` 格式。

- `accounts` 是 Anchor 在编译时自动生成的模块。它包含了你用 `#[derive(Accounts)]` 宏定义的所有账户上下文结构体。
- `Initialize {}` 是定义的 `Initialize` 账户上下文结构体的一个实例。因为 `Initialize` 结构体没有字段，所以初始化时使用了空的 `{}`。
- `to_account_metas()` 方法的作用是遍历 `Initialize` 结构体中定义的账户字段，并将每个账户转换成一个 **`AccountMeta`** 对象。**`AccountMeta`** 包含了执行指令所需账户的**元数据**：
    - `pubkey`: 账户的公钥。
    - `is_writable`: 一个布尔值，指示程序是否需要对该账户进行写入（例如，修改余额或数据）。
    - `is_signer`: 一个布尔值，指示该账户是否需要提供签名来授权操作。
    
    `is_signer` 和 `is_writable` 标志会根据字段上使用的属性**自动设置**， ****
    
    - 一个带有 `#[account(mut)]` 的字段将被标记为 `is_writable: true`（可写）。
    - 一个带有 `#[account(mut, signer)]` 或仅 `#[account(signer)]` 的字段将被标记为 `is_signer: true`（签名者）。
    - 一个没有任何属性的字段将被标记为 `is_writable: false` 和 `is_signer: false`（只读）。
    
- `None` 表示不提供可选账户。在那些需要根据不同条件选择性地提供账户的复杂指令中，通过这个参数来告诉 `to_account_metas` 如何处理这些可选账户。
    - 当传入 `Some(key)` 时，这个 `key` 会被用来填充可选账户的公钥。
    - 当传入 **`None`** 时，它告诉 Anchor **“这个账户是可选的，但在这笔交易中，我不提供它。”**上面代码中的 `Initialize` 结构体中没有可选账户需要处理。即使传入了 `None`，`to_account_metas` 也会正常工作，因为它知道所有账户都是必需的，并会为它们生成正确的 `AccountMeta`。

`mollusk.process_and_validate_instruction(...);`

这是 `mollusk` 库提供的核心方法，执行 Solana 指令并验证结果。

💡

`process` -> 模拟 Solana 运行时处理一条指令的整个过程，包括账户权限检查、程序执行、状态更新等。

`validate` -> 在指令执行后，检查执行结果是否符合预期。

- `instruction`
    
    构建好的指令。`process_and_validate_instruction` 方法需要一个指令作为输入，它包含了程序ID、指令数据和账户元数据。
    
- `&[]`
    
    **该指令会用到的所有账户**的集合，在这里提供了一个空的列表。
    
    由于 Mollusk 是一个模拟器，它没有真正的区块链数据库来加载账户数据。因此，需要我们手动提供所有在指令中会用到的账户及其初始状态。
    
- `&[Check::success()]`
    
    一个验证检查列表，用于告诉 `mollusk` 你期望的执行结果。
    
    - `[Check::success()]` 创建了一个**包含单个 `Check::success()` 元素的数组**。
    - `&` 获取了这个数组的**引用**，从而创建了一个数组切片。
    - `Check` 是 `mollusk` 库中的一个枚举，定义了多种检查类型。**`Check::success()`** 是最常用的一种检查。它告诉 `mollusk`，你期望这条指令执行成功。如果指令在模拟运行时中失败了（比如返回了一个错误），`mollusk` 就会让整个测试失败。`Check` 枚举还有其他变体。

# Accounts

## System Account

不包含数据（data 字段为空），由 **System Program** 所拥有的账户。有两种变体，

### Payer

拥有 lamports，用于为程序账户创建或 lamport 转账提供资金。

```rust
let payer_pubkey = Pubkey::new_unique();
let payer_account = Account::new(LAMPORTS_PER_SOL / 2, 0, &system_program_pubkey);
```

`Account::new()` 创建一个 `Account` 结构体的实例，函数的三个参数：

- `lamports: u64` - SOL 余额，决定了账户中存储的 SOL 数量，以 lamports（Solana 的最小单位）为单位。
- `space: usize` - 数据空间大小，定义了账户存储**自定义数据**的空间大小，以字节为单位。
- `owner: &Pubkey` - 账户所有者，指定了账户的**所有者**。

### Default Account

没有 lamports，表示一个等待初始化的账户。

```rust
let default_account = Account::default();
```

`Account::default()` 用于创建一个 `Account` 结构体的实例，并为其字段赋予默认值。它利用了 Rust 的 **Default trait**，该 trait 提供了一种标准方式来创建一个“零值”或空实例。

- lamports: u64：`u64` 类型的默认值是 `0`。这意味着账户的余额为零 SOL（即零 lamports）。
- data: Vec<u8>：`Vec<u8>`（动态字节数组）的默认值是一个空向量。这意味着账户没有数据，数据大小为零。
- owner: Pubkey：`Pubkey` 类型的默认值是一个零值的 `Pubkey`，它对应于 System Program ID（`11111111111111111111111111111111`）。
- executable: bool：`bool` 类型的默认值是 `false`。这意味着账户不是一个可执行的程序。
- rent_epoch: Epoch：`Epoch` 类型的默认值是 `0`。这表示账户的租金（rent）已经过期，且是在过去的一个 epoch 中到期（对于一个未初始化的账户，这通常不重要）。

## Data Account

```rust
let data = vec![
    // Your serialized account data
];
let lamports = mollusk
    .sysvars
    .rent
    .minimum_balance(data.len());
 
let program_account = Pubkey::new_unique();
let program_account_account = Account {
    lamports,
    data,
    owner: ID, // The program's that owns the account
    executable: false,
    rent_epoch: 0,
};
```

这段代码使用**结构体字面量**的方式，创建了一个 `Account` 结构体的实例。

- **`lamports`**: 设置为上一步计算出的租金豁免额。
- **`data`**: 设置为第一步准备好的序列化数据。
- **`owner: ID`**: **`ID`** 通常代表你的程序 ID（Program ID）。这行代码将账户的**所有者**设置为你的智能合约。这意味着只有你的智能合约才能修改这个账户的数据。
- **`executable: false`**: 这个字段被设置为 `false`，明确地表明这个账户是一个**数据账户**，而不是一个存储可执行代码的**程序账户**。
- **`rent_epoch: 0`**: 这是租金相关的字段，在测试中通常设置为默认值。

let lamports = mollusk.sysvars.rent.minimum_balance(data.len());

- **Sysvars** 是 Solana 上的一些特殊只读账户，用于提供网络范围内的系统信息，例如当前的时隙、epoch，最重要的是，租金（rent）费率。`mollusk.sysvars` 是测试框架提供的，用于在模拟环境中访问这些系统变量。
- **rent**：这是访问 **Rent Sysvar** 的方式。Rent Sysvar 包含了与 Solana 租金机制相关的所有参数和逻辑。Solana 的租金机制确保账户存储的数据有足够的余额来支付存储费用，否则账户会被删除。
- **minimum_balance()**：该方法计算一个账户必须持有的 lamports 数量，以便使该账户免于支付租金。一个 **免租账户** 永远不会被收取租金，也不会被网络“垃圾收集”（即删除）。
- **data.len()**：这是传递给 `minimum_balance()` 函数的参数，表示账户数据的大小（以字节为单位）。账户的租金豁免所需的 lamports 数量与账户存储的数据量成正比。

# Instructions

```rust
let instruction = Instruction::new_with_bytes(
    ID, // Your program's ID
    &[0], // Instruction data (discriminator + parameters)
    vec![AccountMeta::new(payer, true)], // Account metadata
);
```

- 指令数据必须包含指令的鉴别符，后面跟着指令所需的任何参数。
- 对于 `AccountMeta` 结构体，我们需要根据账户权限使用适当的构造函数：
    
    ```rust
    AccountMeta::new(pubkey, is_signer) // For mutable accounts
    AccountMeta::new_readonly(pubkey, is_signer) // For read-only accounts
    ```
    
    is_signer 参数表明该账户是否必须对交易进行签名。大多数账户是非签名者（false），但支付者 （payers） 和需要对操作授权的账户（authorities）除外。
    

# Execution

准备好账户和指令后，你可以使用 Mollusk 的 execution API 来执行和验证你的程序逻辑。Mollusk 提供了四种不同的执行方法，具体取决于你是否需要验证检查，以及你是在测试单个指令还是多个指令。

这些方法中的第二个参数为 `accounts: &[(Pubkey, Account)]` ，这个参数是一个切片（slice），其中包含了元组 `(Pubkey, Account)` ，是所有账户的集合。

```rust
let accounts = [
    (user, user_account),
    (program_account, program_account_account)
];
```

当指令被执行时，Solana 运行时（由 Mollusk 模拟）会根据指令中的 `AccountMeta` 列表，从你提供的 `accounts` 集合中查找并加载相应的账户数据。

在指令执行后，Mollusk 会更新这个 `accounts` 集合中的 `Account` 结构体，反映出账户状态的变化（例如，`lamports` 减少，`data` 改变）。这样，你就可以在测试中检查这些变化是否符合预期。

- 最简单的执行方法是执行单个指令但不验证，这将返回执行结果，你可以手动验证。

```rust
mollusk.process_instruction(&instruction, &accounts);
```

- 对于复杂的测试，允许指定预期的结果。

```rust
mollusk.process_and_validate_instruction(
    &instruction,
    &accounts,
    &[
        Check::success(), // Verify the transaction succeeded
        Check::compute_units(5_000), // Expect specific compute usage
        Check::account(&payer).data(&expected_data).build(), // Validate account data
        Check::account(&payer).owner(&ID).build(), // Validate account owner
        Check::account(&payer).lamports(expected_lamports).build(), // Check lamport balance
    ],
);
```

- 验证系统支持多种检查类型，以验证执行结果的不同方面。对于边缘情况测试（Edge Case Testing），您可以验证指令是否按预期失败。

```rust
mollusk.process_and_validate_instruction(
    &instruction,
    &accounts,
    &[
        Check::err(ProgramError::MissingRequiredSignature), // Expect specific error
    ],
);
```

💡

**边缘情况测试（Edge Case Testing）指的是对软件、系统或程序在极端、不寻常或边界条件**下进行测试，以确保其行为正确、稳定且不会崩溃。

- 使用指令链方法测试需要多个指令的复杂工作流程。

```rust
mollusk.process_instruction_chain(
    &[
        (&instruction, &accounts),
        (&instruction_2, &accounts_2)    
    ]
);
```

- 结合多个指令和全面的验证。

```rust
mollusk.process_and_validate_instruction_chain(&[
    (&instruction, &accounts, &[Check::success()]),
    (&instruction_2, &accounts_2, &[
        Check::success(),
        Check::account(&target_account).lamports(final_balance).build(),
    ]),
]);
```

---

[返回课程首页](../../README.md)
