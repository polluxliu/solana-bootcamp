# 10 - Vote program

一个完整的投票应用的功能通常包括：

1. **创建投票（`create_poll`）**
2. **添加候选项（`add_candidate`）**
3. **移除候选项（`remove_candidate`）**
4. **投票（`vote`）**
5. **结束/关闭投票**（通常是时间驱动，不需要单独的指令）

# 账户模型设计

数据如何存储和访问

## Poll（投票）

```rust
#[account]
#[derive(InitSpace, Debug)]
pub struct Poll {
    pub poll_id: u64,
    pub creator: Pubkey,
    #[max_len(128)]
    pub title: String,
    #[max_len(280)]
    pub description: String,
    pub poll_start: u64,
    pub poll_end: u64,
    pub candidate_amount: u64,
    pub next_index: u64,
}
```

- **`pub poll_id: u64`**:
    - **类型：** `u64`，一个 8 字节的无符号 64 位整数。
    - **作用：** 用于存储一个独特的、由客户端提供的 ID，作为 Poll 账户的唯一标识。

        💡
        
        这里 `poll_id` 被设计为在链下（off-chain）生成并通过指令参数传递给链上（on-chain）程序。如果 `poll_id` 在链上生成，例如通过一个自增计数器，多个用户同时创建 Poll 时可能会发生**竞态条件（race condition）**。虽然可以通过在程序中锁定账户来解决，但这会增加复杂性并可能导致交易失败。将 `poll_id` 的生成放在链下，可以彻底避免这种问题。

- **`pub creator: Pubkey`**:
    - **类型：** `Pubkey`，一个 32 字节的公钥类型。
    - **作用：** 存储创建这个 Poll 的钱包地址。

        💡
        
        `creator` 字段记录了账户的拥有者，表明是谁发起了这个投票。这让程序能够验证只有原始创建者才能执行某些特权操作。例如，只有创建者才能关闭投票、添加新选项或编辑投票详情。如果没有这个字段，任何人都可以执行这些敏感操作，从而破坏投票系统的完整性。

- **`#[max_len(128)] pub title: String`**:
    - **类型：** `String`，一个可变长度的 UTF-8 编码字符串。
    - **作用：** 存储 Poll 的标题，最大长度被限制在 128 字节。
- **`#[max_len(280)] pub description: String`**:
    - **类型：** `String`。
    - **作用：** 存储 Poll 的描述，最大长度被限制在 280 字节。
- **`pub poll_start: u64`**:
    - **类型：** `u64`。
    - **作用：** 存储 Poll 开始投票的时间戳（例如 Unix 时间戳）。
- **`pub poll_end: u64`**:
    - **类型：** `u64`。
    - **作用：** 存储 Poll 结束投票的时间戳。
- **`pub candidate_amount: u64`**:
    - **类型：** `u64`。
    - **作用：** 存储此 Poll 中有多少个候选选项。
- **`pub next_index: u64`**：
    - **类型：** `u64`。
    - 表示下一个新候选人应该使用的索引。这个索引永远只增不减。在移除候选项时，剩余候选项的 index 不用改变，而后续新的候选项可以继续增加。

## Candidate（投票选项）

如何存储投票选项？

**选项 1：固定大小数组**

我们可以使用类似 `pub candidates: [String; 5]` 的方式。这种方法简单，但限制了选项的数量。

**选项 2：向量（Vector）**

`pub candidates: Vec<String>`。这种方式更灵活，但我们仍然需要对向量的大小设置一个合理的限制，以便管理账户大小。

**选项 3：单独账户**

我们可以为每个选项创建一个单独的 `Candidate` 账户，并在其中添加一个 `Poll` 相关的字段，将它与投票关联起来。这种方式更复杂，但如果投票选项很多，扩展性更强。

```rust
#[account]
#[derive(InitSpace)]
pub struct Candidate {
    pub poll_id: u64,
    pub index: u64,
    #[max_len(128)]
    pub title: String,
    pub vote_amount: u64,
}
```

- **`pub poll_id: u64`**
    
    这个字段用于存储**所属投票的唯一标识符**。
    
    - **作用**：它将 `Candidate` 账户与创建它的 `Poll` 账户关联起来。
    - **设计原因**：由于我们之前约定 `poll_id` 是全局唯一的，它就成了一个可靠的“外键”。通过这个 ID，你的程序可以找到对应的 `Poll` 账户，尽管它不是一个直接的区块链地址。
