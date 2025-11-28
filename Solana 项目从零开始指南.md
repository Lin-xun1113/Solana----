# Solana 全栈从零开始完整指南 (2025 版)

> 从合约开发到前端集成的完整教程

---

# 第一部分：合约开发 (Anchor)

---

## 第一步：安装前置依赖 🖥️ **每台电脑一次性**

```bash
# =============================================
# 1. 安装 Rust
# =============================================
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source ~/.cargo/env

# 2. 安装 Solana CLI
sh -c "$(curl -sSfL https://release.anza.xyz/stable/install)"

# 添加到 PATH (添加到 ~/.bashrc 或 ~/.zshrc)
echo 'export PATH="$HOME/.local/share/solana/install/active_release/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc

# 3. 验证安装
solana --version    # 应该显示版本号
rustc --version     # 应该显示版本号

# =============================================
# 4. 安装 Anchor Version Manager (AVM) 和 Anchor CLI
# =============================================
cargo install --git https://github.com/coral-xyz/anchor avm --force
avm install 0.32.1
avm use 0.32.1

# 5. 验证 Anchor
anchor --version    # 应该显示 anchor-cli 0.32.1

# =============================================
# 6. (可选) 安装 Surfpool - 本地开发环境
# =============================================
# Linux
sudo snap install surfpool

# macOS
brew install txtx/taps/surfpool

# 7. 创建 Solana 钱包 (如果没有)
solana-keygen new
```

---

## 第二步：创建新项目 📁 **每个项目一次性**

```bash
# 方式 A: 纯 Anchor 项目（推荐学习）
anchor init my-project
cd my-project

# 方式 B: 使用 Solana 全栈模板（包含 Next.js 前端）
npx create-solana-dapp my-project
cd my-project
```

---

## 第三步：项目结构

```
my-project/
├── anchor/                    # Anchor 程序目录
│   ├── Anchor.toml           # Anchor 配置文件
│   ├── Cargo.toml            # Rust 工作空间配置
│   ├── programs/
│   │   └── my-program/       # 你的 Solana 程序
│   │       ├── Cargo.toml
│   │       └── src/
│   │           └── lib.rs    # 程序代码
│   ├── tests/                # 测试文件目录
│   │   └── my-program.test.ts
│   └── target/               # 构建输出 (自动生成)
│       ├── deploy/           # .so 程序二进制
│       ├── idl/              # IDL JSON
│       └── types/            # TypeScript 类型
├── package.json
├── tsconfig.json
└── vitest.config.ts
```

---

## 第四步：安装测试依赖 📦 **每个项目一次性**

```bash
cd my-project

# =============================================
# 核心依赖
# =============================================
npm install @coral-xyz/anchor @solana/web3.js

# =============================================
# 测试框架 - 选择一种
# =============================================

# 方式 A: anchor-litesvm (推荐，更新更快)
npm install anchor-litesvm

# 方式 B: anchor-bankrun (旧版，但稳定)
# npm install anchor-bankrun

# =============================================
# 开发依赖
# =============================================
npm install -D vitest typescript @types/node
```

---

## 第五步：配置文件 ⚙️ **每个项目一次性**

### 5.1 vitest.config.ts

```typescript
import { defineConfig } from 'vitest/config'

export default defineConfig({
  test: {
    globals: true,           // 允许全局 describe, it, expect
    testTimeout: 30000,      // 30秒超时
  },
})
```

### 5.2 anchor/Anchor.toml

```toml
[toolchain]
anchor_version = "0.32.1"

[features]
resolution = true
skip-lint = false

[programs.localnet]
my_program = "你的程序ID"  # anchor keys list 查看

[provider]
cluster = "localnet"
wallet = "~/.config/solana/id.json"

[scripts]
test = "vitest"
```

### 5.3 tsconfig.json

```json
{
  "compilerOptions": {
    "target": "esnext",
    "module": "esnext",
    "moduleResolution": "bundler",
    "esModuleInterop": true,
    "resolveJsonModule": true,
    "strict": true,
    "skipLibCheck": true,
    "types": ["vitest/globals"]
  },
  "include": ["**/*.ts"],
  "exclude": ["node_modules"]
}
```

---

## 第六步：生成程序密钥 🔑 **每个程序一次性**

