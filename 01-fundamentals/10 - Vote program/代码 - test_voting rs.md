# 代码 - test_voting.rs

每一个测试函数都包含 4 部分：

1. **准备测试用的样例数据**
2. **准备指令的上下文账户**
3. **准备指令**
4. **实际执行**

```rust
#![cfg(feature = "test-sbf")]

use {
    anchor_lang::{
        prelude::SolanaSysvar,
        solana_program::{
            instruction::Instruction, native_token::LAMPORTS_PER_SOL, pubkey::Pubkey, rent::Rent,
            sysvar::clock,
        },
        AnchorDeserialize, AnchorSerialize, Discriminator, InstructionData, ToAccountMetas,
    },
    bincode,
    mollusk_svm::{program, result::Check, Mollusk},
    solana_sdk::account::Account,
    std::time::{SystemTime, UNIX_EPOCH},
};
```

```rust
#[test]
fn test_create_poll() {
    let program_id = voting::id();

    let mollusk = Mollusk::new(&program_id, "voting");

    /******* Hardcode Arguments *******/

    let poll_id: u64 = 12345;
    let title = String::from("Favorite Programming Language");
    let description = String::from("Vote for your favorite programming language.");
    let poll_start: u64 = 1755100800;
    let poll_end: u64 = 1755104400;

    /******* Accounts *******/

    // Program Account
    let (system_program_pubkey, system_program_account) =
        program::keyed_account_for_system_program();

    // Data Account

    // ---poll account
    let (poll_pubkey, _bump) =
        Pubkey::find_program_address(&[b"poll", poll_id.to_le_bytes().as_ref()], &program_id);
    let poll_account = Account::default();

    // System Account
    let creator_pubkey = Pubkey::new_unique();
    let creator_account = Account::new(LAMPORTS_PER_SOL / 2, 0, &system_program_pubkey);

    /******* Instruction *******/

    let instruction = Instruction::new_with_bytes(
        program_id,
        &voting::instruction::CreatePoll {
            poll_id,
            title,
            description,
            poll_start,
            poll_end,
        }
        .data(),
        voting::accounts::CreatePoll {
            poll: poll_pubkey,
            creator: creator_pubkey,
            system_program: system_program_pubkey,
        }
        .to_account_metas(None),
    );

    let accounts = vec![
        (poll_pubkey, poll_account),
        (creator_pubkey, creator_account),
        (system_program_pubkey, system_program_account),
    ];

    /******* Execution *******/

    let result =
        mollusk.process_and_validate_instruction(&instruction, &accounts, &[Check::success()]);
}
```

