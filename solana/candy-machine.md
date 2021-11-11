# A Simplified Explanation of Solana's Candy Machine

[Candy Machine](https://github.com/metaplex-foundation/metaplex/tree/master/rust/nft-candy-machine) is a Solana program that is commonly used to mint NFT collections on the Solana blockchain. This post is about how the Candy Machine program works, not about how to use Candy Machine to mint a collection (there are already plenty of resources about that).

There are three main steps involved to set up a Candy Machine.

1. First, you must create an account that contains configuration data (e.g. the symbol of your NFTs). This is done by calling the initialize_config instruction of the Candy Machine program.
2. Next, you must create an account that contains the rest of the Candy Machine's info (e.g. the address of the wallet that minting proceeds should go to). This is done by calling the initialize_candy_machine instruction of the Candy Machine program. I'm actually not sure why the configuration account is different from the Candy Machine account-maybe it's to work around size limitations?
3. Finally, when someone wants to mint an NFT, they call the mint_nft instruction of the Candy Machine program. 

Let's go over each of these steps in more detail.

## 1. Creating the Config

```rust
#[derive(Accounts)]
#[instruction(data: ConfigData)]
pub struct InitializeConfig<'info> {
    #[account(mut, constraint= config.to_account_info().owner == program_id && config.to_account_info().data_len() >= CONFIG_ARRAY_START+4+(data.max_number_of_lines as usize)*CONFIG_LINE_SIZE + 4 + (data.max_number_of_lines.checked_div(8).ok_or(ErrorCode::NumericalOverflowError)? as usize))]
    config: AccountInfo<'info>,
    #[account(constraint= authority.data_is_empty() && authority.lamports() > 0 )]
    authority: AccountInfo<'info>,
    #[account(mut, signer)]
    payer: AccountInfo<'info>,
    // rent: Sysvar<'info, Rent>,
}
```