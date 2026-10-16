# initialize_platform

使用 Solana CLI 配置文件中的钱包地址 `3ZoojjwWCgArGREegaQgifwwtZF1EQdid5BjEX5weXah` 作为 **平台管理员** 创建平台。

# 测试代码分析

```tsx
		it("initialize_platform", async () => {
        // 在指令调用之前先设置事件监听
        const listener = program.addEventListener("platformInitialized", (event, slot) => {
            console.log("捕获事件: platformInitialized", {
                authority: event.authority.toBase58(),
                treasury: event.treasury.toBase58(),
                platformFeeRate: event.platformFeeRate,
                auctionDurationMin: event.auctionDurationMin.toString(),
                auctionDurationMax: event.auctionDurationMax.toString(),
                createdAt: event.createdAt.toString(),
            }, "at slot", slot);

            // 使用 chai 断言检查 event 的内容
            expect(event.authority.toBase58()).to.equal(anchor.getProvider().wallet.publicKey.toBase58());
            expect(event.treasury.toBase58()).to.equal(treasury.toBase58());
            expect(event.platformFeeRate).to.equal(100);
            expect(event.auctionDurationMin.toNumber()).to.equal(300);
            expect(event.auctionDurationMax.toNumber()).to.equal(86_400);

            // 时间戳一般断言为存在即可（因为是动态值）
            expect(event.createdAt.toNumber()).to.be.a("number");
        });

        // 指令调用
        const txSig = await program.methods.initializePlatform({
            auctionHouseCreationFee: new anchor.BN(1_000_000), // 0.001 SOL，以 lamports 为单位
            platformFeeRate: 100, // 1%，以基点为单位
            auctionDurationMin: new anchor.BN(300), // 5 分钟，以秒为单位
            auctionDurationMax: new anchor.BN(86_400), // 1 天，以秒为单位
        }).accounts({
            platform: platformPda,
            authority: anchor.getProvider().wallet.publicKey,
            treasury: treasury,
            systemProgram: anchor.web3.SystemProgram.programId,
        }).rpc();
        
        // 移除监听

        // rpc() 返回之后，有可能事件回调还没触发，如果直接移除监听，会导致客户端没有接收到事件数据
        // 所以在移除监听之前添加一定延迟
        await new Promise(resolve => setTimeout(resolve, 1000));

        await program.removeEventListener(listener);
    });
```

- `program.addEventListener`
    
    这行代码执行时，客户端会通过 WebSocket 连接到 RPC 节点，订阅程序的日志更新，实时接收区块链上的事件。RPC 节点会 **主动推送** 匹配的事件到客户端。
    
    返回一个**监听器 ID**（listener ID），这是一个数字标识符，用于后续移除监听器。
    
    `authority: event.authority.toBase58()`
    
    - `event.authority` 是 `PublicKey` 对象。
        
        在 Rust 中，Pubkey 是元组结构体：
        
        `pub struct Pubkey([u8; 32])`
        
        在 TypeScript 中，Pubkey 是一个类，内部存的是 32 字节的 Uint8Array。
        
    - `toBase58()` 的作用就是把 `PublicKey` 对象转成字符串。
    
- `program.methods.initializePlatform`
    - **参数**：
        - `auctionHouseCreationFee`: 每次用户要创建一个新的 AuctionHouse，需要付的手续费，这里是 `0.001 SOL`。
        - `platformFeeRate`: 平台手续费率，100 基点 = 1%。
        - `auctionDurationMin`: 最小拍卖时长，300 秒 = 5 分钟。
        - `auctionDurationMax`: 最大拍卖时长，86400 秒 = 1 天。
    - **账户**：
        - `platform`: 平台 PDA（存放全局配置）。
        - `authority`: 平台管理员的钱包（provider.wallet）。
        - `treasury`: 平台收费账户。
        - `systemProgram`: 系统程序（用于创建账户）。
        
        `new anchor.BN()`
        
        表示创建一个 BN 类型的整数。`BN` 来自 **bn.js** 库（Anchor TS SDK 内部 re-export 了），全称是 **Big Number**，就是大整数类型。
        
        Anchor TS SDK 规定凡是 IDL 里标注 `u64` 或 `i64` 的参数，必须用 `BN` 对象传。Rust 里 `u64` 的取值范围是 $0 到 2^{64}-1$，`i64` 的取值范围是 $-2^{63} 到 2^{63} - 1$。而 JavaScript 的 `number` 是 **64 位浮点数**（双精度），其布局为：
        
        ```json
        1 位符号位 (sign)
        11 位指数 (exponent)
        52 位尾数 (fraction / mantissa)
        ```
        
        其尾数部分有一个隐藏位 (implicit leading 1)，因此实际能表示 **53 位有效二进制精度**，其范围为 $-(2^{53} - 1)$ 到 $(2^{53} - 1)$。
        