- **`pub index: u64`**
    
    这个字段用于存储该选项在特定投票中的**索引或编号**。
    
    - **作用**：它为每一个候选选项提供了一个独一无二的编号。例如，一个投票的选项可以依次被赋予 `index: 0`, `index: 1`, `index: 2` 等。
    - **设计原因**：当你需要为某个投票创建多个选项账户时，这个字段是 PDA 种子的关键部分，确保每个选项都有一个独一无二的地址。
- **`#[max_len(128)] pub title: String`**
    
    这个字段用于存储候选选项的**具体名称或描述**。
    
    - **作用**：它包含了用户在投票时能看到的选项文字，比如“是”、“否”或“选项A”。
    - **设计原因**：`String` 类型非常灵活，能够存储 UTF-8 编码的文本。`#[max_len(128)]` 属性则强制规定了字符串的最大长度为 128 个字节，这确保了账户的大小是固定的，方便 Anchor 计算所需的存储空间。
- **`pub vote_amount: u64`**
    
    这个字段用于存储该候选选项的**总投票数**。
    
    - **作用**：每当有用户投票给这个选项，这个字段的值就会增加。
    - **设计原因**：`u64` 类型可以存储一个巨大的无符号整数，足以应对绝大多数投票场景。这个字段是投票程序的核心，因为它直接反映了投票结果。

## VoterRecord（投票记录）

只记录某个投票人在某投票（poll_id）中是否已经投过票，并不保存具体投给了哪个候选选项，保证了投票的**匿名性**。

```rust
#[account]
#[derive(InitSpace)]
pub struct VoterRecord {
    pub voter: Pubkey,
    pub poll_id: u64,
}
```

**`pub voter: Pubkey`**

- **类型**: `Pubkey`，这是 Solana 中表示公钥（Public Key）的标准类型。
- **作用**: 这个字段用于存储**投票人的公钥**。

**`pub poll_id: u64`**

- **类型**: `u64`，一个 64 位无符号整数。
- **作用**: 这个字段用于存储**投票的唯一标识符**。

# 指令设计

账户模型定义了你的程序在链上存储和操作的数据结构，而**指令**则定义了你的程序可以执行的**操作行为**。账户模型是**数据**，指令是**行为**。

## create_poll

创建一个投票

账户上下文

```rust
#[derive(Accounts)]
#[instruction(poll_id: u64, title: String, description: String, poll_start: u64, poll_end: u64)]
pub struct CreatePoll<'a> {
    #[account(
        init,
        payer = creator,
        space = 8 + Poll::INIT_SPACE,
        seeds = [b"poll", poll_id.to_le_bytes().as_ref()],
        bump
    )]
    pub poll: Account<'a, Poll>,

    #[account(mut)]
    pub creator: Signer<'a>,

    pub system_program: Program<'a, System>,
}
```

**`init`**: 这个参数表明这是一个新账户，需要进行初始化。当 `init` 被指定时，Anchor 会自动生成代码来创建账户，并将 `payer` 账户中足够的 SOL 转移到该账户中以支付租金（rent）。

- 当一个用户调用 `create_poll` 指令时，Anchor 会根据 `poll_id` 和种子（`seeds = [b"poll", poll_id.to_le_bytes().as_ref()]`）来计算出 `poll` 账户的预期地址。
- 然后，它会检查这个地址上是否有账户。
- 如果这个地址上**没有账户**，创建一个新账户。
- 如果这个地址上**已经有账户**（例如，之前已经有人用相同的 `poll_id` 创建过投票），`init` 宏就会阻止交易。

**`payer = creator`**: 指定了创建和支付新账户租金的账户。在这里，`creator` 账户将为 `poll` 账户支付租金。

💡

Solana 运行时会**自动从 `payer` 账户中划走**一笔 SOL，并将其存入新创建的账户中，以支付其存储所需的租金。Solana 运行时会根据在 `#[account(init, ...)]` 宏中指定的 `space` 字段的值自动计算出所需的**免租金**（rent-exempt）金额，作为开发者，不需要手动计算账户的租金金额。

如果 `payer` 账户的余额**不足以支付**创建新账户所需的租金（以及其他交易费用），那么整个**交易会立即失败并被中止**。

**`space = 8 + Poll::INIT_SPACE`**: 指定了账户所需的字节空间大小。

- **`8`**: 这是 Anchor 自动为每个账户添加的 8 字节的账户鉴别器（discriminator）。这是 Anchor 内部用于区分不同类型账户的机制。
- **`Poll::INIT_SPACE`**: 这是 `Poll` 数据结构自身所需的空间大小。

**`seeds = [b"poll", poll_id.to_le_bytes().as_ref()]`**: 定义了账户的程序地址派生（Program Derived Address，PDA）种子。

