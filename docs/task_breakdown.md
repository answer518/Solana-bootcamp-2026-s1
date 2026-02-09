# Artifex 项目任务拆分计划 (Monorepo 版本)

本文档为“Artifex”项目提供了一个详细的、分步执行的开发计划。此版本基于 `https://github.com/solana-foundation/templates/tree/main/community/phantom-embedded-react` 的单体仓库（Monorepo）最佳实践进行设计，确保开发流程的高效、自动化和可维护性。

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
    *   **验证方法:** 在 `finalProject/anchor` 目录下运行 `anchor build`。命令应该能成功执行，并在 `target/idl/` 目录下生成 `artifex.json` 文件。

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

---

### 子任务 1.6: 定义 `combine_nfts` 的账户上下文 (`Context`)

**背景:** 现在我们来实现核心的合成功能。此指令将接收两个基础 NFT，销毁它们，并创建一个新的、代表“合成品”的 NFT。首先，我们需要定义这个复杂操作所需的所有账户。

*   **子任务 1.6.1: 编写 `CombineNfts` 结构体**
    *   **操作步骤:**
        1.  打开 `finalProject/anchor/programs/artifex/src/lib.rs`。
        2.  在 `MintBaseNft` 结构体下方，添加 `CombineNfts` 结构体。为了清晰起见，我们将明确定义销毁两个基础 NFT 并铸造一个新 NFT 所需的所有账户。
            ```rust
            #[derive(Accounts)]
            pub struct CombineNfts<'info> {
                // 用户，交易的发起者和付费者
                #[account(mut)]
                pub signer: Signer<'info>,

                // --- 要销毁的第一个基础 NFT ---
                #[account(mut)]
                pub base_nft_1_mint: Account<'info, Mint>,
                #[account(
                    mut,
                    associated_token::mint = base_nft_1_mint,
                    associated_token::authority = signer,
                )]
                pub base_nft_1_ata: Account<'info, TokenAccount>,
                /// CHECK: Checked by metaplex program
                #[account(mut)]
                pub base_nft_1_metadata: UncheckedAccount<'info>,
                /// CHECK: Checked by metaplex program
                #[account(mut)]
                pub base_nft_1_edition: UncheckedAccount<'info>,

                // --- 要销毁的第二个基础 NFT ---
                #[account(mut)]
                pub base_nft_2_mint: Account<'info, Mint>,
                #[account(
                    mut,
                    associated_token::mint = base_nft_2_mint,
                    associated_token::authority = signer,
                )]
                pub base_nft_2_ata: Account<'info, TokenAccount>,
                /// CHECK: Checked by metaplex program
                #[account(mut)]
                pub base_nft_2_metadata: UncheckedAccount<'info>,
                /// CHECK: Checked by metaplex program
                #[account(mut)]
                pub base_nft_2_edition: UncheckedAccount<'info>,

                // --- 新创建的合成 NFT ---
                #[account(
                    init,
                    payer = signer,
                    mint::decimals = 0,
                    mint::authority = signer,
                    mint::freeze_authority = signer,
                )]
                pub combined_nft_mint: Account<'info, Mint>,
                #[account(
                    init,
                    payer = signer,
                    associated_token::mint = combined_nft_mint,
                    associated_token::authority = signer,
                )]
                pub combined_nft_ata: Account<'info, TokenAccount>,
                /// CHECK: Checked by metaplex program
                #[account(
                    mut,
                    seeds = [b"metadata", metadata_program.key().as_ref(), combined_nft_mint.key().as_ref()],
                    bump,
                )]
                pub combined_nft_metadata: UncheckedAccount<'info>,
                /// CHECK: Checked by metaplex program
                #[account(
                    mut,
                    seeds = [b"metadata", metadata_program.key().as_ref(), combined_nft_mint.key().as_ref(), b"edition"],
                    bump,
                )]
                pub combined_nft_edition: UncheckedAccount<'info>,

                // --- 系统程序 ---
                pub token_program: Program<'info, Token>,
                pub associated_token_program: Program<'info, AssociatedToken>,
                pub system_program: Program<'info, System>,
                pub rent: Sysvar<'info, Rent>,
                pub metadata_program: Program<'info, Metadata>,
            }
            ```
        3.  **添加占位函数:** 在 `#[program]` 模块中添加一个空的 `combine_nfts` 函数。
            ```rust
            pub fn combine_nfts(ctx: Context<CombineNfts>) -> Result<()> {
                Ok(())
            }
            ```
    *   **验证方法:** 在 `finalProject/anchor` 目录下运行 `anchor build`。如果成功，说明账户定义是正确的。
    *   **注意事项:**
        *   这个结构体非常大，因为它需要处理三个 NFT（两个旧的，一个新的）的所有相关账户。
        *   对于要销毁的 NFT，我们需要 `mut` 权限，因为我们将要修改它们（销毁 Token，关闭账户）。
        *   `TokenAccount` 类型比 `UncheckedAccount` 更安全，因为它会自动反序列化并检查所有权。我们在这里用于 ATA。

