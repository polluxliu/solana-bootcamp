# place_bid

英式拍卖，由买家发起竞拍。

注意，这里需要将具体的拍卖活动的 pda 传入。

```tsx
it("place_bid", async () => {
        // 拍卖活动账户
        const auctionPda = new anchor.web3.PublicKey("FjjCFkiy7ui6W8rzLRXnz1gsfJiog9dgLEsUZKfSV6TF");

        // 获取最新 auctionId
        const auctionHouseAccount = await program.account.auctionHouse.fetch(auctionHousePda);
        const settlementMint = auctionHouseAccount.settlementMint;

        // 出价资金托管账户
        const [escrowBidTa] = anchor.web3.PublicKey.findProgramAddressSync(
            [Buffer.from("escrow"), auctionPda.toBuffer()],
            program.programId
        );

        // 出价人资金账户 ATA
        const bidderAta = anchor.utils.token.associatedAddress({
            mint: settlementMint,
            owner: bidderKeypair.publicKey,
        });

        // refund 账户（上一个最高出价人的 ATA）
        let refundAta: anchor.web3.PublicKey;
        const auctionAccount = await program.account.auction.fetch(auctionPda);
        if (auctionAccount.englishHighestBid.toNumber() === 0) {
            // 第一次出价，不需要退款
            // 这里直接使用竞拍人的账户作为退款账户
            refundAta = bidderAta;
        } else {
            refundAta = anchor.utils.token.associatedAddress({
                mint: settlementMint,
                owner: auctionAccount.englishHighestBidder,
            });
        }

        // -------- 出价前的余额快照 --------
        const bidderAtaBefore = await getAccount(anchor.getProvider().connection, bidderAta);
        const escrowBidTaBefore = await getAccount(anchor.getProvider().connection, escrowBidTa);

        console.log("出价前余额:");
        console.log("- 出价人 ATA:", bidderAtaBefore.amount.toString());
        console.log("- 托管账户:", escrowBidTaBefore.amount.toString());
        console.log("- 当前最高价:", auctionAccount.englishHighestBid.toString());

        // 7. 构造 place_bid 指令
        const bidAmount = new anchor.BN(600); // 出价金额

        const tx = await program.methods
            .placeBid(bidAmount)
            .accounts({
                auctionHouse: auctionHousePda,
                auction: auctionPda,
                escrowBidTa,
                bidder: bidderKeypair.publicKey,
                bidderAta,
                refundAta,
                tokenProgram: anchor.utils.token.TOKEN_PROGRAM_ID,
            }).transaction();

        tx.feePayer = bidderKeypair.publicKey;
        const txSig = await anchor.web3.sendAndConfirmTransaction(
            anchor.getProvider().connection,
            tx,
            [bidderKeypair],
        );

        // -------- 出价后的余额快照 --------
        const auctionAccountAfter = await program.account.auction.fetch(auctionPda);
        const bidderAtaAfter = await getAccount(anchor.getProvider().connection, bidderAta);
        const escrowBidTaAfter = await getAccount(anchor.getProvider().connection, escrowBidTa);

        console.log("出价后余额:");
        console.log("- 出价人 ATA:", bidderAtaAfter.amount.toString());
        console.log("- 托管账户:", escrowBidTaAfter.amount.toString());
        console.log("- 当前最高价:", auctionAccountAfter.englishHighestBid.toString());
        console.log("- 当前最高出价人:", auctionAccountAfter.englishHighestBidder.toBase58());

        // -------- chai 断言 --------
        expect(auctionAccountAfter.englishHighestBid.toString()).to.equal(bidAmount.toString());
        expect(auctionAccountAfter.englishHighestBidder.toBase58()).to.equal(bidderKeypair.publicKey.toBase58());

        expect(Number(escrowBidTaAfter.amount)).to.equal(Number(bidAmount));
    });
```

---

[返回课程首页](../../../../README.md)
