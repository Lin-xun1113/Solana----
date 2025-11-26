# anchor-litesvm 测试完全指南

## 概述

`anchor-litesvm` 是 LiteSVM 的 Anchor 封装，让你可以直接使用 Anchor 的 `program.methods` 语法进行快速测试。

| 信息         | 内容                                      |
| :----------- | :---------------------------------------- |
| **GitHub**   | https://github.com/LiteSVM/anchor-litesvm |
| **最新版本** | 0.2.1                                     |
| **底层**     | LiteSVM (比 solana-test-validator 快得多) |

## 安装

bash

```bash
# 推荐：直接从 npm 安装
yarn add anchor-litesvm

# 或使用 npm
npm install anchor-litesvm
```

## 文件结构



```
anchor/
├── programs/votingapp/src/lib.rs   # Rust 程序
├── tests/votingapp.test.ts          # TypeScript 测试 (用 anchor-litesvm)
├── target/
│   ├── idl/votingapp.json           # 自动生成的 IDL
│   └── types/votingapp.ts           # 自动生成的类型
└── Anchor.toml
```

## 完整测试模板

typescript

```typescript
// ============================================
// 第 1 部分：导入
// ============================================
import { fromWorkspace, LiteSVMProvider } from "anchor-litesvm";
import { Program } from "@coral-xyz/anchor";
import { Keypair, PublicKey } from "@solana/web3.js";
import * as anchor from "@coral-xyz/anchor";
import { Votingapp } from "../target/types/votingapp";

const IDL = require("../target/idl/votingapp.json");

// ============================================
// 第 2 部分：测试套件
// ============================================
describe("Votingapp", () => {
  let provider: LiteSVMProvider;
  let program: Program<Votingapp>;
  let payer: Keypair;

  // ============================================
  // 第 3 部分：初始化
  // ============================================
  beforeAll(async () => {
    // fromWorkspace 会自动加载 anchor 目录中编译好的程序
    const client = fromWorkspace("anchor");
    
    // 创建 provider
    provider = new LiteSVMProvider(client);
    anchor.setProvider(provider);
    
    // 创建 program 实例
    program = new Program<Votingapp>(IDL, provider);
    
    // 获取 payer (自带 SOL)
    payer = provider.wallet.payer;
  });

  // ============================================
  // 第 4 部分：测试用例
  // ============================================
  it("should create a new poll", async () => {
    const pollId = new anchor.BN(1);
    const description = "Best Solana Project 2024";
    const pollStart = new anchor.BN(Math.floor(Date.now() / 1000));
    const pollEnd = new anchor.BN(Math.floor(Date.now() / 1000) + 86400);

    // 计算 PDA
    const [pollPda] = PublicKey.findProgramAddressSync(
      [pollId.toArrayLike(Buffer, "le", 8)],
      program.programId
    );

    // 调用程序
    await program.methods
      .initializePoll(pollId, description, pollStart, pollEnd)
      .accounts({
        signer: payer.publicKey,
      })
      .signers([payer])
      .rpc();

    // 验证账户数据
    const poll = await program.account.poll.fetch(pollPda);
    expect(poll.pollId.toNumber()).toBe(1);
    expect(poll.description).toBe(description);
  });
});
```

## 核心 API

### 1. 初始化

typescript

```typescript
import { fromWorkspace, LiteSVMProvider } from "anchor-litesvm";

// 从 anchor workspace 加载程序
// 参数是 anchor 目录的相对路径
const client = fromWorkspace("anchor");

// 创建 provider (类似 AnchorProvider)
const provider = new LiteSVMProvider(client);

// 设置为全局 provider
anchor.setProvider(provider);

// 创建 program 实例
const program = new Program(IDL, provider);

// 获取 payer
const payer = provider.wallet.payer;
```

### 2. 调用程序指令

typescript

```typescript
// 基本模式
await program.methods
  .instructionName(arg1, arg2, ...)   // 指令名 (camelCase)
  .accounts({                          // 账户
    signer: payer.publicKey,
    // PDA 账户可以自动推导，也可以手动指定
  })
  .signers([payer])                    // 签名者
  .rpc();                              // 发送交易
```

### 3. 计算 PDA

typescript

```typescript
// 单个 seed (u64)
const [pollPda] = PublicKey.findProgramAddressSync(
  [pollId.toArrayLike(Buffer, "le", 8)],
  program.programId
);

// 多个 seeds (u64 + string)
const [candidatePda] = PublicKey.findProgramAddressSync(
  [
    pollId.toArrayLike(Buffer, "le", 8),
    Buffer.from(candidateName),
  ],
  program.programId
);
```

### 4. 读取账户

typescript

