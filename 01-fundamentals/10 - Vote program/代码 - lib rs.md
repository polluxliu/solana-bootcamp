# 代码 - lib.rs

这是最终的代码，进行了一定的重构，使得代码更加简洁。

```rust
use anchor_lang::prelude::*;

declare_id!("9aeoZT1WyTfT9cXEEDyb32orFgPsXHpQmcthjGs5Z4qf");

#[program]
pub mod voting {
    use super::*;

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

    pub fn add_candidate(ctx: Context<AddCandidate>, title: String) -> Result<()> {
        let poll = &mut ctx.accounts.poll;
        let candidate = &mut ctx.accounts.candidate;

        // 验证投票时间
        poll.expect_not_started()?;

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

    pub fn remove_candidate(ctx: Context<RemoveCandidate>) -> Result<()> {
        let poll = &mut ctx.accounts.poll;
        let candidate = &mut ctx.accounts.candidate;

        // 验证投票时间
        poll.expect_not_started()?;

        // Anchor 的 `close` 约束会自动关闭账户

        // 更新 Poll 账户的选项数量
        poll.candidate_amount -= 1;

        Ok(())
    }

    pub fn vote(ctx: Context<Vote>) -> Result<()> {
        let poll = &mut ctx.accounts.poll;
        let candidate = &mut ctx.accounts.candidate;
        let voter_record = &mut ctx.accounts.voter_record;

        // 验证投票时间
        poll.expect_in_progress()?;

        // 初始化 VoterRecord 账户
        voter_record.voter = ctx.accounts.voter.key();
        voter_record.poll_id = ctx.accounts.poll.poll_id;

        // 增加候选项的得票数
        candidate.vote_amount += 1;

        Ok(())
    }
}

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
        seeds = [b"candidate", poll.poll_id.to_le_bytes().as_ref(), poll.next_index.to_le_bytes().as_ref()],
        bump
    )]
    pub candidate: Account<'a, Candidate>,

    #[account(mut)]
    pub creator: Signer<'a>,

    pub system_program: Program<'a, System>,
}

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

#[account]
#[derive(InitSpace, Debug)]
pub struct Candidate {
    pub poll_id: u64,
    pub index: u64,
    #[max_len(128)]
    pub title: String,
    pub vote_amount: u64,
}

#[account]
#[derive(InitSpace, Debug)]
pub struct VoterRecord {
    pub voter: Pubkey,
    pub poll_id: u64,
}

#[error_code]
pub enum ErrorCode {
    #[msg("Voting is not open yet.")]
    VotingNotStarted,
    #[msg("Voting period has ended.")]
    VotingPeriodOver,
    #[msg("Voting is in progress.")]
    VotingInProgress,
}

impl Poll {
    pub fn expect_not_started(&self) -> Result<()> {
        let current_timestamp = Clock::get()?.unix_timestamp as u64;

        msg!("current timestamp - {}", current_timestamp);
        msg!(
            "poll start timestamp - {}, poll end timestamp - {}",
            self.poll_start,
            self.poll_end
        );

        if current_timestamp >= self.poll_start {
            return if current_timestamp <= self.poll_end {
                err!(ErrorCode::VotingInProgress)
            } else {
                err!(ErrorCode::VotingPeriodOver)
            };
        }

        Ok(())
    }

    pub fn expect_in_progress(&self) -> Result<()> {
        let current_timestamp = Clock::get()?.unix_timestamp as u64;

        msg!("current timestamp - {}", current_timestamp);
        msg!(
            "poll start timestamp - {}, poll end timestamp - {}",
            self.poll_start,
            self.poll_end
        );

        if current_timestamp < self.poll_start {
            return err!(ErrorCode::VotingNotStarted);
        }
        if current_timestamp > self.poll_end {
            return err!(ErrorCode::VotingPeriodOver);
        }

        Ok(())
    }
}
```

---

[返回课程首页](../../README.md)
