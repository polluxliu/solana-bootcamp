# 代码 - Cargo.toml

```toml
[package]
name = "voting"
version = "0.1.0"
description = "Created with Anchor"
edition = "2021"

[lib]
crate-type = ["cdylib", "lib"]
name = "voting"

[features]
default = []
cpi = ["no-entrypoint"]
no-entrypoint = []
no-idl = []
no-log-ix-name = []
idl-build = ["anchor-lang/idl-build"]
test-sbf = []

[dependencies]
anchor-lang = "0.31.1"

[dev-dependencies]
mollusk-svm = "0.4.2"
solana-program = "3.0.0"
solana-sdk = "2.3.1"
bincode = "1.3.3"
```

---

[返回课程首页](../../README.md)
