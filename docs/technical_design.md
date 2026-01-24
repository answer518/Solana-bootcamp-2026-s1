# Artifex项目技术方案设计文档

## 1. 概述

本文档旨在为“Artifex - 您的数字潮玩藏品架”项目提供一份全面的技术实现方案。方案基于 `project_plan_v2.md` 中定义的需求，重点围绕在Solana区块链上实现一个具有“可组合进化”特性的NFT项目。

## 2. 系统架构

项目采用前后端分离的DApp架构，主要由三部分组成：

1.  **前端 (Frontend DApp):** 一个基于Next.js的Web应用，用户通过它可以连接钱包、铸造NFT、浏览藏品以及执行核心的“组合进化”操作。
2.  **链上程序 (On-chain Program):** 部署在Solana区块链上的智能合约，使用Rust和Anchor框架开发。它负责处理所有与资产相关的核心逻辑，包括NFT的铸造、销毁和组合。
3.  **去中心化存储 (Decentralized Storage):** 使用Arweave存储所有的NFT图片和元数据，确保资产的永久性和不可变性。

### 交互流程图

```
+----------------+      (Connect Wallet)      +-------------------+      (Read NFT Data)      +-------------------+
|                | <------------------------> |                   | ----------------------> |                   |
|   User/Browser |                            |  Frontend DApp    |                         | Solana Blockchain |
|                |      (Send Transaction)    |    (Next.js)      |      (Invoke Program)   | (Ledger)          |
+----------------+ --------------------------> |                   | ----------------------> +-------------------+
                                               +-------------------+            ^            |                   |
                                                        |                       |            |  Artifex Program  |
                                                        | (Read Metadata)       | (CPI)      |   (Anchor/Rust)   |
                                                        |                       |            +-------------------+
                                                        v                       |                       ^
                                               +-------------------+            |                       | (CPI)
                                               |                   |            +-----------------------+
                                               |      Arweave      |
                                               | (Image/JSON Data) |
                                               +-------------------+
```

## 3. 前端 (DApp) 设计

*   **框架:** **Next.js + TypeScript**
    *   **理由:** Next.js提供优秀的开发体验和性能优化（如图片优化、路由），TypeScript提供类型安全，减少运行时错误，是现代Web3 DApp开发的黄金组合。
*   **核心库:**
    *   **`@solana/wallet-adapter`:** 用于快速集成Phantom, Solflare等主流钱包，提供统一的连接和签名接口。
    *   **`@solana/web3.js`:** Solana官方库，用于构建和发送交易，与链上程序进行底层交互。
    *   **`@project-serum/anchor`:** Anchor框架的客户端库，可以根据合约的IDL自动生成类型安全的客户端，极大简化与合约的交互代码。
    *   **UI组件库:** **Tailwind CSS**。推荐使用，它与Solana官方模板保持一致，提供了高度的定制化能力。
*   **项目结构 (参考 `solana-foundation/templates/kit/nextjs-anchor`):**
    *   **`anchor/`:** Anchor智能合约的工作区。
        *   `programs/artifex/`: 存放项目核心的链上程序（Rust代码）。这里将实现 `mint_base_nft` 和 `combine_nfts` 等指令。
        *   `tests/`: 合约的测试脚本。
    *   **`app/`:** Next.js 14+ 的 App Router 目录，作为DApp的前端。
        *   `components/`: 存放React组件。
            *   `ui/`: 基础UI组件，如按钮、对话框等。
            *   `layout/`: 页面布局组件，如导航栏 `Navbar`。
            *   `solana/`: 与Solana交互相关的组件，如钱包连接按钮。
            *   `feature-mint/`: 专门处理铸造逻辑的组件。
            *   `feature-combine/`: 处理组合进化逻辑的组件。
        *   `generated/artifex/`: **自动生成**的合约客户端。当合约编译后，会根据IDL（接口定义语言）文件生成类型安全的TypeScript代码，极大简化前端与合约的交互。
        *   `providers/`: 全局上下文提供者，如封装了 `@solana/wallet-adapter-react` 的 `WalletProvider`。
        *   `lib/`: 存放工具函数和常量。
        *   `app/(routes)/page.tsx`: 具体的页面路由。
    *   **`codama.json`:** 用于配置自动生成客户端的工具 `Codama` 的文件。
