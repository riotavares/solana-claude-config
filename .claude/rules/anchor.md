---
paths:
  - "programs/**/src/**/*.rs"
---

## Anchor/Solana Program Rules

### Account Validation
- **ALWAYS** validate account ownership
- **ALWAYS** check signer status for privileged operations
- **ALWAYS** verify PDA derivation with canonical bump
- Use constraints in account struct

```rust
#[derive(Accounts)]
#[instruction(amount: u64)]  // amount is passed from instruction handler
pub struct Transfer<'info> {
    #[account(
        mut,
        has_one = authority,
        constraint = source.amount >= amount @ ErrorCode::InsufficientFunds
    )]
    pub source: Account<'info, TokenAccount>,

    #[account(mut)]
    pub destination: Account<'info, TokenAccount>,

    pub authority: Signer<'info>,

    #[account(
        seeds = [b"vault", source.key().as_ref()],
        bump = vault.bump,
    )]
    pub vault: Account<'info, Vault>,
}

// Instruction handler that uses this accounts struct
pub fn transfer(ctx: Context<Transfer>, amount: u64) -> Result<()> {
    // amount is available here and in the constraint above
    // ...
    Ok(())
}
```

### Arithmetic Safety
- **ALWAYS** use checked arithmetic
- **NEVER** use unchecked operations without verification
- Handle overflow/underflow explicitly

```rust
// Good
let total = amount_a
    .checked_add(amount_b)
    .ok_or(ErrorCode::Overflow)?;

// Bad
let total = amount_a + amount_b;  // can panic!
```

### Error Handling
- **NEVER** use `unwrap()` or `expect()` in program code (OK in tests)
- Define custom error codes
- Use descriptive error messages

```rust
#[error_code]
pub enum ErrorCode {
    #[msg("Arithmetic overflow occurred")]
    Overflow,
    #[msg("Insufficient funds for operation")]
    InsufficientFunds,
    #[msg("Unauthorized signer")]
    Unauthorized,
}
```

### PDA Management
- Store canonical bump in account data
- **NEVER** recalculate bump on every transaction
- Use stored bump for PDA derivation

```rust
#[account]
pub struct Vault {
    pub authority: Pubkey,
    pub bump: u8,  // Store this!
}

// Initialize with canonical bump
let (vault_pda, bump) = Pubkey::find_program_address(
    &[b"vault", authority.key().as_ref()],
    ctx.program_id
);

// Use stored bump later
let vault_seeds = &[
    b"vault",
    authority.key().as_ref(),
    &[vault.bump],  // Use stored bump
];
```

### CPI (Cross-Program Invocation)
- Validate target program ID
- Don't forward signer privileges blindly
- Reload accounts after CPI if they're modified

```rust
// Validate program
if cpi_program.key() != expected_program_id {
    return Err(ErrorCode::InvalidProgram.into());
}

// Make CPI
token::transfer(cpi_ctx, amount)?;

// Reload if needed
ctx.accounts.token_account.reload()?;
```

### Account Closing
- Zero account data
- Set closed discriminator
- Use `close` constraint in Anchor

```rust
#[account(mut, close = destination)]
pub account_to_close: Account<'info, MyAccount>,
```

### Testing Requirements

**Testing Frameworks:**
- **[Mollusk](https://github.com/buffalojoec/mollusk)** - Lightweight SVM harness for unit testing individual instructions without spinning up a full validator
- **[LiteSVM](https://github.com/LiteSVM/litesvm)** - Fast, in-process Solana VM for integration testing with full transaction simulation
- **[Trident](https://ackee.xyz/trident/docs/latest/)** - Fuzzing framework for Anchor programs that generates random inputs to find edge cases

**Testing Checklist:**
- Test ALL error paths
- Verify account validation failures
- Test arithmetic edge cases (overflow, underflow, zero)
- Fuzz test critical operations

### CU (Compute Unit) Optimization
- Minimize logging in production
- Use stored bumps (don't use find_program_address on-chain)
- Use zero-copy when possible
- Profile with `sol_log_compute_units!()` from `solana_program`

```rust
#[cfg(feature = "debug")]
msg!("Debug message");  // Only in debug builds
```

### Security Checklist
- [ ] All accounts validated (owner, signer, PDA)
- [ ] Checked arithmetic used
- [ ] No unwrap() in program code
- [ ] Error codes defined
- [ ] CPI target programs validated
- [ ] Accounts reloaded after CPI
- [ ] Account closing done safely
- [ ] PDA bumps stored and reused

### Deployment
- **NEVER** deploy to mainnet without explicit confirmation
- Test on devnet first
- Verify program with `solana program show`
- Document upgrade authority
- Consider using Squads for multisig upgrades
