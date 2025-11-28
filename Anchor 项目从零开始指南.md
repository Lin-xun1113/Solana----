# Anchor 项目从零开始完整指南 (2025 版)

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

## 测试库对比

| 特性         | anchor-litesvm    | anchor-bankrun                |
| ------------ | ----------------- | ----------------------------- |
| **状态**     | ✅ 活跃维护        | ⚠️ 依赖已弃用的 solana-bankrun |
| **速度**     | 更快              | 快                            |
| **Provider** | `LiteSVMProvider` | `BankrunProvider`             |
| **初始化**   | `fromWorkspace()` | `startAnchor()`               |
| **推荐**     | ✅ 新项目推荐      | 现有项目可继续用              |

---

**建议**：新项目直接使用 `anchor-litesvm`，已有项目如果 `anchor-bankrun` 能跑就先用着。