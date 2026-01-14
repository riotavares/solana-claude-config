# Programs with Pinocchio

Pinocchio is a minimalist Rust library for crafting Solana programs without the heavyweight `solana-program` crate. It delivers significant performance gains through zero-copy techniques and minimal dependencies.

## When to Use Pinocchio

Use Pinocchio when you need:
- **Maximum compute efficiency**: 84% CU savings compared to Anchor
- **Minimal binary size**: Leaner code paths and smaller deployments
- **Zero external dependencies**: Only Solana SDK types required
- **Fine-grained control**: Direct memory access and byte-level operations
- **no_std environments**: Embedded or constrained contexts

## When to Use Anchor Instead

- Team is learning Solana
- Development speed > performance
- Program complexity is high
- IDL generation required
- Maintenance burden is a concern

## Core Architecture

### Entrypoint Pattern

```rust
use pinocchio::{entrypoint, program_error::ProgramError, pubkey::Pubkey, ProgramResult};
use pinocchio::account_info::AccountInfo;

entrypoint!(process_instruction);

fn process_instruction(
    _program_id: &Pubkey,
    accounts: &[AccountInfo],
    instruction_data: &[u8],
) -> ProgramResult {
    match instruction_data.split_first() {
        Some((0, data)) => Deposit::try_from((data, accounts))?.process(),
        Some((1, _)) => Withdraw::try_from(accounts)?.process(),
        _ => Err(ProgramError::InvalidInstructionData)
    }
}
```

Single-byte discriminators support 255 instructions; use two bytes for up to 65,535 variants.

### Lazy Entrypoint (Maximum Performance)

```rust
use pinocchio::lazy_program_entrypoint;

lazy_program_entrypoint!(process_instruction);
```

### Instruction Structure

Separate validation from business logic using the `TryFrom` trait:

```rust
pub struct Deposit<'a> {
    pub accounts: DepositAccounts<'a>,
    pub data: DepositData,
}

impl<'a> TryFrom<(&'a [u8], &'a [AccountInfo])> for Deposit<'a> {
    type Error = ProgramError;

    fn try_from((data, accounts): (&'a [u8], &'a [AccountInfo])) -> Result<Self, Self::Error> {
        let accounts = DepositAccounts::try_from(accounts)?;
        let data = DepositData::try_from(data)?;
        Ok(Self { accounts, data })
    }
}

impl<'a> Deposit<'a> {
    pub const DISCRIMINATOR: &'a u8 = &0;

    pub fn process(&self) -> ProgramResult {
        // Business logic only - validation already complete
        Ok(())
    }
}
```

## Account Validation

Pinocchio requires manual validation. Wrap all checks in `TryFrom` implementations:

### Account Struct Validation

```rust
pub struct DepositAccounts<'a> {
    pub owner: &'a AccountInfo,
    pub vault: &'a AccountInfo,
    pub system_program: &'a AccountInfo,
}

impl<'a> TryFrom<&'a [AccountInfo]> for DepositAccounts<'a> {
    type Error = ProgramError;

    fn try_from(accounts: &'a [AccountInfo]) -> Result<Self, Self::Error> {
        let [owner, vault, system_program, _remaining @ ..] = accounts else {
            return Err(ProgramError::NotEnoughAccountKeys);
        };

        // Signer check
        if !owner.is_signer() {
            return Err(ProgramError::MissingRequiredSignature);
        }

        // Owner check
        if !vault.is_owned_by(&pinocchio_system::ID) {
            return Err(ProgramError::InvalidAccountOwner);
        }

        // Program ID check (prevents arbitrary CPI)
        if system_program.key() != &pinocchio_system::ID {
            return Err(ProgramError::IncorrectProgramId);
        }

        Ok(Self { owner, vault, system_program })
    }
}
```

### Instruction Data Validation

```rust
pub struct DepositData {
    pub amount: u64,
}

impl<'a> TryFrom<&'a [u8]> for DepositData {
    type Error = ProgramError;

    fn try_from(data: &'a [u8]) -> Result<Self, Self::Error> {
        if data.len() != core::mem::size_of::<u64>() {
            return Err(ProgramError::InvalidInstructionData);
        }

        let amount = u64::from_le_bytes(data.try_into().unwrap());

        if amount == 0 {
            return Err(ProgramError::InvalidInstructionData);
        }

        Ok(Self { amount })
    }
}
```

