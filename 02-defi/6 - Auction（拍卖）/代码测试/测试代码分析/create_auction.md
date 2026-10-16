# create_auction

由卖家创建拍卖活动。

# 测试代码

创建英式拍卖：

```tsx
    it("create_english_auction", async () => {
        // 拍卖 ID（从链上 auction_house.next_auction_id 取）
        const auctionHouseAccount = await program.account.auctionHouse.fetch(auctionHousePda);
        const nextAuctionId = auctionHouseAccount.nextAuctionId;
        const settlementMint = auctionHouseAccount.settlementMint;

        // 拍卖账户
        const [auctionPda, _auctionBump] = anchor.web3.PublicKey.findProgramAddressSync(
            [
                Buffer.from("auction"),
                auctionHousePda.toBuffer(),
                new anchor.BN(nextAuctionId).toArrayLike(Buffer, "le", 8)
            ],
            program.programId
        );

        // 拍品托管 ATA
        const escrowAssetAta = anchor.utils.token.associatedAddress({
            mint: nftPk,
            owner: auctionPda,
        });

        // 出价资金托管 Token Account
        const [escrowBidTa, _escrowBidBump] = anchor.web3.PublicKey.findProgramAddressSync(
            [
                Buffer.from("escrow"),
                auctionPda.toBuffer()
            ],
            program.programId
        );

        const now = Math.floor(Date.now() / 1000);
        const startTime = new anchor.BN(now + 60); // 1分钟后开始
        const endTime = new anchor.BN(now + 3600); // 1小时后结束

        // --------- 执行前的状态 ---------
        const sellerAtaBefore = await getAccount(anchor.getProvider().connection, sellerNFTAta);
        console.log("Seller ATA balance (before):", sellerAtaBefore.amount.toString());

        let escrowAssetAtaBefore: Account;
        try {
            escrowAssetAtaBefore = await getAccount(anchor.getProvider().connection, escrowAssetAta);
            console.log("Escrow ATA balance (before):", escrowAssetAtaBefore.amount.toString());
        } catch {
            console.log("Escrow ATA does not exist yet.");
        }

        // 英式拍卖参数
        const args = {
            amount: new anchor.BN(1),
            startPrice: new anchor.BN(100),
            startTime,
            endTime,
            auctionType: 0,

            englishReservePrice: new anchor.BN(200),

            dutchPriceDropInterval: new anchor.BN(0),
            dutchPriceDropAmount: new anchor.BN(0),
            dutchEndPrice: new anchor.BN(0),
        };

        const tx = await program.methods
            .createAuction(args)
            .accounts({
                platform: platformPda,
                auctionHouseAuthority: auctionHouseAdminKeypair.publicKey,
                settlementMintAccount: settlementMint,
                auctionHouse: auctionHousePda,
                seller: sellerKeypair.publicKey,
                assetMintAccount: nftPk,
                sellerAssetAta: sellerNFTAta,
                escrowAssetAta,
                escrowBidTa,
                auction: auctionPda,
                systemProgram: anchor.web3.SystemProgram.programId,
                tokenProgram: anchor.utils.token.TOKEN_PROGRAM_ID,
                associatedTokenProgram: anchor.utils.token.ASSOCIATED_PROGRAM_ID,
            }).transaction();

        tx.feePayer = sellerKeypair.publicKey;
        const txSig = await anchor.web3.sendAndConfirmTransaction(
            anchor.getProvider().connection,
            tx,
            [sellerKeypair],
        );

        // --------- 执行后的状态 ---------
        const sellerAtaAfter = await getAccount(anchor.getProvider().connection, sellerNFTAta);
        console.log("Seller ATA balance (after):", sellerAtaAfter.amount.toString());

        const escrowAssetAtaAfter = await getAccount(anchor.getProvider().connection, escrowAssetAta);
        console.log("Escrow ATA balance (after):", escrowAssetAtaAfter.amount.toString());

        // --------- 检查 Auction 账户 ---------
        const auctionAccount = await program.account.auction.fetch(auctionPda);
        console.log("Auction account:", {
            pda: auctionPda.toBase58(),
            auctionHouse: auctionAccount.auctionHouse.toBase58(),
            seller: auctionAccount.seller.toBase58(),
            mint: auctionAccount.assetMintAccount.toBase58(),
            escrowAssetAta: auctionAccount.escrowAssetAta.toBase58(),
            escrowBidTa: auctionAccount.escrowBidTa.toBase58(),
            amount: auctionAccount.amount.toString(),
            startPrice: auctionAccount.startPrice.toString(),
            startTime: auctionAccount.startTime.toString(),
            endTime: auctionAccount.endTime.toString(),
            auctionType: auctionAccount.auctionType,

            englishReservePrice: auctionAccount.englishReservePrice.toString(),
            englishHighestBid: auctionAccount.englishHighestBid.toString(),
            englishHighestBidder: auctionAccount.englishHighestBidder.toBase58(),

            dutchPriceDropInterval: auctionAccount.dutchPriceDropInterval.toString(),
            dutchPriceDropAmount: auctionAccount.dutchPriceDropAmount.toString(),
            dutchEndPrice: auctionAccount.dutchEndPrice.toString(),

            status: auctionAccount.status,
            finalPrice: auctionAccount.finalPrice.toString(),
            buyer: auctionAccount.buyer.toBase58()
        });
    });
```

创建荷兰拍卖：

