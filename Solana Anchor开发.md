#### Solana Anchor开发



export https_proxy=http://127.0.0.1:7890



npx create-solana-dapp 创建项目框架（前端带合约端，nextjs、tailwind、anchor）

solana-test-validator 运行本地验证节点

npm install anchor-bankrun 安装bankrun测试套件

anchor keys sync 同步Program ID

### 运行测试命令

```
# 推荐：使用 vitest (bankrun 内存测试，最快)
cd voting-app/anchor
npx vitest run

# 或者使用 surfpool (需要先启动)
surfpool start  # 终端 1
npx vitest run  # 终端 2
```

### Foundry 开发者快速参考

| 你熟悉的 Foundry       | Solana/Surfpool 对应            |
| :--------------------- | :------------------------------ |
| `forge test`           | `npx vitest run`                |
| `anvil`                | `surfpool start`                |
| `vm.startPrank(alice)` | `context.payer` / signers       |
| `vm.expectRevert()`    | `expect(...).rejects.toThrow()` |
| `assertEq(a, b)`       | `expect(a).toBe(b)`             |
| `CREATE2` 地址         | PDA (`findProgramAddressSync`)  |