- `await program.removeEventListener(listener)`
    
    移除事件监听。如果在 `.rpc()` 之后立即移除事件监听，有可能客户端还没有接收到事件数据，所以可以添加一定的延迟，然后再移除监听。
    

- `anchor.getProvider().wallet`
    
    当前 Provider 使用的 Wallet 对象。
    
- `wallet.publicKey`
    
    PublicKey 类型，表示这个钱包的公钥（地址）。只有地址，没有私钥，不能签名。
    
- `wallet.payer`
    
    Keypair 类型，密钥对类型，包含私钥和公钥。在 Solana 里，签名者必须是 `Keypair` 类型。

    💡
    
    `Keypair` = 公钥 + 私钥
    
    当你运行 `anchor test` 的时候，Anchor 会去读取环境变量 **ANCHOR_WALLET** 或者如果没有设置环境变量会去读取 Solana CLI 配置文件中的 **Keypair Path**，它们都指向一个 JSON 文件，这个文件就是本地钱包的完整 Keypair，Anchor 把这个文件恢复成一个本地钱包。这个 Keypair 就赋值给了 `provider.wallet.payer`，钱包的地址就赋值给了 `provider.wallet.publicKey`。

上面的测试代码是可以使用的，但严格来讲存在问题，比如 `.rpc()` 抛出异常时代码就无法移除事件监听。另外更大的一个问题是，如果 `expect(...)` 抛了 `AssertionError`，但测试用例依然显示 `passing`，这因为现在代码中的 **这些断言代码都在事件监听的回调函数中**，而不是直接在 `it(...)` 的测试上下文里执行。

```tsx
it("test", async () => {
    // ← 测试上下文开始
    
    ...
    
    // ← 测试上下文结束
});
```

只有直接在测试上下文中抛出的异常才能被 Mocha 感知：

```tsx
it("直接抛出", async () => {
    // 直接在测试上下文中抛出
    if (someCondition) {
        throw new Error("测试失败");
    }
    
    // 同步断言
    expect(result).to.equal(expected);
});
```

对于在测试上下文中的异步回调，无法被 Mocha 感知，比如上面测试代码中 `addEventListener` 的回调函数中的 `expect` 抛出的 `AssertionError`。这种情况下，可以使用两种方式来解决。

一种是使用 `await new Promise()`：

```tsx
it("await Promise", async () => {
    // 将异步回调包装成 Promise
    await new Promise((resolve, reject) => {
        setTimeout(() => {
            try {
                expect(1).to.equal(2);
                resolve();
            } catch (error) {
                reject(error); // 通过 Promise rejection 传递给 Mocha
            }
        }, 100);
    });
});
```

一种是使用 `done()`，这是 Mocha 提供的回调函数，专门用于异步测试，用来告诉测试框架异步测试什么时候完成。

```tsx
it("done", (done) => {
    setTimeout(() => {
        try {
            expect(1).to.equal(2);
            done();
        } catch (error) {
            done(error);
        }
    }, 100);
});
```

- **成功**：`done()` - 不传参数
- **失败**：`done(error)` - 传递错误对象或错误消息

我们将上面的测试代码进行了如下调整：