- `b"poll"`: 一个字节切片，作为第一个种子。
- `poll_id.to_le_bytes().as_ref()`: 将 `poll_id`（一个 `u64`）转换为小端字节数组，并作为第二个种子。
- 通过使用这些种子，我们可以确定性地为每个 `poll_id` 派生一个唯一的账户地址。这使得我们可以从 `poll_id` 重新计算出账户地址，而无需将其存储在其他地方。

**`bump`**: 这个参数与 `seeds` 一起使用。这里 `bump` 并没有设置一个值，它告诉 Anchor 宏，PDA 地址的 `bump` 种子（一个用于确保地址不在椭圆曲线上的 `u8` 值）应该由 Anchor 在运行时自动寻找。Anchor 会找到一个 `bump`，使得 `(seeds, bump)` 组合下的地址是一个有效的 PDA，并且这个值会写入 `ctx.bumps` 中供使用。

**`mut`**: 这个参数表示该账户在指令执行过程中会被修改。在这里，`creator` 账户的 SOL 余额会因为支付 `poll` 账户的租金而减少，所以需要 `mut` 标记。

指令处理器

```rust
    pub fn create_poll(
        ctx: Context<CreatePoll>,
        poll_id: u64,
        title: String,
        description: String,
        poll_start: u64,
        poll_end: u64,
    ) -> Result<()> {
        let poll = &mut ctx.accounts.poll;
        let creator = &ctx.accounts.creator;

        // 初始化 Poll 账户
        poll.poll_id = poll_id;
        poll.creator = creator.key();
        poll.title = title;
        poll.description = description;
        poll.poll_start = poll_start;
        poll.poll_end = poll_end;
        // 选项数量为 0
        poll.candidate_amount = 0_u64;
        poll.next_index = 0_u64;

        Ok(())
    }
```

客户端在调用该指令的时候，需要传递的参数（以 TypeScript 为例）：

- **指令（Instruction）参数**
这些参数是 `#[instruction(...)]` 宏中定义的，是指令逻辑本身需要的输入。它们将作为交易数据的一部分被序列化。
    - **`poll_id`**: `number` 或 `BN` 类型，对应 `u64`。
    - **`title`**: `string` 类型。
    - **`description`**: `string` 类型。
    - **`poll_start`**: `number` 或 `BN` 类型，对应 `u64`。
    - **`poll_end`**: `number` 或 `BN` 类型，对应 `u64`。
- **账户（Accounts）参数**

这些是 struct CreatePoll 中定义的账户，用于指令的上下文。你需要为每个账户提供其公钥（PublicKey）。

💡

Solana 的设计原则是：**所有指令需要访问的账户都必须在交易被提交时** **预先声明**。

- **`poll`**: `PublicKey` 类型。这是最重要的一个账户。由于它是一个 PDA，客户端需要使用与链上指令**完全相同的种子**来计算出它的地址。
    - 种子：`['poll', pollId]`
    - 你需要调用 Anchor 提供的 `PublicKey.findProgramAddress()` 或 `program.findProgramAddress()` 方法来计算出这个 PDA 的地址。
- **`creator`**: `PublicKey` 类型。这是发起交易的签名者账户的公钥。通常，这是钱包连接后返回的账户。
- **`systemProgram`**: `PublicKey` 类型。这是 Solana 系统程序的公钥，通常可以通过 `anchor.web3.SystemProgram.programId` 访问。

客户端代码示例，

```tsx
import * as anchor from '@project-serum/anchor';
import { PublicKey, SystemProgram } from '@solana/web3.js';
import { Program } from '@project-serum/anchor';
import { MyProgram } from './my-program-idls'; // 替换为你的 IDL 类型

// 假设你已经设置好了 provider 和 program
const provider = anchor.AnchorProvider.local();
anchor.setProvider(provider);
const program = anchor.workspace.MyProgram as Program<MyProgram>;

// 1. 定义指令参数
const pollId = new anchor.BN(123);
const title = "Vote for the best fruit";
const description = "Choose your favorite fruit.";
const pollStart = new anchor.BN(Date.now() / 1000);
const pollEnd = new anchor.BN(pollStart.add(new anchor.BN(86400))); // 24小时后结束

// 2. 计算 poll 账户的 PDA 地址
const [pollAccount, _bump] = PublicKey.findProgramAddressSync(
    [
        anchor.utils.bytes.utf8.encode("poll"), // 种子1: "poll"
        pollId.toBuffer("le", 8) // 种子2: pollId (u64) 的小端字节表示
    ],
    program.programId // 你的程序 ID
);

// 3. 构造并发送交易
async function createPoll() {
    try {
        const tx = await program.methods
            .createPoll(
                pollId,
                title,
                description,
                pollStart,
                pollEnd
            )
            .accounts({
                poll: pollAccount,
                creator: provider.wallet.publicKey,
                systemProgram: SystemProgram.programId,
            })
            .rpc();

        console.log("Poll created! Transaction signature:", tx);
    } catch (error) {
        console.error("Failed to create poll:", error);
    }
}

createPoll();
```

