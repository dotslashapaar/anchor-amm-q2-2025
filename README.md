# Anchor AMM - Q4 2024

This project implements an Automated Market Maker (AMM) on the Solana blockchain using the Anchor framework. It's designed to facilitate liquidity pools and token swaps with high efficiency and security.

---

## Overview

This AMM allows users to:

- Create and manage liquidity pools for different token pairs.
- Swap tokens with minimal slippage, and slippage protection.

## Let's walk through the architecture:

For this program, we will have one state account:
- A Config account


### A Config account consists of:
```rust
#[account]
pub struct Config {
    pub seed: u64,
    pub authority: Option<Pubkey>,
    pub mint_x: Pubkey,
    pub mint_y: Pubkey,
    pub fee: u16,
    pub locked: bool,
    pub config_bump: u8,
    pub lp_bump: u8,
}
```
### In this state account, we will store:

-    seed: A unique identifier to differentiate between various pool configurations.
-    authority: An optional admin key that can lock the pool.
-    mint_x: The first token in the pool.
-    mint_y: The second token in the pool.
-    fee: The fee applied to each swap in the pool, in basis points.
-    locked: Whether the pool is locked.
-    config_bump: The bump for creating the Config account PDA.
-    lp_bump: The bump for the Liquidity Provider PDA.


### The pool creator will be able to create a Config for a liquidity pool. For that, we create the Initialize context
```rust

#[derive(Accounts)]
#[instruction(seed: u64)]
pub struct Initialize <'info> {
    #[account(mut)]
    pub initializer: Signer<'info>,
    pub mint_x: Account<'info, Mint>,
    pub mint_y: Account<'info, Mint>,
    #[account(
        init,
        payer = initializer,
        seeds = [b"lp", config.key.as_ref()],
        bump,
        mint::decimals = 6,
        mint::authority = config,
    )]
    pub mint_lp: Account<'info, Mint>,
    #[account(
        init,
        payer = initializer,
        associated_token::mint = mint_x,
        associated_token::authority = config,
    )]
    pub vault_x: Account<'info, TokenAccount>,
    #[account(
        init,
        payer = initializer,
        associated_token::mint = mint_y,
        associated_token::authority = config,
    )]
    pub vault_y: Account<'info, TokenAccount>,
    #[account(
        init,
        payer = initializer,
        seeds = [b"config", seed.to_le_bytes().as_ref()],
        bump,
        space = Config::INIT_SPACE,
    )]
    pub config: Account<'info, Config>,
    pub token_program: Program<'info, Token>,
    pub associated_token_program: Program<'info, AssociatedToken>,
    pub system_program: Program<'info, System>,
```

Let´s have a closer look at the accounts that we are passing in this context:

- initializer: The user who signs and pays for the initialization of new Config. This must be mutable as lamports will be deducted.
- mint_x, mint_y: The mint accounts for the two tokens that will be part of the liquidity pool.
- mint_lp: A newly created mint for the liquidity provider (LP) tokens. It's initialized with 6 decimals and its authority is set to the config account, ensuring the pool controls minting.
- vault_x, vault_y: These are token accounts associated with mint_x and mint_y respectively, where tokens will be stored. They are owned by the authority account, acting as vaults for the pool's assets.
- config: This is the Config account for the pool, initialized with a seed for uniqueness across different pools. It holds all the pool's configuration data like fees, token mints, and bumps for PDAs.
- token_program: to initialize vault accounts

### We implement simple initialize functionality for the Initialize context:

```rust
impl<'info> Initialize<'info> {
    pub fn init(&mut self, seed: u64, fee: u16, authority: Option<Pubkey>, bumps: InitializeBumps) -> Result<()> {
        self.config.set_inner(Config {
            seed,
            authority,
            mint_x: self.mint_x.key(),
            mint_y: self.mint_y.key(),
            fee,
            locked: false,
            config_bump: bumps.config,
            lp_bump: bumps.mint_lp,
        });

        Ok(())
    }
}
```

We basically just set the initial data of our LP Config account.

---