```tsx
    it("initialize_platform", async () => {
        const event: any = await new Promise(async (resolve, reject) => {
            let listener: number | null = null;
            try {
                listener = program.addEventListener("platformInitialized", async (event, slot) => {
                    try {
                        console.log("捕获事件: platformInitialized", {
                            authority: event.authority.toBase58(),
                            platformFeeRate: event.platformFeeRate,
                            auctionDurationMin: event.auctionDurationMin.toString(),
                            auctionDurationMax: event.auctionDurationMax.toString(),
                            createdAt: event.createdAt.toString(),
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

                const txSig = await program.methods.initializePlatform({
                    auctionHouseCreationFee: new anchor.BN(1_000_000), // 0.001 SOL，以 lamports 为单位
                    platformFeeRate: 100, // 1%，以基点为单位
                    auctionDurationMin: new anchor.BN(300), // 5 分钟，以秒为单位
                    auctionDurationMax: new anchor.BN(86_400), // 1 天，以秒为单位
                }).accounts({
                    platform: platformPda,
                    authority: anchor.getProvider().wallet.publicKey,
                    systemProgram: anchor.web3.SystemProgram.programId,
                }).rpc();

                // 如果这里 resolve()，Promise 就结束了，但实际上我们需要等待监听事件
                // resolve(txSig);
            } catch (err) {
                if (listener != null) await program.removeEventListener(listener);
                reject(err);
            }
        });

        // 使用 chai 断言检查 event 的内容
        expect(event.authority.toBase58()).to.equal(anchor.getProvider().wallet.publicKey.toBase58());
        expect(event.platformFeeRate).to.equal(100);
        expect(event.auctionDurationMin.toNumber()).to.equal(300);
        expect(event.auctionDurationMax.toNumber()).to.equal(86_400);

        // 时间戳一般断言为存在即可（因为是动态值）
        expect(event.createdAt.toNumber()).to.be.a("number");
    });
```

这段代码中做了以下调整：

- 在整个测试代码逻辑上添加了 `try…catch`，这样发生异常也能移除监听。
- 将整个测试代码使用 `Promise` 进行了包装，放到了 `Promise` 的执行器函数中。`Promise` 实例创建的时候，执行器函数会立即执行。
    
    执行器函数的 `resolve` 参数将 `Promise` 标记为成功，`reject` 参数将 `Promise` 标记为失败。
    
    ```tsx
    const returnValue = await new Promise((resolve, reject) => {
      // resolve(value)
      // reject(error)
    })
    ```
    
    - 当 Promise 被 resolve 时，`returnValue` 的值就是传给 `resolve(value)` 的参数 `value`。
    - 当 Promise 被 reject 时，await 会抛出异常，而不是给 `returnValue` 赋值。

# 测试代码运行

- 使用命令 `anchor test`
    
    在当前进程中启动一个临时的、非持久的运行环境。
    
- 使用命令 `solana-test-validator`
    
    在一个独立进程中开启一个持久化的环境，然后通过 `anchor test --skip-local-validator` 命令运行测试代码。
    
    注意，由于 `initialize_platform` 指令的账户上下文中的 `platform` 账户的约束为：
    
    ```rust
        #[account(
            init,
            payer = authority,
            space = 8 + Platform::INIT_SPACE,
            seeds = [b"platform"],
            bump
        )]
    ```
    
    其 `seeds` 为固定的 “platform”，意味着只能创建 1 次。多次运行 `anchor test --skip-local-validator` 会报错：
    
    ```
    [                                                                                                                                                                                    
      "Program 8zpNETKx1mHmWcd1wBj5cE5ztn642GbxMwtHTPAk6TSL invoke [1]",                                                                                                                 
      "Program log: Instruction: InitializePlatform",                                                                                                                                    
      "Program 11111111111111111111111111111111 invoke [2]",                                                                                                                             
      "Allocate: account Address { address: HVtSY4wdg5vtQjTJdn6YWCtMEAa1rA2FtfpM2eoSwUSy, base: None } already in use",                                                                  
      "Program 11111111111111111111111111111111 failed: custom program error: 0x0",                                                                                                      
      "Program 8zpNETKx1mHmWcd1wBj5cE5ztn642GbxMwtHTPAk6TSL consumed 5316 of 200000 compute units",                                                                                      
      "Program 8zpNETKx1mHmWcd1wBj5cE5ztn642GbxMwtHTPAk6TSL failed: custom program error: 0x0"                                                                                           
    ]
    ```
    
    可以使用命令 `solana-test-validator --reset` 重置这个持久化环境，但需要每次运行测试的时候都要使用这个命令重置环境，这其实就相当于使用 `anchor test` 这个命令了。

---

[返回课程首页](../../../../README.md)
