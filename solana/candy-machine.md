# A Simplified Explanation of Solana's Candy Machine

[Candy Machine](https://github.com/metaplex-foundation/metaplex/tree/master/rust/nft-candy-machine) is a Solana program that is commonly used to mint NFT collections on the Solana blockchain. This post is about how the Candy Machine program works, not about how to use Candy Machine to mint a collection (there are already plenty of resources about that).

There are four main steps involved to set up a Candy Machine.

1. First, you create an account that contains configuration data (e.g. the symbol of your NFTs). This is done by calling the initialize_config instruction of the Candy Machine program.
2. Second, you add "config lines" to the config account. There's one line for each NFT in the collection, and each line contains the name of the asset and a URI pointing to its metadata.
2. Next, you create an account that contains the rest of the Candy Machine's info (e.g. the address of the wallet that minting proceeds should go to). This is done by calling the initialize_candy_machine instruction of the Candy Machine program. I'm actually not sure why the configuration account is different from the Candy Machine account-maybe it's to work around size limitations?
3. Finally, when someone wants to mint an NFT, they call the mint_nft instruction of the Candy Machine program. 

Let's go over each of these steps in more detail.

## 1. Creating the Config
`nft_candy_machine::initialize_config` initializes an account that contains the Candy Machine's configuration info. Note that the account must be created beforehand—this instruction just initializes the account's data.

Here's the function signature:

```rust
pub fn initialize_config(ctx: Context<InitializeConfig>, data: ConfigData) -> ProgramResult {
    ...
}
```

Here are the accounts that get passed in:


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
}
```

And here's what actually stored in the config account:

```rust
#[account]
#[derive(Default)]
pub struct Config {
    pub authority: Pubkey,
    pub data: ConfigData,
    // there's a borsh vec u32 denoting how many actual lines of data there are currently (eventually equals max number of lines)
    // There is actually lines and lines of data after this but we explicitly never want them deserialized.
    // here there is a borsh vec u32 indicating number of bytes in bitmask array.
    // here there is a number of bytes equal to ceil(max_number_of_lines/8) and it is a bit mask used to figure out when to increment borsh vec u32
}