```rust
#[test]
fn test_add_candidate() {
    let program_id = voting::id();

    let mollusk = Mollusk::new(&program_id, "voting");

    /******* Hardcode Arguments *******/

    let poll_id: u64 = 12345;
    let title = String::from("Favorite Programming Language");
    let description = String::from("Vote for your favorite programming language.");
    let poll_start: u64 = 1855402331;
    let poll_end: u64 = 1855452331;

    let candidate_index_1: u64 = 0;
    let candidate_title_1 = String::from("Rust");

    let candidate_index_2: u64 = 1;
    let candidate_title_2 = String::from("TypeScript");

    /******* Accounts *******/

    // Program Account
    let (system_program_pubkey, system_program_account) =
        program::keyed_account_for_system_program();

    // Sysvar Account

    // ---Clock Sysvar
    let (clock_pubkey, clock_account) = create_clock_sysvar();

    // System Account
    let creator_pubkey = Pubkey::new_unique();
    let creator_account = Account::new(LAMPORTS_PER_SOL / 2, 0, &system_program_pubkey);

    // Data Account

    // ---poll account

    // 1. 创建 Poll 结构体实例
    let poll_data = voting::Poll {
        poll_id,
        creator: creator_pubkey,
        title,
        description,
        poll_start,
        poll_end,
        candidate_amount: 0,
        next_index: 0,
    };

    // 2. 将 Poll 结构体序列化为字节向量
    // ---获取 discriminator
    let mut poll_data_bytes = voting::Poll::DISCRIMINATOR.to_vec();
    // ---序列化 poll_data
    let mut serialized_data = poll_data.try_to_vec().unwrap();
    // ---追加 poll_data 字节
    poll_data_bytes.append(&mut serialized_data);

    // 3. 计算账户的 PDA
    let (poll_pubkey, _bump) =
        Pubkey::find_program_address(&[b"poll", poll_id.to_le_bytes().as_ref()], &program_id);

    // 4. 创建账户
    let poll_account = Account {
        lamports: Rent::default().minimum_balance(poll_data_bytes.len()),
        data: poll_data_bytes,
        owner: program_id,
        executable: false,
        rent_epoch: 0,
    };

    // ---candidate account

    // 1. candidate 1
    let (candidate_pubkey_1, _bump) = Pubkey::find_program_address(
        &[
            b"candidate",
            poll_id.to_le_bytes().as_ref(),
            candidate_index_1.to_le_bytes().as_ref(),
        ],
        &program_id,
    );
    let candidate_account_1 = Account::default();

    // 2. candidate 2
    let (candidate_pubkey_2, _bump) = Pubkey::find_program_address(
        &[
            b"candidate",
            poll_id.to_le_bytes().as_ref(),
            candidate_index_2.to_le_bytes().as_ref(),
        ],
        &program_id,
    );
    let candidate_account_2 = Account::default();

    /******* Instruction *******/

    // 1. instruction 1
    let instruction_1 = Instruction::new_with_bytes(
        program_id,
        &voting::instruction::AddCandidate {
            title: candidate_title_1,
        }
        .data(),
        voting::accounts::AddCandidate {
            poll: poll_pubkey,
            candidate: candidate_pubkey_1,
            creator: creator_pubkey,
            system_program: system_program_pubkey,
        }
        .to_account_metas(None),
    );

    // 2. instruction 2
    let instruction_2 = Instruction::new_with_bytes(
        program_id,
        &voting::instruction::AddCandidate {
            title: candidate_title_2,
        }
        .data(),
        voting::accounts::AddCandidate {
            poll: poll_pubkey,
            candidate: candidate_pubkey_2,
            creator: creator_pubkey,
            system_program: system_program_pubkey,
        }
        .to_account_metas(None),
    );

    // 3. All accounts needed
    let accounts = vec![
        (poll_pubkey, poll_account),
        (candidate_pubkey_1, candidate_account_1),
        (candidate_pubkey_2, candidate_account_2),
        (creator_pubkey, creator_account),
        (system_program_pubkey, system_program_account),
        (clock_pubkey, clock_account),
    ];

    /******* Execution *******/

    let checks_1 = vec![Check::success()];
    let checks_2 = vec![Check::success()];

    let result = mollusk.process_and_validate_instruction_chain(
        &[(&instruction_1, &checks_1), (&instruction_2, &checks_2)],
        &accounts,
    );

    // validate poll_account
    let poll_account = result.get_account(&poll_pubkey).unwrap();
    println!("{:#?}", poll_account);
    let mut poll_data_bytes = &poll_account.data[8..]; // 先得到 &mut &[u8]
    let poll_data = voting::Poll::deserialize(&mut poll_data_bytes)
        .expect("Failed to deserialize poll account data");
    println!("{:#?}", poll_data);

    // validate candidate_account_1
    let candidate_account_1 = result.get_account(&candidate_pubkey_1).unwrap();
    println!("{:#?}", candidate_account_1);
    let candidate_1_data = voting::Candidate::deserialize(&mut &candidate_account_1.data[8..])
        .expect("Failed to deserialize candidate account data");
    println!("{:#?}", candidate_1_data);

    // validate candidate_account_2
    let candidate_account_2 = result.get_account(&candidate_pubkey_2).unwrap();
    println!("{:#?}", candidate_account_2);
    let candidate_2_data = voting::Candidate::deserialize(&mut &candidate_account_2.data[8..])
        .expect("Failed to deserialize candidate account data");
    println!("{:#?}", candidate_2_data);
}
```

