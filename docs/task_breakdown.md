# Artifex 项目任务拆分计划 (Monorepo 版本)

本文档为“Artifex”项目提供了一个详细的、分步执行的开发计划。此版本基于 `solana-foundation/templates/kit/nextjs-anchor` 的单体仓库（Monorepo）最佳实践进行设计，确保开发流程的高效、自动化和可维护性。

---

## 阶段 0: 环境与项目初始化

**目标:** 使用官方模板创建项目骨架，并将其调整为我们 Artifex 项目所需的结构。

*   **任务 0.1: 使用官方模板创建项目**
    *   **操作:** 在工作区根目录，运行 `npx -y create-solana-dapp@latest -t solana-foundation/templates/kit/nextjs-anchor finalProject`。
    *   **产出:** 一个名为 `finalProject` 的文件夹被创建，内部包含 `anchor` 和 `app` 两个核心目录，以及完整的 monorepo 配置。
    *   **Commit Message:** `feat: 初始化项目，基于官方 nextjs-anchor 模板`

*   **任务 0.2: 安装依赖并完成初始设置**
    *   **操作:** 进入 `finalProject` 目录，运行 `npm install`。此命令会自动安装所有前后端依赖，并首次编译合约、生成客户端。
    *   **产出:** 所有依赖安装完成，`app/generated/` 目录下出现 `vault` 客户端代码，项目可以无误地启动。
    *   **Commit Message:** `chore: 安装项目依赖并生成初始客户端`

---

## 阶段 1: 链上程序核心功能开发

**目标:** 完成智能合约的两个核心指令 `mint_base_nft` 和 `combine_nfts`，并编写测试脚本验证其正确性。

*   **任务 1.1: 实现基础NFT铸造指令 (`mint_base_nft`)**
    *   **操作:**
        1.  在 `anchor/programs/vault/src/lib.rs` 中，定义 `mint_base_nft` 指令的上下文和处理函数。
        2.  实现通过CPI调用Metaplex Token Metadata程序来创建和铸造一个新NFT的逻辑。URI可以暂时使用一个占位符字符串。
    *   **产出:** 在 `anchor` 目录下运行 `anchor build`，智能合约编译成功。
    *   **Commit Message:** `feat(program): 实现基础 NFT 铸造指令 (mint_base_nft)`

*   **任务 1.2: 编写 `mint_base_nft` 的测试脚本**
    *   **操作:** 在 `anchor/tests/` 目录下，修改或创建一个测试脚本，该脚本可以部署合约，并成功调用 `mint_base_nft` 指令。
    *   **产出:** 在 `anchor` 目录下运行 `anchor test`，测试通过。
    *   **Commit Message:** `test(program): 添加 mint_base_nft 指令的测试用例`

*   **任务 1.3: 实现NFT组合进化指令 (`combine_nfts`)**
    *   **操作:**
        1.  在 `anchor/programs/vault/src/lib.rs` 中，定义 `combine_nfts` 指令。
        2.  实现其核心逻辑：验证用户所有权 -> 根据配方验证输入的NFTs -> 通过CPI销毁输入的NFTs -> 通过CPI铸造一个新的进化版NFT。
        3.  配方逻辑可以暂时硬编码在合约中。
    *   **产出:** 在 `anchor` 目录下运行 `anchor build`，智能合约再次编译成功。
    *   **Commit Message:** `feat(program): 实现 NFT 组合进化指令 (combine_nfts)`

*   **任务 1.4: 编写 `combine_nfts` 的测试脚本**
    *   **操作:** 在 `anchor/tests/` 目录下，编写一个测试脚本，模拟完整流程：先铸造两个“材料”NFT，然后调用 `combine_nfts` 将它们销毁，并生成一个新的“进化”NFT。
    *   **产出:** 在 `anchor` 目录下运行 `anchor test`，测试通过。
    *   **Commit Message:** `test(program): 添加 combine_nfts 指令的端到端测试`

---

## 阶段 2: 前端DApp基础功能搭建

**目标:** 搭建DApp的基础UI，实现钱包连接和NFT数据显示。

