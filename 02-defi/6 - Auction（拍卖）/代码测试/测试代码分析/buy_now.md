# buy_now

荷兰拍卖，由买家发起竞拍。

注意，这里需要将具体的拍卖活动的 pda 传入。

```tsx
    it("buy_now", async () => {
        // 拍卖账户
        const auctionPda = new anchor.web3.PublicKey("BHfKweN5MxjitwnmCs6QJ3iVwPsZmFfsw6FXvn97ySJM");

        // 获取最新 auctionId
        const auctionHouseAccount = await program.account.auctionHouse.fetch(auctionHousePda);
        const settlementMint = auctionHouseAccount.settlementMint;

        // 出价资金托管账户
        const [escrowBidTa] = anchor.web3.PublicKey.findProgramAddressSync(
            [Buffer.from("escrow"), auctionPda.toBuffer()],
            program.programId
        );

        // 买家资金账户 ATA
        const buyerAta = anchor.utils.token.associatedAddress({
            mint: settlementMint,
            owner: bidderKeypair.publicKey,
        });

        // -------- 买断前的余额快照 --------
        const auctionAccountBefore = await program.account.auction.fetch(auctionPda);
        const buyerAtaBefore = await getAccount(anchor.getProvider().connection, buyerAta);
        const escrowBidTaBefore = await getAccount(anchor.getProvider().connection, escrowBidTa);

        console.log("买断前状态:");
        console.log("- 买家 ATA:", buyerAtaBefore.amount.toString());
        console.log("- 托管账户:", escrowBidTaBefore.amount.toString());
        console.log("- 拍卖状态:", auctionAccountBefore.status);
        console.log("- 最低价:", auctionAccountBefore.dutchEndPrice.toString());

        // -------- 构造 buy_now 指令 --------
        const tx = await program.methods
            .buyNow()
            .accounts({
                auctionHouse: auctionHousePda,
                auction: auctionPda,
                escrowBidTa,
                buyer: bidderKeypair.publicKey,
                buyerAta,
                tokenProgram: anchor.utils.token.TOKEN_PROGRAM_ID,
            }).transaction();

        tx.feePayer = bidderKeypair.publicKey;
        const txSig = await anchor.web3.sendAndConfirmTransaction(
            anchor.getProvider().connection,
            tx,
            [bidderKeypair],
        );

        // -------- 买断后的余额快照 --------
        const auctionAccountAfter = await program.account.auction.fetch(auctionPda);
        const buyerAtaAfter = await getAccount(anchor.getProvider().connection, buyerAta);
        const escrowBidTaAfter = await getAccount(anchor.getProvider().connection, escrowBidTa);

        console.log("买断后状态:");
        console.log("- 买家 ATA:", buyerAtaAfter.amount.toString());
        console.log("- 托管账户:", escrowBidTaAfter.amount.toString());
        console.log("- 拍卖状态:", auctionAccountAfter.status);
        console.log("- 成交价:", auctionAccountAfter.finalPrice.toString());
        console.log("- 买家:", auctionAccountAfter.buyer.toBase58());
    });
```

---

[返回课程首页](../../../../README.md)
