# create_auction_house

由 **拍卖行管理员** 创建拍卖行。

# 测试代码

平台创建和添加支持代币时的指令，我们使用 Solana CLI 配置文件中指定的默认 wallet 来签名的，在这里创建拍卖行时的指令，需要由拍卖行管理员账户来签名。

```tsx
    it("create_auction_house", async () => {
        // 平台管理员账户（默认钱包）的 SOL 余额
        const platformAdminSolBefore = await anchor.getProvider().connection.getBalance(anchor.getProvider().wallet.publicKey);
        // 平台 ATA 中的代币余额
        const platformTokenBefore = await getAccount(anchor.getProvider().connection, platformTreasuryAta);
        // 拍卖行管理员账户的 SOL 余额
        const houseAdminSolBefore = await anchor.getProvider().connection.getBalance(auctionHouseAdminKeypair.publicKey);
        // 拍卖行管理员 ATA 中的代币余额
        const houseAdminTokenBefore = await getAccount(anchor.getProvider().connection, authorityAta);

        console.log("拍卖行创建前余额:");
        console.log("- 平台管理员账户的 SOL：", platformAdminSolBefore.toString());
        console.log("- 平台 ATA 的代币：", platformTokenBefore.amount.toString());
        console.log("- 拍卖行管理员账户的 SOL：", houseAdminSolBefore.toString());
        console.log("- 拍卖行管理员 ATA 的代币：", houseAdminTokenBefore.amount.toString());

        // const txSig = await program.methods
        //     .createAuctionHouse({feeRate: 200}) // 2%
        //     .accounts({
        //         platform: platformPda,
        //         auctionHouse: auctionHousePda,
        //         settlementMint: mintPk,
        //         auctionHouseTreasury: auctionHouseTreasuryAta,
        //         authorityAta,
        //         platformTreasuryAta,
        //         authority: auctionHouseAdminKeypair.publicKey,
        //         systemProgram: anchor.web3.SystemProgram.programId,
        //         tokenProgram: anchor.utils.token.TOKEN_PROGRAM_ID,
        //         associatedTokenProgram: anchor.utils.token.ASSOCIATED_PROGRAM_ID,
        //     }).signers([auctionHouseAdminKeypair]).rpc();

        const tx = await program.methods
            .createAuctionHouse({feeRate: 200}) // 2%
            .accounts({
                platform: platformPda,
                auctionHouse: auctionHousePda,
                settlementMint: mintPk,
                auctionHouseTreasury: auctionHouseTreasuryAta,
                authorityAta,
                platformTreasuryAta,
                authority: auctionHouseAdminKeypair.publicKey,
                systemProgram: anchor.web3.SystemProgram.programId,
                tokenProgram: anchor.utils.token.TOKEN_PROGRAM_ID,
                associatedTokenProgram: anchor.utils.token.ASSOCIATED_PROGRAM_ID,
            }).transaction();

        tx.feePayer = auctionHouseAdminKeypair.publicKey;
        const txSig = await anchor.web3.sendAndConfirmTransaction(
            anchor.getProvider().connection,
            tx,
            [auctionHouseAdminKeypair],
        );

        // 平台管理员账户（默认钱包）的 SOL 余额
        const platformAdminSolAfter = await anchor.getProvider().connection.getBalance(anchor.getProvider().wallet.publicKey);
        // 平台 ATA 中的代币余额
        const platformTokenAfter = await getAccount(anchor.getProvider().connection, platformTreasuryAta);
        // 拍卖行管理员账户的 SOL 余额
        const houseAdminSolAfter = await anchor.getProvider().connection.getBalance(auctionHouseAdminKeypair.publicKey);
        // 拍卖行管理员 ATA 中的代币余额
        const houseAdminTokenAfter = await getAccount(anchor.getProvider().connection, authorityAta);

        console.log("拍卖行创建后余额:");
        console.log("- 平台管理员账户的 SOL：", platformAdminSolAfter.toString());
        console.log("- 平台 ATA 的代币：", platformTokenAfter.amount.toString());
        console.log("- 拍卖行管理员账户的 SOL：", houseAdminSolAfter.toString());
        console.log("- 拍卖行管理员 ATA 的代币：", houseAdminTokenAfter.amount.toString());
    });
```