*   **任务 2.1: 搭建基础布局与钱包连接**
    *   **操作:**
        1.  在 `app/components/` 目录下创建 `layout` 文件夹，并添加 `Navbar.tsx` 组件。
        2.  在 `app/layout.tsx` 中应用 `Navbar` 和项目已有的 `WalletProvider`。
        3.  在 `Navbar` 中添加 `@solana/wallet-adapter-react-ui` 提供的 `WalletMultiButton` 组件。
    *   **产出:** 启动前端应用 (`npm run dev`)，页面显示导航栏，用户可以点击按钮成功连接和断开Solana钱包。
    *   **Commit Message:** `feat(app): 搭建基础布局并集成钱包连接功能`

*   **任务 2.2: 创建“我的藏品”页面 (`/collection`)**
    *   **操作:**
        1.  创建 `app/collection/page.tsx` 路由。
        2.  在该页面中，获取已连接钱包的公钥，使用 `@solana/web3.js` 的 `getParsedTokenAccountsByOwner` 获取用户代币账户。
        3.  安装 `@metaplex-foundation/js` 依赖，用于读取并解析每个NFT的链上元数据。
        4.  将获取到的NFT图片和名称以网格布局的形式展示出来。
    *   **产出:** 当用户连接钱包后，访问 `/collection` 页面可以看到其钱包中拥有的NFT列表。
    *   **Commit Message:** `feat(app): 创建“我的藏品”页面，实现 NFT 数据获取与展示`

---

## 阶段 3: 核心业务功能端到端集成

**目标:** 将前后端连接起来，实现完整的铸造和组合进化流程。

*   **任务 3.1: 实现“铸造”页面 (`/mint`)**
    *   **操作:**
        1.  创建 `app/mint/page.tsx` 路由。
        2.  在 `finalProject` 根目录运行 `npm run setup` 确保前端拥有最新的合约客户端。
        3.  在页面上添加“Mint NFT”按钮，点击时，使用 `app/generated/artifex` 中自动生成的客户端调用链上程序的 `mint_base_nft` 指令。
    *   **产出:** 在本地启动 `solana-test-validator` 后，用户可以在 `/mint` 页面成功铸造一个NFT，并能在 `/collection` 页面立即看到它。
    *   **Commit Message:** `feat(app): 实现铸造页面，完成前后端铸造流程集成`

*   **任务 3.2: 实现“组合进化”页面 (`/combine`)**
    *   **操作:**
        1.  创建 `app/combine/page.tsx` 路由。
        2.  UI上创建“材料选择区”，展示用户拥有的、可用于合成的基础款NFT。
        3.  添加“Combine”按钮，当用户选择有效组合后，调用 `app/generated/artifex` 客户端中的 `combine_nfts` 指令。
    *   **产出:** 用户可以在 `/combine` 页面选择NFT执行组合操作，旧NFT消失，新的进化版NFT出现在 `/collection` 页面。**这是项目的核心亮点演示。**
    *   **Commit Message:** `feat(app): 实现组合进化页面，完成核心玩法端到端集成`

---

## 阶段 4: 部署与最终完善

**目标:** 将项目部署到公开可访问的开发网，并完成最终的文档。

*   **任务 4.1: 准备并上传最终的NFT资产**
    *   **操作:**
        1.  创建最终版的NFT图片及JSON元数据文件。
        2.  使用 `bundlr-cli` 上传到Arweave。
        3.  将获得的URI更新到链上程序的相应位置。
    *   **产出:** DApp中展示的NFT拥有了最终的、永久存储的艺术形象和属性。
    *   **Commit Message:** `chore: 准备并上传最终 NFT 资产至 Arweave`

*   **任务 4.2: 部署到Devnet**
    *   **操作:**
        1.  在 `anchor` 目录下运行 `anchor deploy --provider devnet` 部署合约。
        2.  运行 `anchor keys sync` 将新的Program ID同步到 `Anchor.toml`。
        3.  在 `finalProject` 根目录运行 `npm run setup`，用新的Program ID重新生成客户端。
        4.  将 `finalProject` 应用部署到Vercel。
    *   **产出:** 整个DApp在Devnet上运行，并可通过一个公开的Vercel URL进行访问和测试。
    *   **Commit Message:** `chore: 部署合约与应用至 Devnet`

*   **任务 4.3: 最终文档与演示**
    *   **操作:**
        1.  录制一个简短的项目功能演示视频。
        2.  更新 `finalProject/demo.md` 文件，填入项目Repo地址、Vercel演示地址和视频链接。
    *   **产出:** 项目完全准备就绪，可以提交。
    *   **Commit Message:** `docs: 更新最终文档并添加项目演示`