#[derive(AnchorSerialize, AnchorDeserialize, Clone, Default)]
pub struct ConfigData {
    pub uuid: String,
    /// The symbol for the asset
    pub symbol: String,
    /// Royalty basis points that goes to creators in secondary sales (0-10000)
    pub seller_fee_basis_points: u16,
    pub creators: Vec<Creator>,
    pub max_supply: u64,
    pub is_mutable: bool,
    pub retain_authority: bool,
    pub max_number_of_lines: u32,
}
```

Here's what the instruction looks like on [Solana Explorer](https://explorer.solana.com/tx/2vZzQg8uHa6HoQVu1QDDmARUqAuYotCtiFDq7NTTwEvtj1FsR2mCjnVaX9SXp6RKVFoKyuS2LkzuKZN58tMtQGiL?cluster=devnet). Note that the instruction's transaction also includes an instruction that creates the account (with address `6eZ5ycD92Xw71kQFV1HQ5DV44KgeAAbA8dKnF25Xs73D`) and sets its owner to the Candy Machine program.

| Instruction # | Program | Instruction Name |
|:--|:--|:--|
| 1 | System Program | Create Account |
| 2 | Candy Machine Program | Initialize Config |

Finally, here's a diagram of all the accounts in play so far. It's quite simple for now, but gets complicated later on.

/candy-machine-initialize-config.png

## 2. Adding config lines
`nft_candy_machine::add_config_lines` adds lines to the config. There is one line per NFT in the Candy Machine. Each line contains the NFT's name and a URI pointing to its metadata.

Here's the function signature:

```rust
pub fn add_config_lines(
    ctx: Context<AddConfigLines>,
    index: u32,
    config_lines: Vec<ConfigLine>,
) -> ProgramResult {
    ...
}
```

Here are the accounts that get passed in:

```rust
#[derive(Accounts)]
pub struct AddConfigLines<'info> {
    #[account(mut, has_one = authority)]
    config: ProgramAccount<'info, Config>,
    #[account(signer)]
    authority: AccountInfo<'info>,
}
```

The config lines are stored in the config account, but since they're not meant to be deserialized, they don't appear in the Rust struct:

```rust
#[account]
#[derive(Default)]
pub struct Config {
    pub authority: Pubkey,
    pub data: ConfigData,
    // there's a borsh vec u32 denoting how many actual lines of data there are currently (eventually equals max number of lines)
    // There is actually lines and lines of data after this but we explicitly never want them deserialized.
    // here there is a borsh vec u32 indicating number of bytes in bitmask array.
    // here there is a number of bytes equal to ceil(max_number_of_lines/8) and it is a bit mask used to figure out when to increment borsh vec u32
}
```

Here's what this instruction looks like on [Solana Explorer](https://explorer.solana.com/tx/2mVeP6zirTHSVdrn3ixj1ApT5PSJ6ZPe2PkP7o8YTny3Tb3b4r22xJGVeTD5zPGMYpa5rWqyWCnHaJFAURvph6x7?cluster=devnet). 


| Instruction # | Program | Instruction Name |
|:--|:--|:--|
| 1 | Candy Machine Program | Add Config Lines |

If you copy/paste the instruction data into a [hex decoder](https://www.convertstring.com/EncodeDecode/HexDecode), you can see all the config lines. For example (from the Solana Explorer example):

```
ß2àãsj
Rectangle NFT #1?https://arweave.net/RmOJijL57LWTYMSbOa_xRv0ib77svoSIFQLKjKOSk4ERectangle NFT #2?https://arweave.net/D8qFyBT-POVF8wCz9H19z9goVduaa4TqedYp33wnEB8Rectangle NFT #3?https://arweave.net/n-BpOn70TvpxQNucT5DyGpwlNt-sT78uBGcOfXWtefcRectangle NFT #4?
```

At this point, nothing has changed with our account diagram. The only difference is that the config account has had some data added to it.

/candy-machine-initialize-config.png


## 3. Creating the Candy Machine
`nft_candy_machine::initialize_candy_machine` creates and initializes an account that contains more info about the Candy Machine. I'm not sure why this is separated from the config data, maybe to keep account sizes down?

Here's the function signature:

```rust
pub fn initialize_candy_machine(
    ctx: Context<InitializeCandyMachine>,
    bump: u8,
    data: CandyMachineData,
) -> ProgramResult {
    ...
}
```

Here are the accounts that get passed in:

```rust
#[derive(Accounts)]
#[instruction(bump: u8, data: CandyMachineData)]
pub struct InitializeCandyMachine<'info> {
    #[account(init, seeds=[PREFIX.as_bytes(), config.key().as_ref(), data.uuid.as_bytes()], payer=payer, bump=bump, space=8+32+32+33+32+64+64+64+200)]
    candy_machine: ProgramAccount<'info, CandyMachine>,
    #[account(constraint= wallet.owner == &spl_token::id() || (wallet.data_is_empty() && wallet.lamports() > 0) )]
    wallet: AccountInfo<'info>,
    #[account(has_one=authority)]
    config: ProgramAccount<'info, Config>,
    #[account(signer, constraint= authority.data_is_empty() && authority.lamports() > 0)]
    authority: AccountInfo<'info>,
    #[account(mut, signer)]
    payer: AccountInfo<'info>,
    #[account(address = system_program::ID)]
    system_program: AccountInfo<'info>,~~‌~~
}
```

And here's what's actually stored in the account:

```rust
#[account]
#[derive(Default)]
pub struct CandyMachine {
    pub authority: Pubkey,
    pub wallet: Pubkey,
    pub token_mint: Option<Pubkey>,
    pub config: Pubkey,
    pub data: CandyMachineData,
    pub items_redeemed: u64,
    pub bump: u8,
}

#[derive(AnchorSerialize, AnchorDeserialize, Clone, Default)]
pub struct CandyMachineData {
    pub uuid: String,
    pub price: u64,
    pub items_available: u64,
    pub go_live_date: Option<i64>,
}
```

Note that the account stores the config account's address (`Pubkey`).