重点看一下交易的构造和发送。

```tsx
        const txSig = await program.methods
            .createAuctionHouse({feeRate: 200}) // 2%
            .accounts({
                platform: platformPda,
                auctionHouse: auctionHousePda,
                settlementMint: mintPk,
                auctionHouseTreasury: auctionHouseTreasuryAta,
                authorityAta,
                platformTreasuryAta,
                authority: auctionHouseAdminKeypair.publicKey,
                systemProgram: anchor.web3.SystemProgram.programId,
                tokenProgram: anchor.utils.token.TOKEN_PROGRAM_ID,
                associatedTokenProgram: anchor.utils.token.ASSOCIATED_PROGRAM_ID,
            }).signers([auctionHouseAdminKeypair]).rpc();
```

以上代码使用的是 `.rpc()` 方法。该方法将使用 provider.wallet 作为 fee-payer，同时收集 provider.wallet 和 auctionHouseAdminKeypair（通过 `.signers(…)` 指定）两个签名，这就意味着将由平台管理员账户支付交易费。但我们需要的是由拍卖行管理员支付交易费，因此我们需要手动构造和发送交易。

```tsx
        const tx = await program.methods
            .createAuctionHouse({feeRate: 200}) // 2%
            .accounts({
                platform: platformPda,
                auctionHouse: auctionHousePda,
                settlementMint: mintPk,
                auctionHouseTreasury: auctionHouseTreasuryAta,
                authorityAta,
                platformTreasuryAta,
                authority: auctionHouseAdminKeypair.publicKey,
                systemProgram: anchor.web3.SystemProgram.programId,
                tokenProgram: anchor.utils.token.TOKEN_PROGRAM_ID,
                associatedTokenProgram: anchor.utils.token.ASSOCIATED_PROGRAM_ID,
            }).transaction();

        tx.feePayer = auctionHouseAdminKeypair.publicKey;
        const txSig = await anchor.web3.sendAndConfirmTransaction(
            anchor.getProvider().connection,
            tx,
            [auctionHouseAdminKeypair],
        );
```

- 通过 `tx.feePayer = auctionHouseAdminKeypair.publicKey` 指定 fee-payer 为平台管理员。
- `sendAndConfirmTransaction` 中的第三个参数中指定所有需要签名的账户的 Keypair，包括 fee-payer。由于这里的 fee-payer 以及 Rust 链上程序中要求的 Signer 都是拍卖行管理员账户，因此这里的数组中只有拍卖行管理员的 Keypair。

    💡
    
    fee-payer 必须是签名人。仅仅把账户设为 fee-payer 并 **不会自动签名**，你还需要在客户端提供该账户的私钥（Keypair）来生成签名，否则交易会失败。
    
    传给 `sendAndConfirmTransaction` 的私钥只在本地用来计算签名，真正发到链上的只有交易本身 和 签名。

这个 create_auction_house 测试用例执行后的输出如下：

```
  auction
拍卖行创建前余额:
- 平台管理员账户的 SOL： 499999997935275260
- 平台 ATA 的代币： 0
- 拍卖行管理员账户的 SOL： 10000000000
- 拍卖行管理员 ATA 的代币： 10000000000
拍卖行创建后余额:
- 平台管理员账户的 SOL： 499999997935275260
- 平台 ATA 的代币： 1000000
- 拍卖行管理员账户的 SOL： 9996264440
- 拍卖行管理员 ATA 的代币： 9999000000
    ✔ create_auction_house (531ms)
```

可见，平台管理员账户中的 SOL 余额并没有变化。拍卖行管理员账户中的 SOL 余额被用来支付交易费和新账户的租金。拍卖行管理员账户的 ATA 中的代币余额转了 `1000000` 给拍卖行管理员账户的 ATA，用来支付一次性的拍卖行创建费。

---

[返回课程首页](../../../../README.md)