## add_candidate

为投票添加选项

账户上下文

```rust
#[derive(Accounts)]
pub struct AddCandidate<'a> {
    #[account(
        mut,
        has_one = creator,
        seeds = [b"poll", poll.poll_id.to_le_bytes().as_ref()],
        bump
    )]
    pub poll: Account<'a, Poll>,

    #[account(
        init,
        payer = creator,
        space = 8 + Candidate::INIT_SPACE,
        seeds = [b"candidate", poll.poll_id.to_le_bytes().as_ref(), poll.candidate_amount.to_le_bytes().as_ref()],
        bump
    )]
    pub candidate: Account<'a, Candidate>,

    #[account(mut)]
    pub creator: Signer<'a>,

    pub system_program: Program<'a, System>,
}
```

**`has_one = creator`**: 这个 `poll` 账户中必须有一个名为 `creator` 的字段，并且该字段的值必须与交易签名者 `creator` 的公钥匹配。这样是为了保证：**只有发起投票的人，才能给自己的投票加候选项**，防止别人乱加。

**`poll`** 账户约束中的 **`seeds`** 和 **`bump`** 的作用是告诉 Anchor **如何验证传入的 `poll` 账户地址**。当客户端调用 `AddCandidate` 指令并传入 `poll` 账户地址时，Anchor 在链上会执行以下步骤：

1. 它会从 `seeds` 参数中获取种子列表：`[b"poll", poll.poll_id.le_bytes()]`。
2. 它会使用这些种子和你的程序 ID，**重新计算**出一个 PDA 地址。
3. 它将这个**计算出的地址**与客户端实际传入的 `poll` 账户地址进行比较。

如果这两个地址不匹配，交易就会失败。这个验证过程确保了：

- 你正在操作的 `poll` 账户**确实是一个由你的程序派生出来的 PDA**。
- 你正在操作的 `poll` 账户**确实是与 `poll_id` 匹配的正确账户**。

这个验证可以防止恶意用户传递一个不相关的账户地址，并声称它是你的投票账户。

注意，上面的 `bump` 约束并没有设定值，这就意味着 `bump` 值需要自动找到。我们可以将账户创建时自动找到的 `bump` 值存储起来并在这里指定 `bump = some_value`，这样可避免重新寻找。

这里 `poll` 字段使用了 `poll.poll_id` 作为其 seeds，这理解起来可能有些费劲。其机制如下：

1. 客户端在 TypeScript 等环境中，会根据预先知道的 `poll_id` 来计算出 `poll` 账户的 PDA 地址。然后，它将这个地址以及其他必需的账户（如 `creator`）打包到交易的账户列表中，并发送到 Solana 网络。
2. **Solana 运行时**接收到交易后，会根据客户端提供的账户地址列表（公钥列表），从区块链账本中读取相应的账户数据。它会把这些原始的二进制字节加载到内存中，准备好供程序指令访问。
3. **Anchor 框架在解包指令时**会利用 `#[derive(Accounts)]` 宏的定义来处理内存中的账户数据。
    - Anchor 会自动将加载的 `poll` 账户数据（字节）反序列化成 `Poll` 结构体实例。这个实例现在包含了 `poll_id`、`candidate_amount` 等所有字段。
    - Anchor 宏开始执行 `seeds` 约束。它会从刚刚反序列化得到的 `poll` 结构体实例中，直接访问 `poll.poll_id` 字段的值。
    - Anchor 使用这个**从链上账户数据中读取的 `poll_id`** 来重新计算出一个 PDA 地址。然后，它将这个计算出的地址与客户端在交易中提供的**原始 `poll` 账户地址**进行比较。
        - 如果两者**匹配**，验证通过。这证明了客户端提供的 `poll` 账户不仅存在，而且确实是由您的程序根据 `poll.poll_id` 派生出来的，是**合法的、可信的**账户。
        - 如果两者**不匹配**，验证失败。这通常意味着客户端提供了一个无效或伪造的地址，指令将立即中止并返回错误。

指令处理器

