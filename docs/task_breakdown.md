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

## 阶段 1: 链上程序核心功能开发 (详细分步指南)

**目标:** 遵循最佳实践，为初级开发者提供一份从零开始、每步可验证的详细指南，完成 `mint_base_nft` 和 `combine_nfts` 两个核心指令的开发与测试。

### 子任务 1.1: 清理模板并重命名程序

**背景:** 官方模板自带一个名为 `vault` 的示例程序，我们需要将其重命名为 `artifex` 并移除无关代码，为项目打下干净的基础。

*   **子任务 1.1.1: 重命名程序目录和配置**
    *   **操作步骤:**
        1.  **重命名目录:** 将 `finalProject/anchor/programs/vault` 目录重命名为 `finalProject/anchor/programs/artifex`。
        2.  **更新 `Anchor.toml`:** 打开 `finalProject/anchor/Anchor.toml` 文件，找到 `[programs.localnet]` 部分，将其下的 `vault = "..."` 修改为 `artifex = "7EgQFbJSfXmk6QvArnGTHjQGaKotSWNs7o8DTZ1KVxEJ"` (这里的 Program ID 只是一个占位符，后续会被自动更新)。
        3.  **更新 `lib.rs`:** 打开 `finalProject/anchor/programs/artifex/src/lib.rs` 文件，将 `pub mod vault` 修改为 `pub mod artifex`。
        4.  **更新 `Cargo.toml`:** 打开 `finalProject/anchor/programs/artifex/Cargo.toml` 文件，将 `[package]` 下的 `name = "vault"` 修改为 `name = "artifex"`，并将 `[lib]` 下的 `name = "vault"` 修改为 `name = "artifex"`。
    *   **验证方法:** 在 `finalProject/anchor` 目录下运行 `anchor build`。命令应该能成功执行，并在 `target/idl/` 目录下生成 `artifex.json` 文件。
    *   **注意事项:** 确保以上四个位置全部修改，否则 `anchor build` 会因找不到程序或配置不匹配而失败。

### 子任务 1.2: 添加 Metaplex 依赖

**背景:** 要创建符合 Metaplex 标准的 NFT，我们必须在项目中引入官方的 `mpl-token-metadata` 库。

*   **子任务 1.2.1: 在 `Cargo.toml` 中添加依赖**
    *   **操作步骤:**
        1.  打开 `finalProject/anchor/programs/artifex/Cargo.toml` 文件。
        2.  在 `[dependencies]` 下添加一行: `mpl-token-metadata = { version = "4.1.2", features = ["no-entrypoint"] }`。
    *   **验证方法:** 在 `finalProject/anchor` 目录下运行 `anchor build`。
    *   **注意事项与常见问题:**
        *   **为什么需要 `features = ["no-entrypoint"]`?** 因为我们只是想通过跨程序调用（CPI）来“调用” Metaplex 程序，而不是要将它作为我们自己程序的一部分来编译。这个特性标志会移除它的入口点，避免编译冲突。
        *   **坑：依赖版本冲突。** 这是 Solana 开发中最常见的问题。你可能会遇到 `solana-sdk` 或其他库的版本不兼容的错误。
        *   **解决方案:** 如果遇到版本冲突，最可靠的方法是在 **工作区根目录** 的 `finalProject/anchor/Cargo.toml` 文件中添加 `[patch]` 来强制统一版本。在该文件末尾添加以下内容：
            ```toml
            [patch.crates-io]
            solana-program = { git = "https://github.com/solana-labs/solana", rev = "5db3839" }
            solana-instruction = { git = "https://github.com/solana-labs/solana", rev = "5db3839" }
            solana-zk-sdk = { git = "https://github.com/solana-labs/solana", rev = "5db3839" }
            ```
            这会告诉 Cargo 在整个工作区都使用指定版本的 Solana 核心库，从而解决冲突。添加后，再次运行 `anchor build`。

### 子任务 1.3: 定义 `mint_base_nft` 的账户上下文 (`Context`)

**背景:** 在实现指令逻辑之前，我们需要使用 Anchor 的 `#[derive(Accounts)]` 宏来精确定义该指令需要哪些账户，并设置约束。