```typescript
// 获取单个账户
const poll = await program.account.poll.fetch(pollPda);
console.log(poll.pollId.toNumber());
console.log(poll.description);

// 获取所有账户
const allPolls = await program.account.poll.all();
```

### 5. 空投 SOL

typescript

```typescript
// LiteSVM 方式
provider.client.airdrop(userPublicKey, BigInt(1_000_000_000)); // 1 SOL
```

### 6. 时间穿越

typescript

```typescript
// 向前跳转 slot
provider.client.warpToSlot(BigInt(100));

// 设置时钟
provider.client.setClock({
  slot: BigInt(100),
  epochStartTimestamp: BigInt(Math.floor(Date.now() / 1000)),
  epoch: BigInt(1),
  leaderScheduleEpoch: BigInt(1),
  unixTimestamp: BigInt(Math.floor(Date.now() / 1000) + 86400), // +1天
});
```

## 断言 (vitest)

typescript

```typescript
// 基本断言
expect(value).toBe(expected);           // 严格相等
expect(value).toEqual(expected);        // 深度相等

// BN 数字比较
expect(poll.pollId.toNumber()).toBe(1);

// 期望抛出错误
await expect(
  program.methods.xxx().accounts({...}).signers([]).rpc()
).rejects.toThrow();

// 期望特定错误消息
await expect(...).rejects.toThrow("AccountNotInitialized");
```

## 数据类型对照表

| Rust 类型     | TypeScript 类型 | 转换方法               |
| :------------ | :-------------- | :--------------------- |
| `u64` / `i64` | `anchor.BN`     | `new anchor.BN(123)`   |
| `u8` / `i8`   | `number`        | 直接用                 |
| `String`      | `string`        | 直接用                 |
| `Pubkey`      | `PublicKey`     | `new PublicKey("...")` |
| `bool`        | `boolean`       | 直接用                 |
| `Vec<u8>`     | `Buffer`        | `Buffer.from([...])`   |
| `Option<T>`   | `T | null`      | 直接用                 |

## 常用测试模式

### 模式 1: 创建账户

typescript

```typescript
it("should create account", async () => {
  const [pda] = PublicKey.findProgramAddressSync([...seeds], program.programId);
  
  await program.methods
    .initialize(...)
    .accounts({ signer: payer.publicKey })
    .signers([payer])
    .rpc();
  
  const account = await program.account.myAccount.fetch(pda);
  expect(account.field).toBe(expectedValue);
});
```

### 模式 2: 更新账户

typescript

```typescript
it("should update account", async () => {
  await program.methods
    .update(newValue)
    .accounts({ 
      myAccount: pda, 
      authority: payer.publicKey 
    })
    .signers([payer])
    .rpc();
  
  const account = await program.account.myAccount.fetch(pda);
  expect(account.value.toNumber()).toBe(newValue);
});
```

### 模式 3: 期望失败

typescript

```typescript
it("should fail without authority", async () => {
  const hacker = Keypair.generate();
  
  await expect(
    program.methods
      .update(999)
      .accounts({ 
        myAccount: pda, 
        authority: hacker.publicKey 
      })
      .signers([hacker])
      .rpc()
  ).rejects.toThrow();
});
```

### 模式 4: 多用户测试

typescript

```typescript
it("should work with multiple users", async () => {
  const user1 = Keypair.generate();
  const user2 = Keypair.generate();
  
  // 给用户空投 SOL
  provider.client.airdrop(user1.publicKey, BigInt(1_000_000_000));
  provider.client.airdrop(user2.publicKey, BigInt(1_000_000_000));
  
  // 分别测试...
});
```

### 模式 5: 时间相关测试

typescript

```typescript
it("should fail before start time", async () => {
  // 设置当前时间在 poll 开始之前
  provider.client.setClock({
    unixTimestamp: BigInt(pollStart - 100),
    // ... 其他字段
  });
  
  await expect(
    program.methods.vote(...).accounts({...}).signers([]).rpc()
  ).rejects.toThrow("VotingNotStarted");
});
```

## 运行测试

bash

```bash
# 运行所有测试
npx vitest run

# 监听模式
npx vitest

# 运行特定测试
npx vitest run -t "should create"

# 详细输出
npx vitest run --reporter=verbose
```

## 调试技巧

typescript

```typescript
// 打印交易签名
const tx = await program.methods.xxx().accounts({...}).rpc();
console.log("TX:", tx);

// 打印账户数据
const account = await program.account.poll.fetch(pda);
console.log("Account:", JSON.stringify(account, null, 2));

// 打印 PDA
console.log("PDA:", pda.toBase58());
```

## 注意事项

1. **异步说明**: LiteSVM 本身是同步的，但 `anchor-litesvm` 实现了异步接口以兼容 Anchor 的 API

   