```rust
    pub fn add_candidate(ctx: Context<AddCandidate>, title: String) -> Result<()> {
        let poll = &mut ctx.accounts.poll;
        let candidate = &mut ctx.accounts.candidate;

        // 验证投票时间
        let current_timestamp = Clock::get()?.unix_timestamp as u64;
        msg!("current timestamp - {}", current_timestamp);
        msg!(
            "poll start timestamp - {}, poll end timestamp - {}",
            poll.poll_start,
            poll.poll_end
        );
        if current_timestamp >= poll.poll_start {
            if current_timestamp <= poll.poll_end {
                return err!(ErrorCode::VotingInProgress);
            } else {
                return err!(ErrorCode::VotingPeriodOver);
            }
        }

        // 初始化 Candidate 账户
        candidate.poll_id = poll.poll_id;
        candidate.index = poll.next_index;
        candidate.title = title;
        // 得票数为 0
        candidate.vote_amount = 0_u64;

        // 更新 Poll 账户的选项数量
        poll.candidate_amount += 1;
        poll.next_index += 1;

        Ok(())
    }
```

`Clock::get()?.unix_timestamp` 获取的时间戳来自 **当前处理这个交易的验证器** 的本地时间，底层都使用了 Solana 的 **`Clock` Sysvar**。虽然 Solana 各个验证器之间会通过网络时间协议（NTP）同步时间，但由于网络延迟、硬件时钟漂移等因素，它们的时间戳仍可能存在微小的差异（通常在几秒内）。另外，一个恶意的验证器可能故意使用一个不准确的时间戳，试图利用时间来操控合约逻辑。例如，它可能会使用一个未来的时间戳来提前激活一个活动。

比如 **验证器 A** 的时钟稍微快一点，**验证器 B** 的时钟稍微慢一点，假设投票的结束时间是 `T`，这会导致用户在投票截止时间点附近提交交易时，**结果不一致**。但是，对于投票或类似应用来说，这种细微的时间差异是**完全可以接受的**。

💡

对于需要精确、不可篡改时间的应用，比如金融衍生品或高频交易，使用去中心化的预言机网络（如 **Chainlink** 或 **Pyth**）来获取外部时间是更好的选择。预言机从多个可靠来源收集时间数据并进行聚合，然后将这个“公认的”时间戳写入链上，这比单个验证器的时间戳更具抗审查性和可信度。

注意，在 **本地测试（mollusk / solana-test-validator）环境** 下，`Clock` 系统变量默认是未初始化的，`Clock::get()?.unix_timestamp` 获取到的时间戳是 **0** 。为了解决这个问题，需要在测试代码中**模拟**一个 `Clock` 账户并添加到指令的账户列表中。

`err!()` 是 Anchor 中的一个声明式宏，其代码如下，

```rust
#[macro_export]
macro_rules! err {
    ($error:tt $(,)?) => {
        Err(anchor_lang::error!($crate::ErrorCode::$error))
    };
    ($error:expr $(,)?) => {
        Err(anchor_lang::error!($error))
    };
}
```

其接收一个自定义错误类型或者一个内置的错误类型（如 `ProgramError` 和 `AnchorError`），并将其转换为 `anchor_lang::prelude::Error` 类型，这是 **Anchor 框架最顶层的、统一的错误类型**。所有从 Anchor 程序返回的错误最终都会被封装成这个类型。

我们可以把错误类型分为三个层次：

- **`ProgramError`**: 来自 Solana 运行时的底层错误，比如账户权限不足。
- **`AnchorError`**: 来自 Anchor 框架自身的错误，比如账户约束检查失败。
- 由 `#[error_code]` 宏创建的**自定义错误**。

尽管有现成的错误可用，但在大多数情况下，**自定义错误是更好的选择**。自定义错误（如 `VotingPeriodOver`）能**精确地表达**错误发生的具体原因。相比之下，如 `ProgramError::InvalidArgument` 或 `AnchorError::ConstraintRaw` 过于通用，无法提供具体上下文。自定义错误的代码如下，

```rust
#[error_code]
pub enum ErrorCode {
    #[msg("Voting is not open yet.")]
    VotingNotStarted,
    #[msg("Voting period has ended.")]
    VotingPeriodOver,
    #[msg("Voting is in progress.")]
    VotingInProgress,
}
```

`#[error_code]` 是一个**属性宏**，被用于 `enum` ，作用是**将一个普通的 Rust 枚举转换成 Anchor 框架可识别的、具有唯一错误码的错误类型**。