*   **子任务 1.3.1: 编写 `MintBaseNft` 结构体**
    *   **操作步骤:**
        1.  打开 `finalProject/anchor/programs/artifex/src/lib.rs`。
        2.  **删除旧代码:** 删除模板自带的 `deposit`、`withdraw` 函数，以及 `VaultAction` 结构体和 `VaultError` 枚举。
        3.  **添加 `use` 语句:** 在文件顶部添加与 Token 和 Metadata 相关的 `use` 语句：
            ```rust
            use anchor_spl::{
                associated_token::AssociatedToken,
                metadata::Metadata,
                token::{Mint, Token},
            };
            ```
        4.  **定义结构体:** 添加 `MintBaseNft` 结构体，并使用 `#[account(...)]` 宏为每个账户添加约束。
            ```rust
            #[derive(Accounts)]
            pub struct MintBaseNft<'info> {
                // 用户，交易的发起者和付费者
                #[account(mut)]
                pub signer: Signer<'info>,

                // 新 NFT 的 Mint 账户，需要被初始化
                #[account(
                    init,
                    payer = signer,
                    mint::decimals = 0,
                    mint::authority = signer,
                    mint::freeze_authority = signer,
                )]
                pub mint: Account<'info, Mint>,

                // 用于接收 NFT 的关联代币账户 (ATA)
                /// CHECK: ATA an Payer are checked
                #[account(
                    init,
                    payer = signer,
                    associated_token::mint = mint,
                    associated_token::authority = signer,
                )]
                pub associated_token_account: UncheckedAccount<'info>,

                // Metaplex Metadata 账户地址
                /// CHECK: This is not dangerous because we don't read or write from this account
                #[account(
                    mut,
                    seeds = [b"metadata", metadata_program.key().as_ref(), mint.key().as_ref()],
                    bump,
                )]
                pub metadata_account: UncheckedAccount<'info>,

                // Metaplex Master Edition 账户地址
                /// CHECK: This is not dangerous because we don't read or write from this account
                #[account(
                    mut,
                    seeds = [b"metadata", metadata_program.key().as_ref(), mint.key().as_ref(), b"edition"],
                    bump,
                )]
                pub master_edition_account: UncheckedAccount<'info>,

                // 必需的系统程序
                pub token_program: Program<'info, Token>,
                pub associated_token_program: Program<'info, AssociatedToken>,
                pub system_program: Program<'info, System>,
                pub rent: Sysvar<'info, Rent>,
                pub metadata_program: Program<'info, Metadata>,
            }
            ```
        5.  **添加占位函数:** 在 `#[program]` 模块中添加一个空的 `mint_base_nft` 函数。
            ```rust
            pub fn mint_base_nft(ctx: Context<MintBaseNft>) -> Result<()> {
                Ok(())
            }
            ```
    *   **验证方法:** 在 `finalProject/anchor` 目录下运行 `anchor build`。如果成功，说明你的账户定义和约束是正确的。
    *   **注意事项:**
        *   `/// CHECK:` 注释是必需的，用于告诉 Anchor 你已经手动确认过某些非 `Account` 类型账户的安全性。
        *   PDA 账户的 `seeds` 必须与 Metaplex 标准完全一致，否则交易会失败。

### 子任务 1.4: 实现 `mint_base_nft` 的核心逻辑

**背景:** 账户结构定义好后，我们将在函数体内编写 CPI 调用，真正地创建 Metaplex NFT。

*   **子任务 1.4.1: 编写 CPI 调用**
    *   **操作步骤:**
        1.  打开 `finalProject/anchor/programs/artifex/src/lib.rs`。
        2.  在 `mint_base_nft` 函数中，添加创建 Metadata 和 Master Edition 的 CPI 调用。
            ```rust
            use mpl_token_metadata::pda::{find_master_edition_account, find_metadata_account};
            use mpl_token_metadata::state::{Collection, Creator, DataV2};
            // ... in mint_base_nft function
            
            // 1. 设置账户上下文
            let cpi_context = CpiContext::new(
                ctx.accounts.token_program.to_account_info(),
                anchor_spl::token::MintTo {
                    mint: ctx.accounts.mint.to_account_info(),
                    to: ctx.accounts.associated_token_account.to_account_info(),
                    authority: ctx.accounts.signer.to_account_info(),
                },
            );

            // 2. Mint 1个 Token 到 ATA
            anchor_spl::token::mint_to(cpi_context, 1)?;

            // 3. 创建 Metadata Account
            let cpi_context_metadata = CpiContext::new(
                ctx.accounts.metadata_program.to_account_info(),
                anchor_spl::metadata::CreateMetadataAccountsV3 {
                    metadata: ctx.accounts.metadata_account.to_account_info(),
                    mint: ctx.accounts.mint.to_account_info(),
                    mint_authority: ctx.accounts.signer.to_account_info(),
                    payer: ctx.accounts.signer.to_account_info(),
                    update_authority: ctx.accounts.signer.to_account_info(),
                    system_program: ctx.accounts.system_program.to_account_info(),
                    rent: ctx.accounts.rent.to_account_info(),
                },
            );
            
            let data_v2 = DataV2 {
                name: "Artifex NFT".to_string(), // 示例名称
                symbol: "ARTFX".to_string(),    // 示例符号
                uri: "http://example.com/1.json".to_string(), // 示例 URI
                seller_fee_basis_points: 500, // 5%
                creators: Some(vec![Creator {
                    address: ctx.accounts.signer.key(),
                    verified: true,
                    share: 100,
                }]),
                collection: None,
                uses: None,
            };

            anchor_spl::metadata::create_metadata_accounts_v3(cpi_context_metadata, data_v2, false, true, None)?;

            // 4. 创建 Master Edition Account
            let cpi_context_master_edition = CpiContext::new(
                ctx.accounts.metadata_program.to_account_info(),
                anchor_spl::metadata::CreateMasterEditionV3 {
                    edition: ctx.accounts.master_edition_account.to_account_info(),
                    mint: ctx.accounts.mint.to_account_info(),
                    update_authority: ctx.accounts.signer.to_account_info(),
                    mint_authority: ctx.accounts.signer.to_account_info(),
                    payer: ctx.accounts.signer.to_account_info(),
                    metadata: ctx.accounts.metadata_account.to_account_info(),
                    token_program: ctx.accounts.token_program.to_account_info(),
                    system_program: ctx.accounts.system_program.to_account_info(),
                    rent: ctx.accounts.rent.to_account_info(),
                },
            );

            anchor_spl::metadata::create_master_edition_v3(cpi_context_master_edition, Some(0))?;

            Ok(())
            ```
    *   **验证方法:** 再次运行 `anchor build`。确保所有逻辑和账户引用都正确无误。
    *   **注意事项:**
        *   CPI 的账户结构 `CreateMetadataAccountsV3` 和 `CreateMasterEditionV3` 必须与 `anchor-spl` 库中的定义完全一致。
        *   `DataV2` 结构体中的 `name`, `symbol`, `uri` 等字段将在前端调用时作为参数传入，这里暂时使用硬编码的占位符。