### Users will be able to deposit tokens into the liquidity pool:
```rust
#[derive(Accounts)]
pub struct Deposit<'info> {
    #[account(mut)]
    pub user: Signer<'info>,
    pub mint_x: Account<'info, Mint>,
    pub mint_y: Account<'info, Mint>,
    #[account(
        has_one = mint_x,
        has_one = mint_y,
        seeds = [b"config", config.seed.to_le_bytes().as_ref()],
        bump = config.config_bump,
    )]
    pub config: Account<'info, Config>,
    #[account(
        mut,
        seeds = [b"lp", config.key().as_ref()],
        bump = config.lp_bump,
    )]
    pub mint_lp: Account<'info, Mint>,
    #[account(
        mut,
        associated_token::mint = mint_x,
        associated_token::authority = config,
    )]
    pub vault_x: Account<'info, TokenAccount>,
    #[account(
        mut,
        associated_token::mint = mint_y,
        associated_token::authority = config,
    )]
    pub vault_y: Account<'info, TokenAccount>,
    #[account(
        mut,
        associated_token::mint = mint_x,
        associated_token::authority = user,
    )]
    pub user_x: Account<'info, TokenAccount>,
    #[account(
        mut,
        associated_token::mint = mint_y,
        associated_token::authority = user,
    )]
    pub user_y: Account<'info, TokenAccount>,
    #[account(
        init_if_needed,
        payer = user,
        associated_token::mint = mint_lp,
        associated_token::authority = user,
    )]
    pub user_lp: Account<'info, TokenAccount>,
    pub token_program: Program<'info, Token>,
    pub system_program: Program<'info, System>,
    pub associated_token_program: Program<'info, AssociatedToken>,
}
```
Accounts in the Deposit Context:

- user: The user who signs the transaction to deposit tokens into the pool. This account is mutable because it has to pay for account creation.
- mint_x, mint_y: Mint accounts for the tokens being deposited. These ensure the correct tokens are being used in the pool.
- config: The configuration account for the pool, checked to ensure it's the right pool for the tokens being deposited. The has_one constraint verifies that mint_x and mint_y match the pool's setup.
- mint_lp: The mint for LP (Liquidity Provider) tokens, which are minted when users deposit into the pool. This account is mutable as new LP tokens will be minted.
- vault_x, vault_y: These are the pool's vaults where deposited tokens are stored. They are mutable because tokens will be added to these accounts. 
- user_x, user_y: The user's token accounts from which tokens will be transferred to the pool's vaults. These accounts are mutable due to the token transfer.
- user_lp: The user's LP token account. If it doesn't exist, it's initialized here. This account is mutable because new LP tokens are deposited here.
- token_program, system_program, associated_token_program: Necessary programs for token transfers, account creation, and managing associated token accounts.

### We then implement the deposit functionality for the Deposit context:
```rust
impl<'info> Deposit<'info> {
    pub fn deposit (
        &mut self,
        amount: u64, // Amount of LP tokens that the user wants to "claim"
        max_x: u64, // Maximum amount of token X that the user is willing to deposit
        max_y: u64, // Maximum amount of token Y that the user is willing to deposit
    ) -> Result<()> {
        require!(self.config.locked == false, AmmError::PoolLocked);
        require!(amount != 0, AmmError::InvalidAmount);

        let (x, y) = match self.mint_lp.supply == 0 && self.vault_x.amount == 0 && self.vault_y.amount == 0 {
            true => (max_x, max_y),
            false => {
                let amounts = ConstantProduct::xy_deposit_amounts_from_l(
                    self.vault_x.amount, 
                    self.vault_y.amount, 
                    self.mint_lp.supply, 
                    amount, 
                6
            ).unwrap();
            (amounts.x, amounts.y)
            }
        };

        require!(x <= max_x && y <= max_y, AmmError::SlippageExceeded);

        // deposit token x
        self.deposit_tokens(true, x)?;
        // deposit token y
        self.deposit_tokens(false, y)?;
        // mint lp tokens
        self.mint_lp_tokens(amount)
    }

    pub fn deposit_tokens(&self, is_x: bool, amount: u64) -> Result<()> {
        let (from, to) = match is_x {
            true => (self.user_x.to_account_info(), self.vault_x.to_account_info()),
            false => (self.user_y.to_account_info(), self.vault_y.to_account_info()),
        };

        let cpi_program = self.token_program.to_account_info();

        let cpi_accounts = Transfer {
            from,
            to,
            authority: self.user.to_account_info(),
        };

        let ctx = CpiContext::new(cpi_program, cpi_accounts);

        transfer(ctx, amount)
    }

    pub fn mint_lp_tokens(&self, amount: u64) -> Result<()> {
        let cpi_program = self.token_program.to_account_info();

        let cpi_accounts = MintTo {
            mint: self.mint_lp.to_account_info(),
            to: self.user_lp.to_account_info(),
            authority: self.config.to_account_info(),
        };

        let seeds = &[
            &b"config"[..],
            &self.config.seed.to_le_bytes(),
            &[self.config.config_bump],
        ];

        let signer_seeds = &[&seeds[..]];

        let ctx = CpiContext::new_with_signer(cpi_program, cpi_accounts, signer_seeds);

        mint_to(ctx, amount)
    }
}
```