```rust
#[test]
fn test_remove_candidate() {
    let program_id = voting::id();

    let mollusk = Mollusk::new(&program_id, "voting");

    /******* Hardcode Arguments *******/

    let poll_id: u64 = 12345;
    let title = String::from("Favorite Programming Language");
    let description = String::from("Vote for your favorite programming language.");
    let poll_start: u64 = 1855402331;
    let poll_end: u64 = 1855452331;

    let candidate_index_1: u64 = 0;
    let candidate_title_1 = String::from("Rust");

    let candidate_index_2: u64 = 1;
    let candidate_title_2 = String::from("TypeScript");

    /******* Accounts *******/

    // Program Account
    let (system_program_pubkey, system_program_account) =
        program::keyed_account_for_system_program();

    // Sysvar Account

    // ---Clock Sysvar
    let (clock_pubkey, clock_account) = create_clock_sysvar();

    // System Account
    let creator_pubkey = Pubkey::new_unique();
    let creator_account = Account::new(LAMPORTS_PER_SOL / 2, 0, &system_program_pubkey);

    // Data Account

    // ---poll account

    // 1. 创建 Poll 结构体实例
    let poll_data = voting::Poll {
        poll_id,
        creator: creator_pubkey,
        title,
        description,
        poll_start,
        poll_end,
        candidate_amount: 2,
        next_index: 2,
    };

    // 2. 将 Poll 结构体序列化为字节向量
    // ---获取 discriminator
    let mut poll_data_bytes = voting::Poll::DISCRIMINATOR.to_vec();
    // ---序列化 poll_data
    let mut serialized_data = poll_data.try_to_vec().unwrap();
    // ---追加 poll_data 字节
    poll_data_bytes.append(&mut serialized_data);

    // 3. 计算账户的 PDA
    let (poll_pubkey, _bump) =
        Pubkey::find_program_address(&[b"poll", poll_id.to_le_bytes().as_ref()], &program_id);

    // 4. 创建账户
    let poll_account = Account {
        lamports: Rent::default().minimum_balance(poll_data_bytes.len()),
        data: poll_data_bytes,
        owner: program_id,
        executable: false,
        rent_epoch: 0,
    };

    // ---candidate account

    // 1. 创建 Candidate 结构体实例
    let candidate_data_1 = voting::Candidate {
        poll_id,
        index: candidate_index_1,
        title: candidate_title_1,
        vote_amount: 0,
    };

    // 2. 将 Candidate 结构体序列化为字节向量
    // ---获取 discriminator
    let mut candidate_data_bytes_1 = voting::Candidate::DISCRIMINATOR.to_vec();
    // ---序列化 poll_data
    let mut serialized_data = candidate_data_1.try_to_vec().unwrap();
    // ---追加 poll_data 字节
    candidate_data_bytes_1.append(&mut serialized_data);

    // 3. 计算账户的 PDA
    let (candidate_pubkey_1, _bump) = Pubkey::find_program_address(
        &[
            b"candidate",
            poll_id.to_le_bytes().as_ref(),
            candidate_index_1.to_le_bytes().as_ref(),
        ],
        &program_id,
    );

    // 4. 创建账户
    let candidate_account_1 = Account {
        lamports: Rent::default().minimum_balance(candidate_data_bytes_1.len()),
        data: candidate_data_bytes_1,
        owner: program_id,
        executable: false,
        rent_epoch: 0,
    };

    /******* Instruction *******/

    let instruction = Instruction::new_with_bytes(
        program_id,
        &voting::instruction::RemoveCandidate {}.data(),
        voting::accounts::RemoveCandidate {
            poll: poll_pubkey,
            candidate: candidate_pubkey_1,
            creator: creator_pubkey,
            system_program: system_program_pubkey,
        }
        .to_account_metas(None),
    );

    let accounts = vec![
        (poll_pubkey, poll_account),
        (candidate_pubkey_1, candidate_account_1),
        (creator_pubkey, creator_account),
        (system_program_pubkey, system_program_account),
        (clock_pubkey, clock_account),
    ];

    /******* Execution *******/

    let result =
        mollusk.process_and_validate_instruction(&instruction, &accounts, &[Check::success()]);

    // validate poll_account
    let poll_account = result.get_account(&poll_pubkey).unwrap();
    println!("{:#?}", poll_account);
    let mut poll_data_bytes = &poll_account.data[8..]; // 先得到 &mut &[u8]
    let poll_data = voting::Poll::deserialize(&mut poll_data_bytes)
        .expect("Failed to deserialize poll account data");
    println!("{:#?}", poll_data);
}
```

