
# Artifex 项目技术架构设计文档

## 1. 概述

本文档旨在为 "Artifex - 您的数字潮玩藏品架" 项目提供全面的技术架构指导。基于项目规划文档 (`project_plan_v2.md`) 的核心需求，本文档将详细阐述技术栈选型、项目结构、开发规范、合约开发建议等，以确保项目开发流程的标准化、高效性和高质量。

## 2. 技术栈选型 (Technology Stack)

为了实现一个高性能、用户体验优秀且安全可靠的 DApp，我们规划了以下技术栈：

*   **区块链 (Blockchain):** **Solana**
    *   **理由:** 凭借其高吞吐量 (TPS)、低交易成本和快速确认时间，Solana 是构建消费级 NFT 应用的最佳公链之一。

*   **前端 (Frontend):** **Next.js (with React & TypeScript)**
    *   **理由:**
        *   **React 生态:** 拥有最庞大的开发者社区和最丰富的组件库。
        *   **Next.js 框架:** 提供服务器端渲染 (SSR) 和静态站点生成 (SSG)，极大利于 SEO 和首屏加载速度。其基于文件系统的路由（App Router）也让项目结构更清晰。
        *   **TypeScript:** 为项目提供静态类型检查，能显著提升代码的健壮性、可维护性，并减少运行时错误。

*   **UI 组件库 (UI Library):** **Tailwind CSS**
    *   **理由:** 作为一个 "Utility-First" 的 CSS 框架，它能让我们快速构建现代化、响应式的界面，而无需离开 HTML (JSX) 代码，极大提升开发效率。

*   **智能合约 (Smart Contract):** **Rust + Anchor Framework**
    *   **理由:**
        *   **Rust:** 兼具内存安全和高性能，是 Solana 智能合约开发的首选语言。
        *   **Anchor:** 一个极大地简化了 Solana Rust 程序开发的框架。它通过宏和接口定义语言 (IDL) 抽象了大量底层复杂性（如账户序列化/反序列化、指令分发），让开发者能更专注于业务逻辑。

*   **钱包集成 (Wallet Integration):** **Solana Wallet-Adapter**
    *   **理由:** 官方推荐的解决方案，提供了一套统一的、模块化的适配器，可以轻松集成 Phantom, Solflare, Ledger 等几乎所有主流 Solana 钱包。

*   **去中心化存储 (Decentralized Storage):** **Arweave**
    *   **理由:** 提供一次性付费、永久存储的解决方案。将 NFT 的图片和元数据存储在 Arweave 上，是保证数字资产永久性、抗审查性的行业黄金标准。

*   **前端状态管理 (State Management):** **Zustand / Jotai**
    *   **理由:** 对于 DApp 来说，需要管理的全局状态（如钱包连接状态、用户持有的NFT列表等）会逐渐增多。相比 React 原生的 Context API，Zustand 或 Jotai 提供了更简洁、高效的状态管理方案，能有效避免不必要的组件重渲染。

## 3. 项目模板推荐

**推荐模板:** `solana-foundation/templates/kit/nextjs-anchor`

**获取方式:**
```bash
npx create-solana-dapp@latest -t solana-foundation/templates/kit/nextjs-anchor your-project-name
```

**推荐原因:**

1.  **官方最佳实践:** 这是由 Solana 基金会官方提供的 "Kit" (套件) 级模板，整合了开发一个完整 DApp 所需的大部分要素，是开始新项目的最佳起点。
2.  **Monorepo 结构:** 模板预置了单体仓库（Monorepo）结构，将链上程序 (`anchor`) 和前端应用 (`app`) 放在同一个代码库中管理。这种结构极大地方便了代码共享、统一构建和部署流程。
3.  **技术栈完美匹配:** 该模板完美集成了我们选择的技术栈：Next.js, TypeScript, Tailwind CSS, Anchor, 以及 Solana Wallet-Adapter。开箱即用，无需手动配置。
4.  **自动化代码生成:** 模板内置了脚本，可以在编译 Anchor 合约后，自动生成用于前端调用的客户端代码和类型定义，实现了前后端的无缝类型安全。

## 4. 项目目录结构

基于推荐的 `nextjs-anchor` 模板，项目将采用以下目录结构：