In this implementation we start by calculating the tokens to be deposited into the pool. In case of an empty pool, we use the arguments from the user. If the pool already contains tokens, we use the ConstantProduct::xy_deposit_amounts_from_l() function from an external crate to calculate the proportinate amount of tokens to deposit into the pool, while we control the slippage to protect the user.

---

### Users can also withdraw tokens from the liquidity pool:
Accounts in the Withdraw Context:

- user: The user initiating the withdrawal.
- mint_x, mint_y: Mint accounts for the tokens in the pool, ensuring the withdrawal involves the correct tokens.
- config: The configuration account for the pool, validated to match the pool from which tokens are being withdrawn. The has_one constraints check if mint_x and mint_y are correct.
- mint_lp: The mint for LP tokens, mutable as supply has to be adjusted during withdraval.
- vault_x, vault_y: Pool vaults from which tokens will be withdrawn, mutable for token transfer.
- user_x, user_y: User's token accounts where withdrawn tokens will be sent. Initialized if they don't exist yet, mutable for receiving tokens.
- user_lp: The user's LP token account from which LP tokens will be burned. Mutable as tokens are removed from here during withdrawal.
- token_program, system_program, associated_token_program: Programs required for token operations, account management, and token burning.


### We then implement the withdraw functionality for the Withdraw context:

```rust
pub fn withdraw(
    &mut self,
    amount: u64, // Amount of LP tokens that the user wants to "burn"
    min_x: u64,  // Minimum amount of token X that the user wants to receive
    min_y: u64,  // Minimum amount of token Y that the user wants to receive
) -> Result<()> {
    require!(self.config.locked == false, AmmError::PoolLocked);
    require!(amount != 0, AmmError::InvalidAmount);
    require!(min_x != 0 || min_y != 0, AmmError::InvalidAmount);

    let amounts = ConstantProduct::xy_withdraw_amounts_from_l(
        self.vault_x.amount, 
        self.vault_y.amount, 
        self.mint_lp.supply, 
        amount, 
    6,
    )
    .map_err(AmmError::from)?;

    require!(min_x <= amounts.x && min_y <= amounts.y, AmmError::SlippageExceeded);

    self.withdraw_tokens(true, amounts.x)?;
    self.withdraw_tokens(false, amounts.y)?;
    self.burn_lp_tokens(amount)?;
    
    Ok(())
}

pub fn withdraw_tokens(&self, is_x: bool, amount: u64) -> Result<()> {
    let (from, to) = match is_x {
        true => (self.vault_x.to_account_info(), self.user_x.to_account_info()),
        false => (self.vault_y.to_account_info(), self.user_y.to_account_info()),
    };

    let cpi_program = self.token_program.to_account_info();

    let cpi_accounts = Transfer {
        from,
        to,
        authority: self.config.to_account_info(),
    };

    let seeds = &[
        &b"config"[..],
        &self.config.seed.to_le_bytes(),
        &[self.config.config_bump],
    ];
    let signer_seeds = &[&seeds[..]];

    let cpi_ctx = CpiContext::new_with_signer(cpi_program, cpi_accounts, signer_seeds);

    transfer(cpi_ctx, amount)?;
    
    Ok(())
}

pub fn burn_lp_tokens(&self, amount: u64) -> Result<()> {
    let cpi_program = self.token_program.to_account_info();

    let cpi_accounts = Burn {
        mint: self.mint_lp.to_account_info(),
        from: self.user_lp.to_account_info(),
        authority: self.user.to_account_info(),
    };

    let cpi_context = CpiContext::new(cpi_program, cpi_accounts);

    burn(cpi_context, amount)?;

    Ok(())
}
```

withdraw: Burns LP tokens and transfers corresponding tokens back to the user, checking for pool lock status and slippage protection.
withdraw_tokens: Transfers tokens from pool vaults to user.
burn_lp_tokens: Burns LP tokens from the user's account, reducing LP token supply.


### Users will be able to swap tokens using the liquidity pool