## Zero-Copy Account Access

### Struct Field Ordering

Order fields from largest to smallest alignment to minimize padding:

```rust
// Good: 16 bytes total
#[repr(C)]
struct GoodOrder {
    big: u64,     // 8 bytes, 8-byte aligned
    medium: u16,  // 2 bytes, 2-byte aligned
    small: u8,    // 1 byte, 1-byte aligned
    // 5 bytes padding
}

// Bad: 24 bytes due to padding
#[repr(C)]
struct BadOrder {
    small: u8,    // 1 byte
    // 7 bytes padding
    big: u64,     // 8 bytes
    medium: u16,  // 2 bytes
    // 6 bytes padding
}
```

### Zero-Copy Reading (Safe Pattern)

Use byte arrays with accessor methods to avoid alignment issues:

```rust
#[repr(C)]
pub struct Config {
    pub authority: Pubkey,
    pub mint: Pubkey,
    seed: [u8; 8],   // Store as bytes
    fee: [u8; 2],    // Store as bytes
    pub state: u8,
    pub bump: u8,
}

impl Config {
    pub const LEN: usize = core::mem::size_of::<Self>();

    pub fn from_bytes(data: &[u8]) -> Result<&Self, ProgramError> {
        if data.len() != Self::LEN {
            return Err(ProgramError::InvalidAccountData);
        }
        // Safe: all fields are byte-aligned
        Ok(unsafe { &*(data.as_ptr() as *const Self) })
    }

    pub fn seed(&self) -> u64 {
        u64::from_le_bytes(self.seed)
    }

    pub fn fee(&self) -> u16 {
        u16::from_le_bytes(self.fee)
    }

    pub fn set_seed(&mut self, seed: u64) {
        self.seed = seed.to_le_bytes();
    }

    pub fn set_fee(&mut self, fee: u16) {
        self.fee = fee.to_le_bytes();
    }
}
```

### Dangerous Patterns to Avoid

```rust
// ❌ transmute with unaligned data
let value: u64 = unsafe { core::mem::transmute(bytes_slice) };

// ❌ Pointer casting to packed structs
#[repr(C, packed)]
pub struct Packed { pub a: u8, pub b: u64 }
let config = unsafe { &*(data.as_ptr() as *const Packed) };

// ❌ Direct field access on packed structs creates unaligned references
let b_ref = &packed.b;

// ❌ Assuming alignment without verification
let config = unsafe { &*(data.as_ptr() as *const Config) };
```

## PDA Validation (Manual)

### Store and use canonical bumps

```rust
use pinocchio::pubkey::Pubkey;

// GOOD - Manual PDA validation with stored bump using create_program_address
fn validate_pda(
    account: &AccountInfo,
    seeds: &[&[u8]],
    stored_bump: u8,
    program_id: &Pubkey,
) -> ProgramResult {
    // Construct full seeds including bump
    let bump_slice = &[stored_bump];
    let mut full_seeds: Vec<&[u8]> = seeds.to_vec();
    full_seeds.push(bump_slice);

    // Create PDA from known seeds + stored bump (no search needed!)
    let derived_address = Pubkey::create_program_address(&full_seeds, program_id)
        .map_err(|_| ProgramError::InvalidSeeds)?;

    // Validate
    if account.key() != &derived_address {
        return Err(ProgramError::InvalidSeeds);
    }

    Ok(())
}
```

## Cross-Program Invocations (CPIs)

### Basic CPI

```rust
use pinocchio_system::instructions::Transfer;

Transfer {
    from: self.accounts.owner,
    to: self.accounts.vault,
    lamports: self.data.amount,
}.invoke()?;
```

### PDA-Signed CPI

```rust
use pinocchio::{seeds::Seed, signer::Signer};

let seeds = [
    Seed::from(b"vault"),
    Seed::from(self.accounts.owner.key().as_ref()),
    Seed::from(&[bump]),
];
let signers = [Signer::from(&seeds)];

Transfer {
    from: self.accounts.vault,
    to: self.accounts.owner,
    lamports: self.accounts.vault.lamports(),
}.invoke_signed(&signers)?;
```

