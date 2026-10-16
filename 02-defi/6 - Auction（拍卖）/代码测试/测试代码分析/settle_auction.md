# settle_auction

英式拍卖或荷兰拍卖的结算。

```tsx
    it("settle_auction", async () => {
        // 拍卖账户
        const auctionPda = new anchor.web3.PublicKey("BHfKweN5MxjitwnmCs6QJ3iVwPsZmFfsw6FXvn97ySJM");

        // 获取最新 auctionId
        const auctionHouseAccount = await program.account.auctionHouse.fetch(auctionHousePda);
        const settlementMint = auctionHouseAccount.settlementMint;

        const auctionAccount = await program.account.auction.fetch(auctionPda);

        // escrow 账户
        const escrowAssetAta = anchor.utils.token.associatedAddress({
            mint: auctionAccount.assetMintAccount,
            owner: auctionPda,
        });
        const [escrowBidTa] = anchor.web3.PublicKey.findProgramAddressSync(
            [Buffer.from("escrow"), auctionPda.toBuffer()],
            program.programId
        );

        // 卖家 ATA
        const sellerSettlementAta = anchor.utils.token.associatedAddress({
            mint: settlementMint,
            owner: auctionAccount.seller,
        });
        const sellerAssetAta = anchor.utils.token.associatedAddress({
            mint: auctionAccount.assetMintAccount,
            owner: auctionAccount.seller,
        });

        // 买家 ATA
        let buyerPubkey: anchor.web3.PublicKey | null = null;

        if (auctionAccount.auctionType === 0) {
            // 英式拍卖：结算前买家 = 当前最高出价人
            if (!auctionAccount.englishHighestBidder.equals(anchor.web3.PublicKey.default)) {
                buyerPubkey = auctionAccount.englishHighestBidder;
            }
        } else if (auctionAccount.auctionType === 1) {
            // 荷兰拍卖：买断成功时 auction.buyer 已经更新
            if (!auctionAccount.buyer.equals(anchor.web3.PublicKey.default)) {
                buyerPubkey = auctionAccount.buyer;
            }
        }

        let buyerSettlementAta: PublicKey | null = null;
        let buyerAssetAta: PublicKey | null = null;

        if (buyerPubkey) {
            buyerSettlementAta = anchor.utils.token.associatedAddress({
                mint: settlementMint,
                owner: buyerPubkey,
            });
            buyerAssetAta = anchor.utils.token.associatedAddress({
                mint: auctionAccount.assetMintAccount,
                owner: buyerPubkey,
            });
        }

        // ----------- 结算前快照 -----------
        const [
            sellerSettlementBefore,
            sellerAssetBefore,
            platformTreasuryBefore,
            auctionHouseTreasuryBefore,
            escrowAssetBefore,
            escrowBidBefore
        ] = await Promise.all([
            getAccount(anchor.getProvider().connection, sellerSettlementAta),
            getAccount(anchor.getProvider().connection, sellerAssetAta),
            getAccount(anchor.getProvider().connection, platformTreasuryAta),
            getAccount(anchor.getProvider().connection, auctionHouseTreasuryAta),
            getAccount(anchor.getProvider().connection, escrowAssetAta),
            getAccount(anchor.getProvider().connection, escrowBidTa),
        ]);

        let buyerSettlementBefore: any = null;
        let buyerAssetBefore: any = null;
        if (buyerSettlementAta && buyerAssetAta) {
            buyerSettlementBefore = await getAccount(anchor.getProvider().connection, buyerSettlementAta);
            buyerAssetBefore = await getAccount(anchor.getProvider().connection, buyerAssetAta);
        }

        console.log("结算前余额:");
        console.log("- 买家资金 ATA:", buyerSettlementBefore?.amount.toString() ?? "N/A");
        console.log("- 买家拍品 ATA:", buyerAssetBefore?.amount.toString() ?? "N/A");
        console.log("- 卖家资金 ATA:", sellerSettlementBefore.amount.toString());
        console.log("- 卖家拍品 ATA:", sellerAssetBefore.amount.toString());
        console.log("- 平台 Treasury:", platformTreasuryBefore.amount.toString());
        console.log("- 拍卖行 Treasury:", auctionHouseTreasuryBefore.amount.toString());
        console.log("- Escrow Asset:", escrowAssetBefore.amount.toString());
        console.log("- Escrow Bid:", escrowBidBefore.amount.toString());

        // ----------- 调用 settleAuction -----------
        await program.methods
            .settleAuction()
            .accounts({
                platform: platformPda,
                auctionHouse: auctionHousePda,
                auction: auctionPda,
                escrowAssetAta,
                escrowBidTa,
                auctionHouseTreasuryAta,
                platformTreasuryAta,
                seller: auctionAccount.seller,
                sellerSettlementAta,
                sellerAssetAta,
                buyerSettlementAta: buyerSettlementAta ?? null,
                buyerAssetAta: buyerAssetAta ?? null,
                tokenProgram: anchor.utils.token.TOKEN_PROGRAM_ID,
            }).rpc();

        // ----------- 结算后快照 -----------
        const [
            sellerSettlementAfter,
            sellerAssetAfter,
            platformTreasuryAfter,
            auctionHouseTreasuryAfter,
            escrowAssetAfter,
            escrowBidAfter
        ] = await Promise.all([
            getAccount(anchor.getProvider().connection, sellerSettlementAta),
            getAccount(anchor.getProvider().connection, sellerAssetAta),
            getAccount(anchor.getProvider().connection, platformTreasuryAta),
            getAccount(anchor.getProvider().connection, auctionHouseTreasuryAta),
            getAccount(anchor.getProvider().connection, escrowAssetAta),
            getAccount(anchor.getProvider().connection, escrowBidTa),
        ]);

        let buyerSettlementAfter: any = null;
        let buyerAssetAfter: any = null;
        if (buyerSettlementAta && buyerAssetAta) {
            buyerSettlementAfter = await getAccount(anchor.getProvider().connection, buyerSettlementAta);
            buyerAssetAfter = await getAccount(anchor.getProvider().connection, buyerAssetAta);
        }

        console.log("结算后余额:");
        console.log("- 买家资金 ATA:", buyerSettlementAfter?.amount.toString() ?? "N/A");
        console.log("- 买家拍品 ATA:", buyerAssetAfter?.amount.toString() ?? "N/A");
        console.log("- 卖家资金 ATA:", sellerSettlementAfter.amount.toString());
        console.log("- 卖家拍品 ATA:", sellerAssetAfter.amount.toString());
        console.log("- 平台 Treasury:", platformTreasuryAfter.amount.toString());
        console.log("- 拍卖行 Treasury:", auctionHouseTreasuryAfter.amount.toString());
        console.log("- Escrow Asset:", escrowAssetAfter.amount.toString());
        console.log("- Escrow Bid:", escrowBidAfter.amount.toString());

        // ----------- 检查状态 -----------
        const auctionAfter = await program.account.auction.fetch(auctionPda);
        console.log("Auction after settle:", {
            status: auctionAfter.status,
            finalPrice: auctionAfter.finalPrice.toString(),
            buyer: auctionAfter.buyer.toBase58(),
        });
    });
```

---

[返回课程首页](../../../../README.md)