### 子任务 1.7: 实现 `combine_nfts` 的核心逻辑

**背景:** 账户结构定义完毕，现在我们来编写销毁和铸造的逻辑。我们将使用 Metaplex 的 `burn_nft` 和我们之前创建的铸造逻辑。

*   **子任务 1.7.1: 编写销毁与铸造的 CPI 调用**
    *   **操作步骤:**
        1.  打开 `finalProject/anchor/programs/artifex/src/lib.rs`。
        2.  在 `combine_nfts` 函数中，添加销毁两个基础 NFT、然后铸造一个新的合成 NFT 的逻辑。
            ```rust
            // ... in combine_nfts function

            // 1. 销毁第一个 NFT
            msg!("Burning Base NFT #1");
            let cpi_burn_accounts_1 = Burn {
                mint: ctx.accounts.base_nft_1_mint.to_account_info(),
                from: ctx.accounts.base_nft_1_ata.to_account_info(),
                authority: ctx.accounts.signer.to_account_info(),
            };
            let cpi_burn_context_1 = CpiContext::new(ctx.accounts.token_program.to_account_info(), cpi_burn_accounts_1);
            anchor_spl::token::burn(cpi_burn_context_1, 1)?;

            // (可选但推荐) 关闭第一个 NFT 的 ATA 账户，将租金返还给用户
            let cpi_close_accounts_1 = CloseAccount {
                account: ctx.accounts.base_nft_1_ata.to_account_info(),
                destination: ctx.accounts.signer.to_account_info(),
                authority: ctx.accounts.signer.to_account_info(),
            };
            let cpi_close_context_1 = CpiContext::new(ctx.accounts.token_program.to_account_info(), cpi_close_accounts_1);
            anchor_spl::token::close_account(cpi_close_context_1)?;

            // 2. 销毁第二个 NFT
            msg!("Burning Base NFT #2");
            let cpi_burn_accounts_2 = Burn {
                mint: ctx.accounts.base_nft_2_mint.to_account_info(),
                from: ctx.accounts.base_nft_2_ata.to_account_info(),
                authority: ctx.accounts.signer.to_account_info(),
            };
            let cpi_burn_context_2 = CpiContext::new(ctx.accounts.token_program.to_account_info(), cpi_burn_accounts_2);
            anchor_spl::token::burn(cpi_burn_context_2, 1)?;

            // (可选但推荐) 关闭第二个 NFT 的 ATA 账户
            let cpi_close_accounts_2 = CloseAccount {
                account: ctx.accounts.base_nft_2_ata.to_account_info(),
                destination: ctx.accounts.signer.to_account_info(),
                authority: ctx.accounts.signer.to_account_info(),
            };
            let cpi_close_context_2 = CpiContext::new(ctx.accounts.token_program.to_account_info(), cpi_close_accounts_2);
            anchor_spl::token::close_account(cpi_close_context_2)?;

            // 3. 铸造新的合成 NFT (逻辑与 mint_base_nft 类似)
            msg!("Minting Combined NFT");
            // 3.1 Mint 1个 Token 到新的 ATA
            let cpi_mint_context = CpiContext::new(
                ctx.accounts.token_program.to_account_info(),
                anchor_spl::token::MintTo {
                    mint: ctx.accounts.combined_nft_mint.to_account_info(),
                    to: ctx.accounts.combined_nft_ata.to_account_info(),
                    authority: ctx.accounts.signer.to_account_info(),
                },
            );
            anchor_spl::token::mint_to(cpi_mint_context, 1)?;

            // 3.2 创建 Metadata Account
            let cpi_metadata_context = CpiContext::new(
                ctx.accounts.metadata_program.to_account_info(),
                anchor_spl::metadata::CreateMetadataAccountsV3 {
                    metadata: ctx.accounts.combined_nft_metadata.to_account_info(),
                    mint: ctx.accounts.combined_nft_mint.to_account_info(),
                    mint_authority: ctx.accounts.signer.to_account_info(),
                    payer: ctx.accounts.signer.to_account_info(),
                    update_authority: ctx.accounts.signer.to_account_info(),
                    system_program: ctx.accounts.system_program.to_account_info(),
                    rent: ctx.accounts.rent.to_account_info(),
                },
            );
            let data_v2 = DataV2 {
                name: "Combined Artifex NFT".to_string(),
                symbol: "C-ARTFX".to_string(),
                uri: "http://example.com/combined.json".to_string(),
                seller_fee_basis_points: 500,
                creators: Some(vec![Creator {
                    address: ctx.accounts.signer.key(),
                    verified: true,
                    share: 100,
                }]),
                collection: None,
                uses: None,
            };
            anchor_spl::metadata::create_metadata_accounts_v3(cpi_metadata_context, data_v2, false, true, None)?;

            // 3.3 创建 Master Edition Account
            let cpi_master_edition_context = CpiContext::new(
                ctx.accounts.metadata_program.to_account_info(),
                anchor_spl::metadata::CreateMasterEditionV3 {
                    edition: ctx.accounts.combined_nft_edition.to_account_info(),
                    mint: ctx.accounts.combined_nft_mint.to_account_info(),
                    update_authority: ctx.accounts.signer.to_account_info(),
                    mint_authority: ctx.accounts.signer.to_account_info(),
                    payer: ctx.accounts.signer.to_account_info(),
                    metadata: ctx.accounts.combined_nft_metadata.to_account_info(),
                    token_program: ctx.accounts.token_program.to_account_info(),
                    system_program: ctx.accounts.system_program.to_account_info(),
                    rent: ctx.accounts.rent.to_account_info(),
                },
            );
            anchor_spl::metadata::create_master_edition_v3(cpi_master_edition_context, Some(0))?;

            msg!("Combine operation successful!");
            Ok(())
            ```
    *   **验证方法:** 再次运行 `anchor build`。确保所有逻辑都已正确添加。
    *   **注意事项:**
        *   你需要从 `anchor_spl::token` 导入 `Burn` 和 `CloseAccount` 结构体。
        *   销毁 NFT 实际上是销毁其代币（Token）。一旦代币数量变为 0，与之关联的 ATA 就可以安全地关闭，以回收租金。
        *   Metaplex 的 `burn_nft` 指令更为复杂，它会同时处理 Token、Metadata 和 Edition 账户。但为了教学目的，手动 `burn` 和 `close_account` 更能清晰地展示底层原理。