```bash
cd anchor

# 查看程序 ID (如果没有会自动生成)
anchor keys list
# 输出: my_program: XXXXXX...

# 同步密钥到代码 (更新 lib.rs 中的 declare_id!)
anchor keys sync
```

---

## 第七步：编写程序 (Rust) ✍️ **每次修改程序**

```rust
// anchor/programs/my-program/src/lib.rs
use anchor_lang::prelude::*;

declare_id!("你的程序ID");  // anchor keys sync 会自动更新

#[program]
pub mod my_program {
    use super::*;

    pub fn initialize(ctx: Context<Initialize>, data: u64) -> Result<()> {
        ctx.accounts.my_account.data = data;
        Ok(())
    }
}

#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(mut)]
    pub signer: Signer<'info>,
    
    #[account(
        init,
        payer = signer,
        space = 8 + 8,  // discriminator (8) + u64 (8)
        seeds = [b"my-seed"],
        bump
    )]
    pub my_account: Account<'info, MyAccount>,
    
    pub system_program: Program<'info, System>,
}

#[account]
pub struct MyAccount {
    pub data: u64,
}
```

---

## 第八步：构建程序 🔨 **每次修改程序后**

```bash
cd anchor
anchor build

# 输出文件:
# - target/deploy/my_program.so      (程序二进制)
# - target/idl/my_program.json       (IDL)
# - target/types/my_program.ts       (TypeScript 类型)
```

---

## 第九步：编写测试 🧪 **每次修改测试**

### 方式 A: 使用 anchor-litesvm (推荐)

```typescript
// anchor/tests/my-program.test.ts
import { fromWorkspace, LiteSVMProvider } from "anchor-litesvm";
import { Program } from "@coral-xyz/anchor";
import { Keypair, PublicKey } from "@solana/web3.js";
import * as anchor from "@coral-xyz/anchor";
import { MyProgram } from "../target/types/my_program";

const IDL = require("../target/idl/my_program.json");

describe("MyProgram", () => {
  let provider: LiteSVMProvider;
  let program: Program<MyProgram>;
  let payer: Keypair;

  beforeAll(async () => {
    const client = fromWorkspace("anchor");
    provider = new LiteSVMProvider(client);
    anchor.setProvider(provider);
    program = new Program<MyProgram>(IDL, provider);
    payer = provider.wallet.payer;
  });

  it("should initialize", async () => {
    const [pda] = PublicKey.findProgramAddressSync(
      [Buffer.from("my-seed")],
      program.programId
    );

    await program.methods
      .initialize(new anchor.BN(42))
      .accounts({ signer: payer.publicKey })
      .signers([payer])
      .rpc();

    const account = await program.account.myAccount.fetch(pda);
    expect(account.data.toNumber()).toBe(42);
  });
});
```

### 方式 B: 使用 anchor-bankrun

```typescript
// anchor/tests/my-program.test.ts
import { startAnchor, BankrunProvider } from "anchor-bankrun";
import { Program } from "@coral-xyz/anchor";
import { Keypair, PublicKey } from "@solana/web3.js";
import * as anchor from "@coral-xyz/anchor";
import { MyProgram } from "../target/types/my_program";

const IDL = require("../target/idl/my_program.json");
const PROGRAM_ID = new PublicKey("你的程序ID");

describe("MyProgram", () => {
  let provider: BankrunProvider;
  let program: Program<MyProgram>;
  let payer: Keypair;

  beforeAll(async () => {
    const context = await startAnchor(
      "anchor",  // anchor 目录路径
      [{ name: "my_program", programId: PROGRAM_ID }],
      []
    );
    
    provider = new BankrunProvider(context);
    anchor.setProvider(provider);
    program = new Program<MyProgram>(IDL, provider);
    payer = context.payer;
  });

  it("should initialize", async () => {
    const [pda] = PublicKey.findProgramAddressSync(
      [Buffer.from("my-seed")],
      PROGRAM_ID
    );

    await program.methods
      .initialize(new anchor.BN(42))
      .accounts({ signer: payer.publicKey })
      .signers([payer])
      .rpc();

    const account = await program.account.myAccount.fetch(pda);
    expect(account.data.toNumber()).toBe(42);
  });
});
```

---

## 第十步：运行测试 ▶️ **每次测试**