```rust
#[derive(Accounts)]
pub struct Swap<'info> {
    #[account(mut)]
    pub user: Signer<'info>,
    pub mint_x: Account<'info, Mint>,
    pub mint_y: Account<'info, Mint>,
    #[account(
        init_if_needed,
        payer = user,
        associated_token::mint = mint_x,
        associated_token::authority = user,
    )]
    pub user_x: Account<'info, TokenAccount>,
    #[account(
        init_if_needed,
        payer = user,
        associated_token::mint = mint_y,
        associated_token::authority = user,
    )]
    pub user_y: Account<'info, TokenAccount>,
    #[account(
        mut,
        associated_token::mint = mint_x,
        associated_token::authority = config,
    )]
    pub vault_x: Account<'info, TokenAccount>,
    #[account(
        mut,
        associated_token::mint = mint_y,
        associated_token::authority = config,
    )]
    pub vault_y: Account<'info, TokenAccount>,
    #[account(
        has_one = mint_x,
        has_one = mint_y,
        seeds = [b"config", config.seed.to_le_bytes().as_ref()],
        bump = config.config_bump,
    )]
    pub config: Account<'info, Config>,
    pub token_program: Program<'info, Token>,
    pub associated_token_program: Program<'info, AssociatedToken>,
    pub system_program: Program<'info, System>,
}
```
This context setup ensures that the swap operation can be securely executed, maintaining the integrity of both the user's assets and the pool's reserves.

- user: The user performing the swap, mutable because this account will pay for any necessary account initialization and token transfers.
- mint_x, mint_y: Mint accounts for the tokens involved in the swap, ensuring the correct tokens are used in the transaction.
- user_x, user_y: User's token accounts for tokens X and Y respectively. These accounts are initialized if needed, using the user as payer. They hold the tokens before and after the swap.
- vault_x, vault_y: The pool's vault accounts where tokens X and Y are stored. These accounts are mutable as tokens will be moved in or out during the swap.
- config: The configuration account for the liquidity pool, checked to ensure it matches the tokens being swapped. The has_one constraints verify that mint_x and mint_y match the pool's setup.
- token_program: The Solana Token program, necessary for handling token transfers during the swap.
- associated_token_program: Used to initialize the user's token accounts if they do not exist, ensuring proper token account management.
- system_program: Required for creating or initializing accounts on Solana, including any new associated token accounts.

### We then implement some functionality for this context:
```rust
pub fn swap(&mut self, is_x: bool, amount: u64, min: u64) -> Result<()> {
    require!(self.config.locked == false, AmmError::PoolLocked);
    require!(amount > 0, AmmError::InvalidAmount);

    let mut curve = ConstantProduct::init(
        self.vault_x.amount,
            self.vault_y.amount, 
            self.vault_x.amount, 
            self.config.fee,
        None,
    )
    .map_err(AmmError::from)?;

    let p = match is_x {
        true => LiquidityPair::X,
        false => LiquidityPair::Y,
    };

    let res = curve.swap(p, amount, min).map_err(AmmError::from)?;

    require!(res.deposit != 0, AmmError::InvalidAmount);
    require!(res.withdraw != 0, AmmError::InvalidAmount);

    // deposit tokens
    self.deposit_tokens(is_x, res.deposit)?;
    // withdraw tokens
    self.withdraw_tokens(is_x, res.withdraw)?;
    // transfer fee

    Ok(())
}

pub fn deposit_tokens(&mut self, is_x: bool, amount: u64) -> Result<()> {
    let (from, to) = match is_x {
        true => (self.user_x.to_account_info() , self.vault_x.to_account_info()),
        false => (self.user_y.to_account_info(), self.vault_y.to_account_info()),
    };

    let cpi_program = self.token_program.to_account_info();

    let accounts = Transfer {
        from: from.to_account_info(),
        to: to.to_account_info(),
        authority: self.user.to_account_info(),
    };

    let cpi_ctx = CpiContext::new(cpi_program, accounts);

    transfer(cpi_ctx, amount)?;

    Ok(())
}

pub fn withdraw_tokens(&mut self, is_x: bool, amount: u64) -> Result<()> {
    let (from, to) = match is_x {
        true => (self.vault_y.to_account_info() , self.user_y.to_account_info()),
        false => (self.vault_x.to_account_info(), self.user_x.to_account_info()),
    };

    let cpi_program = self.token_program.to_account_info();

    let accounts = Transfer {
        from: from.to_account_info(),
        to: to.to_account_info(),
        authority: self.config.to_account_info(),
    };

    let seeds = &[
        &b"config"[..],
        &self.config.seed.to_le_bytes(),
        &[self.config.config_bump],
    ];
    let signer_seeds = &[&seeds[..]];

    let cpi_ctx = CpiContext::new_with_signer(cpi_program, accounts, signer_seeds);

    transfer(cpi_ctx, amount)?;

    Ok(())
}
```

Here we check if the pool is locked, walidation if the swap amount is not invalid, and initializing a ConstantProduct curve with the current state of the pool to calculate the amount of tokens the user should receive from the swap.
Then we deposit the tokens from the user based on token type, and return the other token the user wanted to receive.