```rust
#[test]
fn test_vote() {
    let program_id = voting::id();

    let mollusk = Mollusk::new(&program_id, "voting");

    /******* Hardcode Arguments *******/

    let poll_id: u64 = 12345;
    let title = String::from("Favorite Programming Language");
    let description = String::from("Vote for your favorite programming language.");
    let poll_start: u64 = 1655452331;
    let poll_end: u64 = 1855452331;

    let candidate_index_1: u64 = 0;
    let candidate_title_1 = String::from("Rust");

    /******* Accounts *******/

    // Program Account
    let (system_program_pubkey, system_program_account) =
        program::keyed_account_for_system_program();

    // Sysvar Account

    // ---Clock Sysvar
    let (clock_pubkey, clock_account) = create_clock_sysvar();

    // System Account

    let creator_pubkey = Pubkey::new_unique();
    let creator_account = Account::new(LAMPORTS_PER_SOL / 2, 0, &system_program_pubkey);

    let voter_pubkey = Pubkey::new_unique();
    let voter_account = Account::new(LAMPORTS_PER_SOL / 2, 0, &system_program_pubkey);

    // Data Account

    // ---poll account

    // 1. 创建 Poll 结构体实例
    let poll_data = voting::Poll {
        poll_id,
        creator: creator_pubkey,
        title,
        description,
        poll_start,
        poll_end,
        candidate_amount: 1,
        next_index: 1,
    };

    // 2. 将 Poll 结构体序列化为字节向量
    // ---获取 discriminator
    let mut poll_data_bytes = voting::Poll::DISCRIMINATOR.to_vec();
    // ---序列化 poll_data
    let mut serialized_data = poll_data.try_to_vec().unwrap();
    // ---追加 poll_data 字节
    poll_data_bytes.append(&mut serialized_data);

    // 3. 计算账户的 PDA
    let (poll_pubkey, _bump) =
        Pubkey::find_program_address(&[b"poll", poll_id.to_le_bytes().as_ref()], &program_id);

    // 4. 创建账户
    let poll_account = Account {
        lamports: Rent::default().minimum_balance(poll_data_bytes.len()),
        data: poll_data_bytes,
        owner: program_id,
        executable: false,
        rent_epoch: 0,
    };

    // ---candidate account

    // 1. 创建 Candidate 结构体实例
    let candidate_data = voting::Candidate {
        poll_id,
        index: candidate_index_1,
        title: candidate_title_1,
        vote_amount: 0,
    };

    // 2. 将 Candidate 结构体序列化为字节向量
    // ---获取 discriminator
    let mut candidate_data_bytes = voting::Candidate::DISCRIMINATOR.to_vec();
    // ---序列化 poll_data
    let mut serialized_data = candidate_data.try_to_vec().unwrap();
    // ---追加 poll_data 字节
    candidate_data_bytes.append(&mut serialized_data);

    // 3. 计算账户的 PDA
    let (candidate_pubkey, _bump) = Pubkey::find_program_address(
        &[
            b"candidate",
            poll_id.to_le_bytes().as_ref(),
            candidate_index_1.to_le_bytes().as_ref(),
        ],
        &program_id,
    );

    // 4. 创建账户
    let candidate_account = Account {
        lamports: Rent::default().minimum_balance(candidate_data_bytes.len()),
        data: candidate_data_bytes,
        owner: program_id,
        executable: false,
        rent_epoch: 0,
    };

    // ---voter_record account

    let (voter_record_pubkey, _bump) = Pubkey::find_program_address(
        &[
            b"voter_record",
            poll_id.to_le_bytes().as_ref(),
            voter_pubkey.as_ref(),
        ],
        &program_id,
    );
    let voter_record_account = Account::default();

    /******* Instruction *******/

    let instruction = Instruction::new_with_bytes(
        program_id,
        &voting::instruction::Vote {}.data(),
        voting::accounts::Vote {
            poll: poll_pubkey,
            candidate: candidate_pubkey,
            voter_record: voter_record_pubkey,
            voter: voter_pubkey,
            system_program: system_program_pubkey,
        }
        .to_account_metas(None),
    );

    let accounts = vec![
        (poll_pubkey, poll_account),
        (candidate_pubkey, candidate_account),
        (voter_record_pubkey, voter_record_account),
        (voter_pubkey, voter_account),
        (system_program_pubkey, system_program_account),
        (clock_pubkey, clock_account), // 将 Clock 账户添加到指令账户列表中
    ];

    /******* Execution *******/

    let result =
        mollusk.process_and_validate_instruction(&instruction, &accounts, &[Check::success()]);

    // validate candidate_account
    let candidate_account = result.get_account(&candidate_pubkey).unwrap();
    println!("{:#?}", candidate_account);
    let mut candidate_data_bytes = &candidate_account.data[8..]; // 先得到 &mut &[u8]
    let candidate_data = voting::Candidate::deserialize(&mut candidate_data_bytes)
        .expect("Failed to deserialize candidate account data");
    println!("{:#?}", candidate_data);

    assert_eq!(candidate_data.vote_amount, 1);

    // validate voter_record account
    let voter_record_account = result.get_account(&voter_record_pubkey).unwrap();
    println!("{:#?}", voter_record_account);
    let mut voter_record_data_bytes = &voter_record_account.data[8..]; // 先得到 &mut &[u8]
    let voter_record_data = voting::VoterRecord::deserialize(&mut voter_record_data_bytes)
        .expect("Failed to deserialize candidate account data");
    println!("{:#?}", voter_record_data);

    assert_eq!(voter_record_data.voter, voter_pubkey);
}
```

```rust
/// 创建一个带有模拟时间戳的 Clock Sysvar 账户
fn create_clock_sysvar() -> (Pubkey, Account) {
    // 获取当前 Unix 时间戳
    let current_unix_timestamp = SystemTime::now()
        .duration_since(UNIX_EPOCH)
        .unwrap()
        .as_secs();

    // 创建 Clock 结构体实例
    let clock_data = clock::Clock {
        unix_timestamp: current_unix_timestamp as i64,
        ..Default::default()
    };

    // 序列化 Clock 实例为字节
    let clock_data_bytes = bincode::serialize(&clock_data).unwrap();

    // 创建账户
    let clock_account = Account {
        lamports: Rent::default().minimum_balance(clock::Clock::size_of()),
        data: clock_data_bytes,
        owner: clock::ID,
        executable: false,
        rent_epoch: 0,
    };

    // 返回账户的公钥和账户本身
    (clock::ID, clock_account)
}
```

---

[返回课程首页](../../README.md)