```bash
# 进入项目根目录
cd my-project

# 运行测试
npx vitest run

# 监听模式 (文件变化自动重跑)
npx vitest

# 运行特定测试
npx vitest run -t "should initialize"

# 详细输出
npx vitest run --reporter=verbose
```

---

## 常用命令速查表

| 命令               | 作用           | 频率                 |
| ------------------ | -------------- | -------------------- |
| `anchor build`     | 构建程序       | 每次修改 Rust 代码   |
| `anchor keys list` | 查看程序 ID    | 偶尔                 |
| `anchor keys sync` | 同步密钥到代码 | 新程序或重新生成密钥 |
| `npx vitest run`   | 运行测试       | 每次测试             |
| `npx vitest`       | 监听模式测试   | 开发时               |
| `surfpool start`   | 启动本地网络   | 需要时               |
| `anchor deploy`    | 部署到网络     | 部署时               |

---

## 操作频率总结

| 类别         | 操作                                   | 频率             |
| ------------ | -------------------------------------- | ---------------- |
| 🖥️ **电脑级** | Rust, Solana CLI, Anchor CLI, Surfpool | 一次性安装       |
| 📁 **项目级** | 创建项目, npm install, 配置文件        | 每个新项目一次   |
| 🔑 **程序级** | anchor keys sync                       | 每个新程序一次   |
| 🔨 **开发级** | anchor build                           | 每次修改 Rust 后 |
| 🧪 **测试级** | npx vitest run                         | 每次测试         |

---

# 第二部分：前端开发 (Next.js + React)

---

## 第十一步：前端项目结构 📁

使用 `create-solana-dapp` 创建的项目包含完整的前端：

```
my-project/
├── anchor/                           # 合约部分
│   ├── programs/                     # Rust 程序
│   ├── src/
│   │   ├── client/js/generated/      # Codama 自动生成的 SDK
│   │   └── crudapp-exports.ts        # 导出给前端用
│   └── target/idl/                   # IDL 文件
│
├── src/                              # Next.js 前端
│   ├── app/                          # 路由页面
│   ├── components/                   # 通用组件
│   │   ├── ui/                       # 基础 UI（按钮、卡片）
│   │   └── solana/                   # Solana 相关
│   └── features/                     # ⭐ 按功能模块组织
│       └── crudapp/
│           ├── data-access/          # 数据访问层
│           ├── ui/                   # UI 组件层
│           └── crudapp-feature.tsx   # 主页面
```

---

## 第十二步：理解三层架构 🏗️

```
┌─────────────────────────────────────────────────────────────────┐
│  UI 组件层 (ui/)                                                 │
│  └── 按钮、表单、卡片 - 用户看到和交互的部分                       │
│           │                                                      │
│           ▼                                                      │
├─────────────────────────────────────────────────────────────────┤
│  数据访问层 (data-access/)                                       │
│  └── React Query Hooks - 调用 SDK，管理缓存，处理成功/失败         │
│           │                                                      │
│           ▼                                                      │
├─────────────────────────────────────────────────────────────────┤
│  SDK 层 (anchor/src/client/js/generated/)                        │
│  └── Codama 自动生成 - 构建交易指令，类型安全                      │
│           │                                                      │
│           ▼                                                      │
├─────────────────────────────────────────────────────────────────┤
│  链上合约 (anchor/programs/)                                     │
│  └── Rust 程序 - 执行业务逻辑                                     │
└─────────────────────────────────────────────────────────────────┘
```

---

## 第十三步：生成 SDK (Codama) 🔧

每次修改合约后，需要重新生成 SDK：

```bash
# 1. 构建合约
npm run anchor-build

# 2. 生成 TypeScript SDK
npm run codama:js

# 或者一键完成
npm run setup
```

生成的文件：
```
anchor/src/client/js/generated/
├── instructions/
│   ├── createJournalEntry.ts    # 创建指令
│   ├── updateJournalEntry.ts    # 更新指令
│   └── deleteJournalEntry.ts    # 删除指令
├── accounts/
│   └── journalEntryState.ts     # 账户类型
└── programs/
    └── crudapp.ts               # 程序地址
```

---

## 第十四步：编写导出层 📤

`anchor/src/crudapp-exports.ts` - 封装给前端使用：