```tsx
    it("create_dutch_auction", async () => {
        // 拍卖 ID（从链上 auction_house.next_auction_id 取）
        const auctionHouseAccount = await program.account.auctionHouse.fetch(auctionHousePda);
        const nextAuctionId = auctionHouseAccount.nextAuctionId;
        const settlementMint = auctionHouseAccount.settlementMint;

        // 拍卖账户
        const [auctionPda, _auctionBump] = anchor.web3.PublicKey.findProgramAddressSync(
            [
                Buffer.from("auction"),
                auctionHousePda.toBuffer(),
                new anchor.BN(nextAuctionId).toArrayLike(Buffer, "le", 8)
            ],
            program.programId
        );

        // 拍品托管 ATA
        const escrowAssetAta = anchor.utils.token.associatedAddress({
            mint: ftPk,
            owner: auctionPda,
        });

        // 出价资金托管 Token Account
        const [escrowBidTa, _escrowBidBump] = anchor.web3.PublicKey.findProgramAddressSync(
            [
                Buffer.from("escrow"),
                auctionPda.toBuffer()
            ],
            program.programId
        );

        const now = Math.floor(Date.now() / 1000);
        const startTime = new anchor.BN(now + 60); // 1分钟后开始
        const endTime = new anchor.BN(now + 3600); // 1小时后结束

        // --------- 执行前的状态 ---------
        const sellerAtaBefore = await getAccount(anchor.getProvider().connection, sellerFTAta);
        console.log("Seller ATA balance (before):", sellerAtaBefore.amount.toString());

        let escrowAssetAtaBefore: Account;
        try {
            escrowAssetAtaBefore = await getAccount(anchor.getProvider().connection, escrowAssetAta);
            console.log("Escrow ATA balance (before):", escrowAssetAtaBefore.amount.toString());
        } catch {
            console.log("Escrow ATA does not exist yet.");
        }

        const args = {
            amount: new anchor.BN(1),
            startPrice: new anchor.BN(1000),
            startTime,
            endTime,
            auctionType: 1,

            englishReservePrice: new anchor.BN(0),

            dutchPriceDropInterval: new anchor.BN(60),  // 每 60 秒降价
            dutchPriceDropAmount: new anchor.BN(100),   // 每次降 100
            dutchEndPrice: new anchor.BN(400),          // 最低价 400
        };

        const tx = await program.methods
            .createAuction(args)
            .accounts({
                platform: platformPda,
                auctionHouseAuthority: auctionHouseAdminKeypair.publicKey,
                settlementMintAccount: settlementMint,
                auctionHouse: auctionHousePda,
                seller: sellerKeypair.publicKey,
                assetMintAccount: ftPk,
                sellerAssetAta: sellerFTAta,
                escrowAssetAta,
                escrowBidTa,
                auction: auctionPda,
                systemProgram: anchor.web3.SystemProgram.programId,
                tokenProgram: anchor.utils.token.TOKEN_PROGRAM_ID,
                associatedTokenProgram: anchor.utils.token.ASSOCIATED_PROGRAM_ID,
            }).transaction();

        tx.feePayer = sellerKeypair.publicKey;
        const txSig = await anchor.web3.sendAndConfirmTransaction(
            anchor.getProvider().connection,
            tx,
            [sellerKeypair],
        );

        // --------- 执行后的状态 ---------
        const sellerAtaAfter = await getAccount(anchor.getProvider().connection, sellerFTAta);
        console.log("Seller ATA balance (after):", sellerAtaAfter.amount.toString());

        const escrowAssetAtaAfter = await getAccount(anchor.getProvider().connection, escrowAssetAta);
        console.log("Escrow ATA balance (after):", escrowAssetAtaAfter.amount.toString());

        // --------- 检查 Auction 账户 ---------
        const auctionAccount = await program.account.auction.fetch(auctionPda);
        console.log("Auction account:", {
            pda: auctionPda.toBase58(),
            auctionHouse: auctionAccount.auctionHouse.toBase58(),
            seller: auctionAccount.seller.toBase58(),
            mint: auctionAccount.assetMintAccount.toBase58(),
            escrowAssetAta: auctionAccount.escrowAssetAta.toBase58(),
            escrowBidTa: auctionAccount.escrowBidTa.toBase58(),
            amount: auctionAccount.amount.toString(),
            startPrice: auctionAccount.startPrice.toString(),
            startTime: auctionAccount.startTime.toString(),
            endTime: auctionAccount.endTime.toString(),
            auctionType: auctionAccount.auctionType,

            englishReservePrice: auctionAccount.englishReservePrice.toString(),
            englishHighestBid: auctionAccount.englishHighestBid.toString(),
            englishHighestBidder: auctionAccount.englishHighestBidder.toBase58(),

            dutchPriceDropInterval: auctionAccount.dutchPriceDropInterval.toString(),
            dutchPriceDropAmount: auctionAccount.dutchPriceDropAmount.toString(),
            dutchEndPrice: auctionAccount.dutchEndPrice.toString(),

            status: auctionAccount.status,
            finalPrice: auctionAccount.finalPrice.toString(),
            buyer: auctionAccount.buyer.toBase58()
        });
    });
```

拍卖账户的 `seeds` 中使用拍卖行账户中的 `next_auction_id`，需要先从链上获取到这个值。

`.toArrayLike(Buffer, "le", 8)`

把 BN 转成字节序列，为了和链上 `seeds` 计算一直，使用小端序。

- 第一个参数 `Buffer`
    
    指定输出的类型是 Node.js 的 `Buffer`。
    
- 第二个参数 `"le"`
    
    表示字节序（endianness）：
    
    - `"le"` = little endian（小端序，低位在前）
    - `"be"` = big endian（高位在前）
- 第三个参数 `8`
    
    指定输出字节数 = 8 个字节（64 位整数固定大小）。

---

[返回课程首页](../../../../README.md)