## Token Account Helpers

### SPL Token Validation

```rust
pub struct Mint;

impl Mint {
    pub fn check(account: &AccountInfo) -> Result<(), ProgramError> {
        if !account.is_owned_by(&pinocchio_token::ID) {
            return Err(ProgramError::InvalidAccountOwner);
        }
        if account.data_len() != pinocchio_token::state::Mint::LEN {
            return Err(ProgramError::InvalidAccountData);
        }
        Ok(())
    }
}
```

### Token2022 Support

Token2022 requires discriminator-based validation due to variable account sizes with extensions:

```rust
pub const TOKEN_2022_PROGRAM_ID: [u8; 32] = [...];
const TOKEN_2022_ACCOUNT_DISCRIMINATOR_OFFSET: usize = 165;
pub const TOKEN_2022_MINT_DISCRIMINATOR: u8 = 0x01;

pub struct Mint2022;

impl Mint2022 {
    pub fn check(account: &AccountInfo) -> Result<(), ProgramError> {
        if !account.is_owned_by(&TOKEN_2022_PROGRAM_ID) {
            return Err(ProgramError::InvalidAccountOwner);
        }

        let data = account.try_borrow_data()?;

        if data.len() != pinocchio_token::state::Mint::LEN {
            if data.len() <= TOKEN_2022_ACCOUNT_DISCRIMINATOR_OFFSET {
                return Err(ProgramError::InvalidAccountData);
            }
            if data[TOKEN_2022_ACCOUNT_DISCRIMINATOR_OFFSET] != TOKEN_2022_MINT_DISCRIMINATOR {
                return Err(ProgramError::InvalidAccountData);
            }
        }
        Ok(())
    }
}
```

## Error Handling

Use `thiserror` for descriptive errors (supports `no_std`):

```rust
use thiserror::Error;
use num_derive::FromPrimitive;
use pinocchio::program_error::ProgramError;

#[derive(Clone, Debug, Eq, Error, FromPrimitive, PartialEq)]
pub enum VaultError {
    #[error("Lamport balance below rent-exempt threshold")]
    NotRentExempt,
    #[error("Invalid account owner")]
    InvalidOwner,
    #[error("Account not initialized")]
    NotInitialized,
}

impl From<VaultError> for ProgramError {
    fn from(e: VaultError) -> Self {
        ProgramError::Custom(e as u32)
    }
}
```

## Closing Accounts Securely

Prevent revival attacks by marking closed accounts:

```rust
pub fn close(account: &AccountInfo, destination: &AccountInfo) -> ProgramResult {
    // Mark as closed (prevents reinitialization)
    {
        let mut data = account.try_borrow_mut_data()?;
        data[0] = 0xff;
    }

    // Transfer lamports
    *destination.try_borrow_mut_lamports()? += *account.try_borrow_lamports()?;

    // Shrink and close
    account.realloc(1, true)?;
    account.close()
}
```

## Performance Optimization

### Feature Flags for Logging

```toml
[features]
default = ["perf"]
perf = []
```

```rust
#[cfg(not(feature = "perf"))]
pinocchio::msg!("Instruction: Deposit");
```

### Binary Size Optimization

```toml
# Cargo.toml
[profile.release]
overflow-checks = true  # Keep for security
lto = "fat"            # Link-time optimization
codegen-units = 1      # Single codegen unit
opt-level = "z"        # Optimize for size
strip = true           # Strip symbols
```

### Minimize Allocations

```rust
// BAD - allocates Vec
let seeds = vec![b"vault".as_slice(), authority.as_ref()];

// GOOD - stack-allocated array
let seeds: &[&[u8]] = &[b"vault", authority.as_ref()];
```

### Zero-Allocation Architecture

```rust
// Good: references with borrowed lifetimes
pub struct Instruction<'a> {
    pub accounts: &'a [AccountInfo],
    pub data: &'a [u8],
}

// Enforce no heap usage
no_allocator!();
```

Respect Solana's memory limits: 4KB stack per function, 32KB total heap.

## Batch Instructions