### 子任务 1.8: 编写并运行 `combine_nfts` 的测试

**背景:** 这是最关键的验证步骤。我们需要编写一个测试，模拟用户的完整流程：铸造两个基础 NFT，然后调用合成指令，并验证结果。

*   **子任务 1.8.1: 编写 TypeScript 测试脚本**
    *   **操作步骤:**
        1.  打开 `finalProject/anchor/tests/artifex.ts`。
        2.  在 `describe` 块内，添加一个新的 `it('Should combine two NFTs!', async () => { ... });` 测试用例。
        3.  **实现测试逻辑:**
            ```typescript
            import { getAssociatedTokenAddress } from "@solana/spl-token";
            // ... other imports

            it("Should combine two NFTs!", async () => {
                // --- Helper function to mint a base NFT ---
                const mintBaseNft = async () => {
                    const mint = anchor.web3.Keypair.generate();
                    const associatedTokenAccount = await getAssociatedTokenAddress(mint.publicKey, signer.publicKey);
                    const [metadataAddress] = PublicKey.findProgramAddressSync(
                        [Buffer.from("metadata"), new PublicKey("metaqbxxUerdq28cj1RbAWkYQm3ybzjb6a8bt518x1s").toBuffer(), mint.publicKey.toBuffer()],
                        new PublicKey("metaqbxxUerdq28cj1RbAWkYQm3ybzjb6a8bt518x1s")
                    );
                    const [masterEditionAddress] = PublicKey.findProgramAddressSync(
                        [Buffer.from("metadata"), new PublicKey("metaqbxxUerdq28cj1RbAWkYQm3ybzjb6a8bt518x1s").toBuffer(), mint.publicKey.toBuffer(), Buffer.from("edition")],
                        new PublicKey("metaqbxxUerdq28cj1RbAWkYQm3ybzjb6a8bt518x1s")
                    );

                    await program.methods
                        .mintBaseNft()
                        .accounts({
                            mint: mint.publicKey,
                            signer: signer.publicKey,
                            associatedTokenAccount: associatedTokenAccount,
                            metadataAccount: metadataAddress,
                            masterEditionAccount: masterEditionAddress,
                        })
                        .signers([mint])
                        .rpc();
                    
                    return { mint, associatedTokenAccount, metadataAddress, masterEditionAddress };
                };

                // 1. 铸造两个基础 NFT
                console.log("Minting two base NFTs...");
                const nft1 = await mintBaseNft();
                const nft2 = await mintBaseNft();
                console.log("Base NFTs minted.");

                // 2. 准备合成 NFT 的账户
                const combinedNftMint = anchor.web3.Keypair.generate();
                const combinedNftAta = await getAssociatedTokenAddress(combinedNftMint.publicKey, signer.publicKey);
                const [combinedNftMetadata] = PublicKey.findProgramAddressSync(
                    [Buffer.from("metadata"), new PublicKey("metaqbxxUerdq28cj1RbAWkYQm3ybzjb6a8bt518x1s").toBuffer(), combinedNftMint.publicKey.toBuffer()],
                    new PublicKey("metaqbxxUerdq28cj1RbAWkYQm3ybzjb6a8bt518x1s")
                );
                const [combinedNftEdition] = PublicKey.findProgramAddressSync(
                    [Buffer.from("metadata"), new PublicKey("metaqbxxUerdq28cj1RbAWkYQm3ybzjb6a8bt518x1s").toBuffer(), combinedNftMint.publicKey.toBuffer(), Buffer.from("edition")],
                    new PublicKey("metaqbxxUerdq28cj1RbAWkYQm3ybzjb6a8bt518x1s")
                );

                // 3. 调用合成指令
                console.log("Combining NFTs...");
                await program.methods
                    .combineNfts()
                    .accounts({
                        signer: signer.publicKey,
                        baseNft1Mint: nft1.mint.publicKey,
                        baseNft1Ata: nft1.associatedTokenAccount,
                        baseNft1Metadata: nft1.metadataAddress,
                        baseNft1Edition: nft1.masterEditionAddress,
                        baseNft2Mint: nft2.mint.publicKey,
                        baseNft2Ata: nft2.associatedTokenAccount,
                        baseNft2Metadata: nft2.metadataAddress,
                        baseNft2Edition: nft2.masterEditionAddress,
                        combinedNftMint: combinedNftMint.publicKey,
                        combinedNftAta: combinedNftAta,
                        combinedNftMetadata: combinedNftMetadata,
                        combinedNftEdition: combinedNftEdition,
                        // 其他程序地址 Anchor 会自动推断
                    })
                    .signers([combinedNftMint])
                    .rpc();
                
                console.log("Combine successful!");

                // 4. (可选) 验证旧账户是否已被关闭
                const nft1AtaInfo = await provider.connection.getAccountInfo(nft1.associatedTokenAccount);
                const nft2AtaInfo = await provider.connection.getAccountInfo(nft2.associatedTokenAccount);
                
                if (nft1AtaInfo === null && nft2AtaInfo === null) {
                    console.log("Old ATAs successfully closed.");
                } else {
                    console.error("Old ATAs were not closed.");
                }
            });
            ```
    *   **验证方法:** 在 `finalProject/anchor` 目录下运行 `anchor test`。你应该会看到所有测试（包括之前的 `mint_base_nft` 测试）都成功通过，并打印出 "Combine successful!" 和 "Old ATAs successfully closed."。
    *   **注意事项:**
        *   为了让测试更简洁，我们创建了一个 `mintBaseNft` 辅助函数。
        *   在调用 `combineNfts` 时，`combinedNftMint` 是新生成的 `Keypair`，因此必须作为签名者传入。
        *   测试的最后一步通过 `getAccountInfo` 检查旧的 ATA 是否为 `null` 来验证它们是否已被正确关闭。这是一个很好的实践，可以确保我们的程序没有留下“僵尸”账户。

---

## 阶段 2: 前端集成与开发

**目标:** 将链上程序的功能暴露给用户，构建一个允许用户铸造和合成 NFT 的 Web 界面。

*（此阶段的详细步骤将在链上程序完全通过测试后继续补充。）*