### 子任务 1.5: 编写并运行 `mint_base_nft` 的测试

**背景:** 链上逻辑完成后，我们需要在模拟环境中验证它是否能按预期工作。

*   **子任务 1.5.1: 编写 TypeScript 测试脚本**
    *   **操作步骤:**
        1.  打开 `finalProject/anchor/tests/artifex.ts` (如果不存在，可以复制 `vault.ts` 并重命名)。
        2.  **清理旧测试:** 删除所有与 `vault` 相关的测试代码。
        3.  **编写新测试:** 添加一个 `it('Should mint a base NFT!', async () => { ... });` 测试用例。
        4.  **实现测试逻辑:**
            ```typescript
            import * as anchor from "@coral-xyz/anchor";
            import { Program } from "@coral-xyz/anchor";
            import { Artifex } from "../target/types/artifex";
            import { PublicKey } from "@solana/web3.js";

            describe("artifex", () => {
              const provider = anchor.AnchorProvider.env();
              anchor.setProvider(provider);
              const program = anchor.workspace.Artifex as Program<Artifex>;
              const signer = provider.wallet as anchor.Wallet;

              it("Should mint a base NFT!", async () => {
                const mint = anchor.web3.Keypair.generate();

                // 派生 PDA 地址
                const [metadataAddress] = PublicKey.findProgramAddressSync(
                  [
                    Buffer.from("metadata"),
                    new PublicKey("metaqbxxUerdq28cj1RbAWkYQm3ybzjb6a8bt518x1s").toBuffer(),
                    mint.publicKey.toBuffer(),
                  ],
                  new PublicKey("metaqbxxUerdq28cj1RbAWkYQm3ybzjb6a8bt518x1s")
                );
                const [masterEditionAddress] = PublicKey.findProgramAddressSync(
                  [
                    Buffer.from("metadata"),
                    new PublicKey("metaqbxxUerdq28cj1RbAWkYQm3ybzjb6a8bt518x1s").toBuffer(),
                    mint.publicKey.toBuffer(),
                    Buffer.from("edition"),
                  ],
                  new PublicKey("metaqbxxUerdq28cj1RbAWkYQm3ybzjb6a8bt518x1s")
                );

                // 调用链上方法
                await program.methods
                  .mintBaseNft()
                  .accounts({
                    mint: mint.publicKey,
                    signer: signer.publicKey,
                    metadataAccount: metadataAddress,
                    masterEditionAccount: masterEditionAddress,
                    // 其他账户 Anchor 会自动推断
                  })
                  .signers([mint])
                  .rpc();
                
                console.log("Mint successful!");
              });
            });
            ```
    *   **验证方法:** 在 `finalProject/anchor` 目录下运行 `anchor test`。如果看到 "Mint successful!" 并且没有错误，说明测试通过。
    *   **注意事项:**
        *   **PDA 地址派生:** 在客户端派生 PDA 地址的逻辑必须和链上 `seeds` 完全一致，包括 `metadata_program` 的地址 (`metaqbxxUerdq28cj1RbAWkYQm3ybzjb6a8bt518x1s`)。这是最常见的错误来源。
        *   **Signers:** 因为 `mint` 是一个新生成的 `Keypair`，它需要作为签名者加入到交易中。

---

*（后续 `combine_nfts` 的详细步骤将遵循类似模式：定义Context -> 实现逻辑 -> 编写测试。这部分可以在完成 `mint_base_nft` 后继续添加。）*