Process multiple operations in a single CPI (saves ~1000 CU per batched operation):

```rust
const IX_HEADER_SIZE: usize = 2; // account_count + data_length

pub fn process_batch(mut accounts: &[AccountInfo], mut data: &[u8]) -> ProgramResult {
    loop {
        if data.len() < IX_HEADER_SIZE {
            return Err(ProgramError::InvalidInstructionData);
        }

        let account_count = data[0] as usize;
        let data_len = data[1] as usize;
        let data_offset = IX_HEADER_SIZE + data_len;

        if accounts.len() < account_count || data.len() < data_offset {
            return Err(ProgramError::InvalidInstructionData);
        }

        let (ix_accounts, ix_data) = (&accounts[..account_count], &data[IX_HEADER_SIZE..data_offset]);

        process_inner_instruction(ix_accounts, ix_data)?;

        if data_offset == data.len() {
            break;
        }

        accounts = &accounts[account_count..];
        data = &data[data_offset..];
    }

    Ok(())
}
```

## Testing with Mollusk

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use mollusk_svm::Mollusk;
    use solana_sdk::{account::Account, instruction::Instruction, pubkey::Pubkey};

    #[test]
    fn test_deposit() {
        let program_id = Pubkey::new_unique();
        let mollusk = Mollusk::new(&program_id, "target/deploy/vault");

        let authority = Pubkey::new_unique();
        let (vault_pda, bump) = Pubkey::find_program_address(
            &[b"vault", authority.as_ref()],
            &program_id,
        );

        // Build instruction
        let mut ix_data = vec![0u8]; // deposit discriminator
        ix_data.extend_from_slice(&1000u64.to_le_bytes());

        let instruction = Instruction {
            program_id,
            accounts: vec![/* ... */],
            data: ix_data,
        };

        let result = mollusk.process_instruction(&instruction, &accounts);

        assert!(result.program_result.is_ok());

        // Verify CU usage - Pinocchio should be very efficient
        println!("CU consumed: {}", result.compute_units_consumed);
        assert!(result.compute_units_consumed < 5000);
    }
}
```

## CU Benchmarking

Use Mollusk's compute unit bencher for tracking performance:

```rust
use mollusk_svm::MolluskComputeUnitBencher;

let bencher = MolluskComputeUnitBencher::new(mollusk)
    .must_pass(true)
    .out_dir("../target/benches");

bencher.bench(
    "deposit_instruction",
    &instruction,
    &accounts,
);
// Generates markdown report with CU usage and deltas
```

## Security Checklist

- [ ] Validate all account owners in `TryFrom` implementations
- [ ] Check signer status for authority accounts
- [ ] Verify PDA derivation matches expected seeds
- [ ] Validate program IDs before CPIs (prevent arbitrary CPI)
- [ ] Use checked math (`checked_add`, `checked_sub`, etc.)
- [ ] Mark closed accounts to prevent revival attacks
- [ ] Validate instruction data length before parsing
- [ ] Check for duplicate mutable accounts when accepting multiple of same type

## Integration with Codama

### Generate IDL and clients separately

```bash
# Pinocchio doesn't auto-generate IDL
# Use Shank for IDL generation
cargo install shank-cli

# Generate IDL
shank idl --out-dir idl programs/your-program/src/lib.rs

# Use Codama for client generation
npx @codama/cli generate
```

## Best Practices Summary

1. **Start with Anchor, optimize with Pinocchio if needed**
2. **Always benchmark CU savings** - measure before/after
3. **Document all unsafe code** - explain safety invariants
4. **Comprehensive testing** - more critical without Anchor safety nets
5. **Use TryFrom pattern** - maintains ergonomics with safety
6. **Store canonical bumps** - don't recalculate PDAs
7. **Single-byte discriminators** - saves space and CU
8. **Feature-gate logs** - minimize production overhead

---

**Sources:**
- [Pinocchio GitHub](https://github.com/anza-xyz/pinocchio)
- [Pinocchio Guide](https://www.helius.dev/blog/pinocchio)
- [Pinocchio Optimization Patterns](https://www.quicknode.com/guides/solana-development/pinocchio/how-to-build-and-deploy-a-solana-program-using-pinocchio)