`#[msg]` 也是一个**属性宏**，与 `#[error_code]` 一起使用，被**附加在枚举的每个成员上**，作用是**为错误码提供一个可读的、描述性的字符串信息**。

客户端代码示例，

```tsx
import * as anchor from '@project-serum/anchor';
import { PublicKey, SystemProgram } from '@solana/web3.js';
import { Program } from '@project-serum/anchor';
import { MyProgram } from './my-program-idls'; // 替换为你的 IDL 类型

// 假设你已经设置好了 provider 和 program
// 例如，使用 Anchor 的本地测试环境或连接到 devnet
const provider = anchor.AnchorProvider.local();
anchor.setProvider(provider);
const program = anchor.workspace.MyProgram as Program<MyProgram>;

// 1. 定义需要从客户端传递的参数
const pollId = new anchor.BN(123); // 假设这是你要添加候选人的投票 ID
const title = "New Candidate Option"; // 新候选人的标题

// 2. 准备所有必需的账户公钥
async function addCandidate() {
    try {
        // 首先，计算现有的 poll 账户的 PDA 地址
        const [pollAccount, _pollBump] = PublicKey.findProgramAddressSync(
            [
                anchor.utils.bytes.utf8.encode("poll"),
                pollId.toBuffer("le", 8)
            ],
            program.programId
        );

        // 接下来，计算新 candidate 账户的 PDA 地址
        // 这需要从链上获取 poll 账户的当前状态，以获得 candidate_amount
        const pollAccountState = await program.account.poll.fetch(pollAccount);
        const candidateAmount = pollAccountState.candidateAmount;
        
        const [candidateAccount, _candidateBump] = PublicKey.findProgramAddressSync(
            [
                anchor.utils.bytes.utf8.encode("candidate"),
                pollId.toBuffer("le", 8),
                candidateAmount.toBuffer("le", 8)
            ],
            program.programId
        );
        
        // 3. 调用指令并发送交易
        const tx = await program.methods
            .addCandidate(title)
            .accounts({
                poll: pollAccount,
                candidate: candidateAccount,
                creator: provider.wallet.publicKey,
                systemProgram: SystemProgram.programId,
            })
            .rpc();

        console.log(`Successfully added a new candidate!`);
        console.log(`Transaction signature: ${tx}`);
        console.log(`New candidate account address: ${candidateAccount.toBase58()}`);

    } catch (error) {
        console.error("Failed to add candidate:", error);
    }
}

addCandidate();
```

## remove_candidate

移除候选项

账户上下文

```rust
#[derive(Accounts)]
pub struct RemoveCandidate<'a> {
    #[account(
        mut,
        has_one = creator,
        seeds = [b"poll", poll.poll_id.to_le_bytes().as_ref()],
        bump
    )]
    pub poll: Account<'a, Poll>,

    #[account(
        mut,
        close = creator,
        seeds = [b"candidate", candidate.poll_id.to_le_bytes().as_ref(), candidate.index.to_le_bytes().as_ref()],
        bump
    )]
    pub candidate: Account<'a, Candidate>,

    #[account(mut)]
    pub creator: Signer<'a>,

    pub system_program: Program<'a, System>,
}
```

`close = creator` 是一个 Anchor 特性，它会自动处理关闭账户的逻辑，在账户 **关闭** 时，租金会被退还到 `close` 约束指定的账户，这里就是 `creator` 账户。

💡

Solana 账户的租金机制

在 Solana 上，每个账户都需要支付一笔租金，以保证其数据能够持久地存储在链上。这笔租金是 **预支付** 的，存储在账户的 `lamports` 字段中。

Solana 让账户的存储成本只在账户被创建时支付一次，这笔预付的租金**只要足以支付未来两年的租金，账户就会被标记为免租金（rent-exempt）**。这笔 SOL 会被锁在账户里，不会 **被划走**，账户关闭时会将所有租金退回到指定的账户。注意：Solana 的 `close` 约束允许将账户的租金退还到 **任何** 指定的账户，不一定是最初的付款账户。

指令处理器

```rust
    pub fn remove_candidate(ctx: Context<RemoveCandidate>) -> Result<()> {
        let poll = &mut ctx.accounts.poll;
        let candidate = &mut ctx.accounts.candidate;

        // 验证投票时间
        let current_timestamp = Clock::get()?.unix_timestamp as u64;
        msg!("current timestamp - {}", current_timestamp);
        msg!(
            "poll start timestamp - {}, poll end timestamp - {}",
            poll.poll_start,
            poll.poll_end
        );
        if current_timestamp >= poll.poll_start {
            if current_timestamp <= poll.poll_end {
                return err!(ErrorCode::VotingInProgress);
            } else {
                return err!(ErrorCode::VotingPeriodOver);
            }
        }

        // Anchor 的 `close` 约束会自动关闭账户

        // 更新 Poll 账户的选项数量
        poll.candidate_amount -= 1;

        Ok(())
    }
```