```typescript
import { Account, getBase58Decoder, SolanaClient } from 'gill'
import { getProgramAccountsDecoded } from './helpers/get-program-accounts-decoded'
import {
  JournalEntryState,
  JOURNAL_ENTRY_STATE_DISCRIMINATOR,
  CRUDAPP_PROGRAM_ADDRESS,
  getJournalEntryStateDecoder,
} from './client/js'

// 定义前端使用的账户类型
export type JournalEntryAccount = Account<JournalEntryState, string>

// 导出自动生成的 SDK
export * from './client/js'

// 查询所有账户的便捷函数
export function getJournalEntryAccounts(rpc: SolanaClient['rpc']) {
  return getProgramAccountsDecoded(rpc, {
    decoder: getJournalEntryStateDecoder(),
    filter: getBase58Decoder().decode(JOURNAL_ENTRY_STATE_DISCRIMINATOR),
    programAddress: CRUDAPP_PROGRAM_ADDRESS,
  })
}
```

---

## 第十五步：编写数据访问层 🔌

### 15.1 查询 Hook

`src/features/crudapp/data-access/use-crudapp-accounts-query.ts`：

```typescript
import { useSolana } from '@/components/solana/use-solana'
import { useQuery } from '@tanstack/react-query'
import { getJournalEntryAccounts } from '@project/anchor'

export function useJournalEntriesQuery() {
  const { client } = useSolana()

  return useQuery({
    queryKey: ['crudapp', 'accounts'],
    queryFn: async () => await getJournalEntryAccounts(client.rpc),
  })
}
```

### 15.2 创建 Mutation

`src/features/crudapp/data-access/use-crudapp-initialize-mutation.ts`：

```typescript
import { useMutation, useQueryClient } from '@tanstack/react-query'
import { UiWalletAccount, useWalletUiSigner } from '@wallet-ui/react'
import { useWalletUiSignAndSend } from '@wallet-ui/react-gill'
import { getCreateJournalEntryInstructionAsync } from '@project/anchor'

export function useCreateJournalEntryMutation({ account }: { account: UiWalletAccount }) {
  const queryClient = useQueryClient()
  const signer = useWalletUiSigner({ account })
  const signAndSend = useWalletUiSignAndSend()

  return useMutation({
    mutationFn: async ({ title, message }: { title: string; message: string }) => {
      // 1. 调用 SDK 构建指令
      const instruction = await getCreateJournalEntryInstructionAsync({
        owner: signer,
        title,
        message,
      })
      // 2. 签名并发送交易
      return await signAndSend(instruction, signer)
    },
    onSuccess: async () => {
      // 3. 刷新缓存
      await queryClient.invalidateQueries({ queryKey: ['crudapp', 'accounts'] })
    },
  })
}
```

### 15.3 更新 Mutation

```typescript
export function useUpdateJournalEntryMutation({ account, entry }) {
  const signAndSend = useWalletUiSignAndSend()
  const signer = useWalletUiSigner({ account })

  return useMutation({
    mutationFn: async ({ newMessage }: { newMessage: string }) => {
      const instruction = await getUpdateJournalEntryInstructionAsync({
        owner: signer,
        title: entry.data.title,  // 用现有标题找到账户
        newMessage,
      })
      return await signAndSend(instruction, signer)
    },
  })
}
```

### 15.4 删除 Mutation

```typescript
export function useDeleteJournalEntryMutation({ account, entry }) {
  const signAndSend = useWalletUiSignAndSend()
  const signer = useWalletUiSigner({ account })

  return useMutation({
    mutationFn: async () => {
      const instruction = await getDeleteJournalEntryInstructionAsync({
        owner: signer,
        title: entry.data.title,
      })
      return await signAndSend(instruction, signer)
    },
  })
}
```

---

## 第十六步：编写 UI 组件 🎨

### 16.1 列表组件

```typescript
// src/features/crudapp/ui/crudapp-ui-list.tsx
export function JournalEntryList({ account }) {
  const query = useJournalEntriesQuery()

  if (query.isLoading) return <span>Loading...</span>
  if (!query.data?.length) return <div>No entries found</div>

  return (
    <div className="grid gap-4">
      {query.data.map((entry) => (
        <JournalEntryCard key={entry.address} entry={entry} account={account} />
      ))}
    </div>
  )
}
```

### 16.2 卡片组件