*   **页面/组件拆分:**
    *   **`HomePage` (`/`):** 项目介绍、IP故事、核心功能入口（铸造、我的藏品、组合进化）。
    *   **`MintPage` (`/mint`):** 包含 `feature-mint` 组件，负责展示基础款NFT、剩余数量和铸造功能。
    *   **`CollectionPage` (`/collection`):** 用户连接钱包后，通过RPC调用获取并展示用户持有的本项目NFT。
    *   **`CombinePage` (`/combine`, 核心页面):** 包含 `feature-combine` 组件，允许用户选择可合成的NFT，预览产物，并发送“组合进化”交易。

## 4. 链上程序 (Smart Contract) 设计

*   **框架:** **Anchor (Rust)**
    *   **理由:** Anchor极大地简化了Solana程序的开发，处理了账户序列化/反序列化、指令分发等繁琐工作，并内置了安全检查，是当前Solana开发的首选。
*   **核心指令 (Instructions):**
    *   **`mint_base_nft(ctx)`:**
        *   **功能:** 铸造一个基础款NFT。
        *   **逻辑:**
            1.  接收用户的调用，验证付费等逻辑。
            2.  通过跨程序调用 (CPI) 调用Metaplex的Token Metadata程序。
            3.  创建新的Mint账户、Token账户，并为新Mint的Token附加元数据（元数据URI指向预先上传到Arweave的JSON文件）。
    *   **`combine_nfts(ctx, recipe_id)`:**
        *   **功能:** 销毁多个基础NFT，并铸造一个新的进化版NFT。这是项目的核心技术亮点。
        *   **账户上下文 (`Context`):** 需要传入用户钱包、用于销毁的多个NFT的Token账户和Mint账户、新NFT的Mint账户、以及必要的系统程序和Token程序地址。
        *   **逻辑:**
            1.  **验证 (Validation):**
                *   **所有权验证:** 检查指令的调用者（`signer`）是否是所有待销毁NFT的实际拥有者。
                *   **配方验证:** 根据传入的 `recipe_id` 或直接检查传入的NFT Mint地址，确认这是一个有效的组合“配方”（例如，`[mint_A, mint_B]` 确实可以合成 `mint_C`）。这个配方逻辑可以硬编码在合约中。
                *   **Collection验证:** 确保所有传入的NFT都属于本项目的官方Collection。
            2.  **销毁 (Burn):**
                *   对每一个待销毁的NFT，通过CPI调用 `spl_token::instruction::burn` 指令，将其Token账户中的数量从1变为0。
            3.  **铸造 (Mint):**
                *   与 `mint_base_nft` 类似，通过CPI调用Metaplex Token Metadata程序，铸造一个新的、进化版的NFT。其元数据URI指向进化版NFT的Arweave地址。
*   **账户结构:**
    *   需要定义一个全局的配置账户（`Config`），用于存储管理员地址、铸造价格等信息。
    *   每个指令都需要精确定义其所需的账户结构（`#[derive(Accounts)]`），Anchor会据此进行安全校验。

## 5. NFT元数据与存储方案

*   **流程:**
    1.  **艺术品创作:** 设计并导出所有基础款和进化版的NFT图片（PNG, GIF等）。
    2.  **图片上传:** 使用 `bundlr-cli` 工具将所有图片文件批量上传到Arweave，获得每一个图片的永久URL。Bundlr Network可以极大提升上传速度和体验。
    3.  **元数据生成:** 编写脚本，为每一个NFT（包括基础款和进化版）生成一个JSON元数据文件。文件格式遵循Metaplex标准，`image` 字段填入上一步获得的Arweave URL，并在 `attributes` 中定义其特性。
    4.  **元数据上传:** 将所有JSON文件上传到Arweave，获得每一个元数据文件的永久URL。
*   **关键点:** 在调用智能合约的铸造指令时，传入的 `uri` 参数就是这些元数据JSON文件在Arweave上的URL。

## 6. 部署与环境

*   **开发环境:** 使用 `solana-test-validator` 在本地进行快速开发和测试。
*   **测试网:** 在部署到主网前，将程序部署到Solana **Devnet**，并部署前端DApp到 **Vercel**，进行公开的、端到端的测试。
*   **主网:** 确认所有功能稳定后，将程序部署到 **Mainnet-beta**。

此技术方案为Artifex项目提供了一个清晰、可行且能在黑客松时间内突出亮点的实现路径。