客户端代码示例，

```tsx
import * as anchor from "@coral-xyz/anchor";
import { Program } from "@coral-xyz/anchor";
import { PublicKey, SystemProgram } from "@solana/web3.js";
import { Voting } from "../target/types/voting"; // 确保路径正确

// 假设你已经设置了 Anchor 提供者
const provider = anchor.AnchorProvider.local();
anchor.setProvider(provider);
const program = anchor.workspace.Voting as Program<Voting>;

// 定义必要的参数
const pollId = new anchor.BN(12345); 
const candidateIndex = new anchor.BN(0); // 你要移除的候选项的索引

// 1. 准备所有必需的账户公钥
async function removeCandidate() {
  try {
    // A. 从 provider.wallet 获取创建者的公钥
    const creator = program.provider.wallet.publicKey;

    // B. 根据种子（seeds）计算 Poll 和 Candidate 账户的 PDA
    const [pollAccount] = PublicKey.findProgramAddressSync(
      [anchor.utils.bytes.utf8.encode("poll"), pollId.toBuffer("le", 8)],
      program.programId
    );

    const [candidateAccount] = PublicKey.findProgramAddressSync(
      [
        anchor.utils.bytes.utf8.encode("candidate"),
        pollId.toBuffer("le", 8),
        candidateIndex.toBuffer("le", 8),
      ],
      program.programId
    );

    console.log(`Poll Account PDA: ${pollAccount.toBase58()}`);
    console.log(`Candidate Account PDA: ${candidateAccount.toBase58()}`);

    // 2. 调用 removeCandidate 指令并发送交易
    const tx = await program.methods
      .removeCandidate() 
      .accounts({
        poll: pollAccount,
        candidate: candidateAccount,
        creator: creator,
        systemProgram: SystemProgram.programId,
        // 如果你的 Rust 函数需要 clock，也请在此处传入
        // clock: anchor.web3.SYSVAR_CLOCK_PUBKEY,
      })
      .rpc(); 

    console.log("成功移除候选人！");
    console.log(`交易签名: ${tx}`);
    
  } catch (error) {
    console.error("移除候选人失败:", error);
    // 捕获并解析 Anchor 自定义错误
    if (error instanceof anchor.AnchorError) {
      const parsedError = program.idl.errors.find(
        (err) => err.code === error.error.errorCode.code
      );
      if (parsedError) {
        console.error("程序错误信息:", parsedError.msg);
      }
    }
  }
}

// 执行函数
removeCandidate();
```

## vote

参与投票

账户上下文

```rust
#[derive(Accounts)]
pub struct Vote<'a> {
    #[account(
        seeds = [b"poll", poll.poll_id.to_le_bytes().as_ref()],
        bump
    )]
    pub poll: Account<'a, Poll>,

    #[account(
        mut,
        seeds = [b"candidate", candidate.poll_id.to_le_bytes().as_ref(), candidate.index.to_le_bytes().as_ref()],
        bump
    )]
    pub candidate: Account<'a, Candidate>,

    #[account(
        init,
        payer = voter,
        space = 8 + VoterRecord::INIT_SPACE,
        seeds = [b"voter_record", poll.poll_id.to_le_bytes().as_ref(), voter.key().as_ref()],
        bump
    )]
    pub voter_record: Account<'a, VoterRecord>,

    #[account(mut)]
    pub voter: Signer<'a>,

    pub system_program: Program<'a, System>,
}
```

- **`poll`**: 投票账户。它必须是已存在的，且其 PDA 地址需要被验证。它是只读的，因为投票操作不会改变投票本身的状态。

    💡
    
    这里添加 **`poll`** 账户的作用有两个，一是验证 **`poll`** 的存在性，二是提供关键数据，`poll` 的 `poll_id` 字段被用作 `voter_record` 账户 PDA 的种子，另外指令函数中用到了 `poll` 的时间戳。

- **`candidate`**: 候选项账户。它也必须是已存在的，且其 PDA 地址需要被验证。它必须是可变的 (`mut`)，因为投票会增加它的得票数 (`vote_amount`)。
- **`voter_record`**: 投票记录账户。这是一个新的账户，用于记录用户的投票状态。它必须是可初始化的 (`init`)。
- **`voter`**: 投票人账户。这是发起交易的签名者，负责支付 `voter_record` 账户的租金。它必须是可变的 (`mut`) 和签名者 (`Signer`)。
- **`system_program`**: Solana 系统程序，用于创建 `voter_record` 账户。

