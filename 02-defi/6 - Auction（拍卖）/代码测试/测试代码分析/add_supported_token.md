# add_supported_token

使用 Solana CLI 配置文件中的钱包地址 `3ZoojjwWCgArGREegaQgifwwtZF1EQdid5BjEX5weXah` 作为 **平台管理员** 为平台添加支持的结算代币。

在主网上可以添加 USDT, USDC 等代币作为支持代币，但在本地测试时需要模拟一个 SPL Token Account。

```tsx
    it("add_supported_token", async () => {
        const event: any = await new Promise(async (resolve, reject) => {
            let listener: number | null = null;
            try {
                listener = program.addEventListener("tokenAdded", async (event, slot) => {
                    try {
                        console.log("捕获事件: tokenAdded", {
                            platform: event.platform.toBase58(),
                            mint: event.mint.toBase58(),
                            addedAt: event.addedAt.toString()
                        }, "at slot", slot);

                        // 成功时
                        resolve(event);
                    } catch (err) {
                        // 失败时
                        reject(err);
                    } finally {
                        await program.removeEventListener(listener);
                    }
                });

                const txSig = await program.methods.addSupportedToken().accounts({
                    platform: platformPda,
                    supportedToken: supportedTokenPda,
                    mintAccount: mintPk,
                    platformTreasuryAta,
                    authority: anchor.getProvider().wallet.publicKey,
                    systemProgram: anchor.web3.SystemProgram.programId,
                    tokenProgram: anchor.utils.token.TOKEN_PROGRAM_ID,
                    associatedTokenProgram: anchor.utils.token.ASSOCIATED_PROGRAM_ID
                }).rpc();

                // 如果这里 resolve()，Promise 就结束了，但实际上我们需要等待监听事件
                // resolve(txSig);
            } catch (err) {
                if (listener != null) await program.removeEventListener(listener);
                reject(err);
            }
        });

        // 使用 chai 断言检查 event 的内容
        expect(event.platform.toBase58()).to.equal(platformPda.toBase58());
        expect(event.mint.toBase58()).to.equal(mintPk.toBase58());
        // 时间戳一般断言为存在即可（因为是动态值）
        expect(event.addedAt.toNumber()).to.be.a("number");

        // 检查 ATA
        const ataInfo = await getAccount(
            anchor.getProvider().connection,
            platformTreasuryAta
        );

        console.log("平台 ATA:", {
            address: platformTreasuryAta.toBase58(),
            mint: ataInfo.mint.toBase58(),
            owner: ataInfo.owner.toBase58(),
            amount: ataInfo.amount.toString(),
        });
    });
```

---

[返回课程首页](../../../../README.md)