2. **程序加载**: `fromWorkspace("anchor")` 会自动从 `anchor/target/deploy/` 加载编译好的 `.so` 文件

   

3. **速度**: 比 `solana-test-validator` 快很多，因为不需要启动完整的验证器

   

4. **SPL Token**: LiteSVM 默认包含 System Program 和 SPL Token 程序

   

## 完整示例：Voting App

typescript

```typescript
import { fromWorkspace, LiteSVMProvider } from "anchor-litesvm";
import { Program } from "@coral-xyz/anchor";
import { Keypair, PublicKey } from "@solana/web3.js";
import * as anchor from "@coral-xyz/anchor";
import { Votingapp } from "../target/types/votingapp";

const IDL = require("../target/idl/votingapp.json");

describe("Votingapp", () => {
  let provider: LiteSVMProvider;
  let program: Program<Votingapp>;
  let payer: Keypair;

  beforeAll(async () => {
    const client = fromWorkspace("anchor");
    provider = new LiteSVMProvider(client);
    anchor.setProvider(provider);
    program = new Program<Votingapp>(IDL, provider);
    payer = provider.wallet.payer;
  });

  describe("Initialize Poll", () => {
    it("should create a new poll successfully", async () => {
      const pollId = new anchor.BN(1);
      const description = "Best Solana Project 2024";
      const pollStart = new anchor.BN(Math.floor(Date.now() / 1000));
      const pollEnd = new anchor.BN(Math.floor(Date.now() / 1000) + 86400);

      const [pollPda] = PublicKey.findProgramAddressSync(
        [pollId.toArrayLike(Buffer, "le", 8)],
        program.programId
      );

      await program.methods
        .initializePoll(pollId, description, pollStart, pollEnd)
        .accounts({ signer: payer.publicKey })
        .signers([payer])
        .rpc();

      const poll = await program.account.poll.fetch(pollPda);
      expect(poll.pollId.toNumber()).toBe(1);
      expect(poll.description).toBe(description);
      expect(poll.candidateAmount.toNumber()).toBe(0);
    });
  });

  describe("Initialize Candidate", () => {
    const testPollId = new anchor.BN(100);

    beforeAll(async () => {
      const [pollPda] = PublicKey.findProgramAddressSync(
        [testPollId.toArrayLike(Buffer, "le", 8)],
        program.programId
      );

      await program.methods
        .initializePoll(
          testPollId,
          "Test Poll",
          new anchor.BN(Math.floor(Date.now() / 1000)),
          new anchor.BN(Math.floor(Date.now() / 1000) + 86400)
        )
        .accounts({ signer: payer.publicKey })
        .signers([payer])
        .rpc();
    });

    it("should create a new candidate", async () => {
      const candidateName = "Alice";

      const [candidatePda] = PublicKey.findProgramAddressSync(
        [
          testPollId.toArrayLike(Buffer, "le", 8),
          Buffer.from(candidateName),
        ],
        program.programId
      );

      await program.methods
        .initializeCandidate(candidateName, testPollId)
        .accounts({ signer: payer.publicKey })
        .signers([payer])
        .rpc();

      const candidate = await program.account.candidate.fetch(candidatePda);
      expect(candidate.candidateName).toBe(candidateName);
      expect(candidate.candidateVotes.toNumber()).toBe(0);
    });
  });

  describe("Vote", () => {
    const votePollId = new anchor.BN(200);
    const candidateName = "Bob";

    beforeAll(async () => {
      // 创建 Poll
      await program.methods
        .initializePoll(
          votePollId,
          "Vote Test",
          new anchor.BN(Math.floor(Date.now() / 1000)),
          new anchor.BN(Math.floor(Date.now() / 1000) + 86400)
        )
        .accounts({ signer: payer.publicKey })
        .signers([payer])
        .rpc();

      // 创建 Candidate
      await program.methods
        .initializeCandidate(candidateName, votePollId)
        .accounts({ signer: payer.publicKey })
        .signers([payer])
        .rpc();
    });

    it("should vote successfully", async () => {
      const [candidatePda] = PublicKey.findProgramAddressSync(
        [
          votePollId.toArrayLike(Buffer, "le", 8),
          Buffer.from(candidateName),
        ],
        program.programId
      );

      await program.methods
        .vote(candidateName, votePollId)
        .accounts({ signer: payer.publicKey })
        .signers([payer])
        .rpc();

      const candidate = await program.account.candidate.fetch(candidatePda);
      expect(candidate.candidateVotes.toNumber()).toBe(1);
    });
  });
});
```

**总结**: `anchor-litesvm` 提供了与 Anchor 完全兼容的测试体验，API 简洁，速度快，是目前最推荐的 TypeScript 测试方案。