```typescript
// src/features/crudapp/ui/crudapp-ui-card.tsx
export function JournalEntryCard({ account, entry }) {
  return (
    <Card>
      <CardHeader>
        <CardTitle>{entry.data.title}</CardTitle>
      </CardHeader>
      <CardContent>
        <p>{entry.data.body}</p>
        <div className="flex gap-4">
          <UpdateButton account={account} entry={entry} />
          <DeleteButton account={account} entry={entry} />
        </div>
      </CardContent>
    </Card>
  )
}
```

### 16.3 创建按钮

```typescript
// src/features/crudapp/ui/crudapp-ui-button-initialize.tsx
export function CreateJournalEntryButton({ account }) {
  const mutation = useCreateJournalEntryMutation({ account })
  const [title, setTitle] = useState('')
  const [message, setMessage] = useState('')

  return (
    <AppModal title="Create Journal Entry">
      <Input value={title} onChange={(e) => setTitle(e.target.value)} />
      <Textarea value={message} onChange={(e) => setMessage(e.target.value)} />
      <Button onClick={() => mutation.mutateAsync({ title, message })}>
        Create
      </Button>
    </AppModal>
  )
}
```

---

## 第十七步：运行前端 🚀

```bash
# 1. 确保本地验证器运行
solana-test-validator

# 2. 部署合约
cd anchor && anchor deploy

# 3. 启动前端开发服务器
npm run dev

# 4. 打开浏览器
# http://localhost:3000
```

---

## 第十八步：连接钱包测试 🔗

1. 安装 **Phantom** 钱包浏览器扩展
2. 打开 Phantom → 设置 → Developer Settings → 开启 Testnet Mode → 选择 **Localhost**
3. 给钱包地址空投 SOL：
   ```bash
   solana airdrop 5 <你的Phantom钱包地址> --url localhost
   ```
4. 刷新页面，连接钱包，开始测试 CRUD 操作！

---

# 第三部分：项目部署

---

## 第十九步：部署到 Devnet 🌐

```bash
# 1. 切换到 Devnet
solana config set --url devnet

# 2. 空投 SOL（测试用）
solana airdrop 2

# 3. 部署合约
cd anchor && anchor deploy --provider.cluster devnet

# 4. 前端切换到 Devnet
# 在网页右上角的网络选择器中选择 Devnet
```

---

# 附录

---

## 完整初始化脚本 (一键复制) 🚀

```bash
#!/bin/bash
# 新建 Anchor 项目并配置测试 (使用 anchor-litesvm)

PROJECT_NAME="${1:-my-solana-project}"

# 创建项目
anchor init $PROJECT_NAME
cd $PROJECT_NAME

# 安装依赖
npm install @coral-xyz/anchor @solana/web3.js anchor-litesvm
npm install -D vitest typescript @types/node

# 创建 vitest 配置
cat > vitest.config.ts << 'EOF'
import { defineConfig } from 'vitest/config'
export default defineConfig({
  test: { globals: true, testTimeout: 30000 }
})
EOF

# 修改 Anchor.toml 使用 vitest
sed -i 's/test = "yarn run ts-mocha.*/test = "vitest"/' Anchor.toml

# 构建
anchor build

echo "✅ 项目 $PROJECT_NAME 创建完成！"
echo "📂 cd $PROJECT_NAME"
echo "🧪 运行测试: npx vitest run"
```

使用方法：
```bash
chmod +x init-anchor.sh
./init-anchor.sh my-voting-app
```

---

## 前端技术栈说明 📚

| 技术               | 用途                              |
| ------------------ | --------------------------------- |
| **Next.js 15**     | React 框架，支持 SSR              |
| **TanStack Query** | 数据获取和缓存管理                |
| **Wallet UI**      | 钱包连接组件                      |
| **Gill**           | Solana SDK (新版 @solana/web3.js) |
| **Codama**         | 自动生成 TypeScript SDK           |
| **shadcn/ui**      | UI 组件库                         |
| **Tailwind CSS**   | 样式框架                          |

---

## 常见问题 FAQ ❓

### Q: 点击按钮没反应？
1. 检查 Phantom 钱包是否切换到 Localhost
2. 检查钱包是否有 SOL
3. 打开 F12 查看控制台错误

### Q: 程序 ID 不匹配？
```bash
anchor keys sync
anchor build
npm run codama:js
```

### Q: 交易失败？
1. 确保本地验证器运行中
2. 确保合约已部署
3. 检查账户是否有足够 SOL

---