指令处理器

```rust
    pub fn vote(ctx: Context<Vote>) -> Result<()> {
        let poll = &mut ctx.accounts.poll;
        let candidate = &mut ctx.accounts.candidate;
        let voter_record = &mut ctx.accounts.voter_record;

        // 验证投票时间
        let current_timestamp = Clock::get()?.unix_timestamp as u64;
        msg!("current timestamp - {}", current_timestamp);
        if current_timestamp < poll.poll_start {
            return err!(ErrorCode::VotingNotStarted);
        }
        if current_timestamp > poll.poll_end {
            return err!(ErrorCode::VotingPeriodOver);
        }

        // 初始化 VoterRecord 账户
        voter_record.voter = ctx.accounts.voter.key();
        voter_record.poll_id = ctx.accounts.poll.poll_id;

        // 增加候选项的得票数
        candidate.vote_amount += 1;

        Ok(())
    }
```

客户端代码示例，

```tsx
import * as anchor from "@project-serum/anchor";
import { PublicKey, SystemProgram } from "@solana/web3.js";
import { Program } from "@project-serum/anchor";
import { MyProgram } from "./my-program-idls"; // 替换为你的 IDL 类型

// 假设你已经设置好了 provider 和 program
const provider = anchor.AnchorProvider.local();
anchor.setProvider(provider);
const program = anchor.workspace.MyProgram as Program<MyProgram>;

// 1. 定义必要的参数
const pollId = new anchor.BN(123); // 你要投票的 poll ID
const candidateIndex = new anchor.BN(0); // 你要投票的候选项的索引

// 2. 准备所有必需的账户公钥
async function castVote() {
  try {
    // A. 计算 poll 账户的 PDA
    const [pollAccount] = PublicKey.findProgramAddressSync(
      [anchor.utils.bytes.utf8.encode("poll"), pollId.toBuffer("le", 8)],
      program.programId
    );

    // B. 计算 candidate 账户的 PDA
    const [candidateAccount] = PublicKey.findProgramAddressSync(
      [
        anchor.utils.bytes.utf8.encode("candidate"),
        pollId.toBuffer("le", 8),
        candidateIndex.toBuffer("le", 8),
      ],
      program.programId
    );

    // C. 计算 voter_record 账户的 PDA
    // 注意这里直接使用了 provider.wallet 的公钥
    const [voterRecordAccount] = PublicKey.findProgramAddressSync(
      [
        anchor.utils.bytes.utf8.encode("voter_record"),
        pollId.toBuffer("le", 8),
        provider.wallet.publicKey.toBuffer(),
      ],
      program.programId
    );

    // 3. 调用 vote 指令并发送交易
    const tx = await program.methods
      .vote() // vote 指令没有参数
      .accounts({
        poll: pollAccount,
        candidate: candidateAccount,
        voterRecord: voterRecordAccount,
        voter: provider.wallet.publicKey, // 直接使用 provider.wallet 的公钥
        systemProgram: SystemProgram.programId,
      })
      .rpc(); // Anchor 会自动使用 provider.wallet 签名

    console.log("投票成功！");
    console.log(`交易签名: ${tx}`);
    console.log(`投票记录账户: ${voterRecordAccount.toBase58()}`);
  } catch (error) {
    console.error("投票失败:", error);
  }
}

// 在实际应用中，你需要确保 provider.wallet 已经有 SOL
// 在本地测试时，这通常是通过 airdrop 完成的
castVote();
```

[账户上下文](%E8%B4%A6%E6%88%B7%E4%B8%8A%E4%B8%8B%E6%96%87.md)

[账户的数据类型](%E8%B4%A6%E6%88%B7%E7%9A%84%E6%95%B0%E6%8D%AE%E7%B1%BB%E5%9E%8B.md)

[代码 - [lib.rs](http://lib.rs)](10%20-%20Vote%20program/%E4%BB%A3%E7%A0%81%20-%20lib%20rs%20252f2e69c7dd8094ad10f13053f82a72.md)

[代码 - test_voting.rs](%E4%BB%A3%E7%A0%81%20-%20test_voting%20rs.md)

[代码 - Cargo.toml](%E4%BB%A3%E7%A0%81%20-%20Cargo%20toml.md)

[部署到 devnet](%E9%83%A8%E7%BD%B2%E5%88%B0%20devnet.md)

---

[返回课程首页](../../README.md)