```
/your-project-name
├── anchor/                  # 链上程序 (Anchor 项目)
│   ├── programs/
│   │   └── artifex/         # 我们的 Artifex 合约源码
│   │       └── src/
│   │           └── lib.rs   # 合约主逻辑
│   ├── tests/               # Anchor 集成测试 (TypeScript)
│   │   └── artifex.ts
│   ├── Anchor.toml          # Anchor 项目配置
│   └── Cargo.toml           # Rust 依赖配置
│
├── app/                     # 前端应用 (Next.js 项目)
│   ├── src/                 # Next.js 13+ 的 app router 目录
│   │   ├── app/             # 页面路由
│   │   ├── components/      # 可复用 React 组件 (UI, 业务)
│   │   ├── lib/             # 辅助函数、Solana连接逻辑等
│   │   └── generated/       # 自动生成的合约客户端代码
│   ├── public/              # 静态资源 (图片, 字体)
│   └── ...                  # Next.js 其他配置文件
│
├── node_modules/            # 项目依赖
├── package.json             # Monorepo 根配置和脚本
└── ...                      # 其他顶层配置文件 (.prettierrc, .eslintrc.json)
```

## 5. 项目开发规范

为了保证团队协作效率和项目代码质量，建议遵循以下规范：

*   **代码风格 (Code Style):**
    *   **自动格式化:** 强制使用 `Prettier` 进行代码格式化。在提交代码前自动运行，确保所有代码风格统一。
    *   **代码质量检查:** 使用 `ESLint` (前端) 和 `clippy` (Rust) 进行静态代码分析，遵循推荐规则，及时发现潜在问题。

*   **Commit 规范 (Commit Messages):**
    *   **遵循 Conventional Commits:** 提交信息应采用格式化的规范，如 `feat:`, `fix:`, `docs:`, `style:`, `refactor:`, `test:`, `chore:`。
    *   **示例:** `feat(mint): implement whitelist mint logic`
    *   **好处:** 使提交历史清晰可读，便于自动化生成 `CHANGELOG` 和版本管理。

*   **分支管理 (Branching Strategy):**
    *   **采用 GitFlow (简化版):**
        *   `main`: 主分支，只存放稳定、可随时发布的代码。
        *   `develop`: 开发分支，所有功能开发的基准分支，汇集了所有已完成的功能。
        *   `feat/feature-name`: 功能分支，每个新功能或模块都在独立的分支上开发，完成后合并到 `develop`。
        *   `fix/bug-name`: 修复分支，用于紧急修复 `main` 分支上的 bug。

## 6. NFT 合约开发建议

*   **严格遵循 Metaplex 标准:**
    *   **Token Metadata:** 必须使用 `mpl-token-metadata` crate 来创建和管理 NFT 的元数据。这是确保 NFT 能在所有 Solana 生态（钱包、市场）中被正确识别和展示的唯一途径。
    *   **Collection NFT:** 善用 Metaplex 的 "Collection NFT" 标准来组织系列。将每个铸造的 NFT 设置为某个“合集 NFT”的成员，可以方便地进行系列验证和聚合展示。

*   **安全是第一要务:**
    *   **善用 Anchor 约束:** 在 `#[derive(Accounts)]` 结构体中，充分利用 `init`, `mut`, `has_one`, `seeds`, `bump` 等约束来验证账户关系和权限，将大部分安全检查前置到指令执行之前。
    *   **权限控制:** 对于关键操作（如修改费用、提取收益），务必添加 `has_one = authority` 等检查，确保只有合约的管理员才能执行。
    *   **防范重放攻击:** 对于需要唯一性的操作（如每个用户只能 mint 一次），可以创建一个以用户钱包地址为 `seed` 的 PDA 账户作为凭证，防止重复执行。

*   **为“可组合性”设计:**
    *   **合成/销毁逻辑:** 在设计“合成”功能时，指令需要原子化地完成以下操作：
        1.  接收用户拥有的多个基础 NFT 的账户信息。
        2.  验证这些 NFT 确实属于该用户，且符合合成规则。
        3.  通过 CPI (Cross-Program Invocation) 调用 `token-program` 的 `burn` 指令销毁这些基础 NFT 的代币。
        4.  通过 CPI 调用 `token-program` 和 `mpl-token-metadata` 的相关指令，铸造一个新的“合成品”NFT 给用户。
    *   **状态管理:** 合成规则（如哪些 NFT 可以合成什么）可以硬编码在合约中，也可以存储在某个配置账户中，以便未来可以灵活更新。

*   **编写全面的测试:**
    *   **单元测试 (Rust):** 对合约内部的纯逻辑函数（如果有）编写单元测试。
    *   **集成测试 (TypeScript):** 使用 `anchor test` 框架，为每一条指令编写详尽的测试用例，覆盖所有成功路径和预期的失败场景（如权限不足、账户不匹配、余额不足等）。这是保障合约质量最关键的一环。
