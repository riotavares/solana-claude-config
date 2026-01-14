---
paths:
  - "programs/**/src/**/*.rs"
when_filename_contains:
  - "pinocchio"
---

# Pinocchio Zero-Copy Framework Rules

These rules apply when using Pinocchio for maximum CU optimization.

## Overview

Pinocchio achieves **80-95% CU reduction** vs Anchor through:
- Zero-copy account access
- No external dependencies
- Minimal binary size
- Manual validation patterns

**Trade-off**: Less ergonomic than Anchor, but maximum performance.

## Entrypoint Patterns

### Standard Entrypoint
```rust
use pinocchio::{
    account_info::AccountInfo,
    entrypoint,
    program_error::ProgramError,
    pubkey::Pubkey,
    ProgramResult,
};

entrypoint!(process_instruction);

pub fn process_instruction(
    program_id: &Pubkey,
    accounts: &[AccountInfo],
    data: &[u8],
) -> ProgramResult {
    // Dispatch based on instruction discriminator
    match data.first().ok_or(ProgramError::InvalidInstructionData)? {
        0 => initialize(program_id, accounts, &data[1..]),
        1 => transfer(program_id, accounts, &data[1..]),
        _ => Err(ProgramError::InvalidInstructionData),
    }
}
```

### Lazy Entrypoint (Maximum Performance)
```rust
use pinocchio::lazy_program_entrypoint;

lazy_program_entrypoint!(process_instruction);

// Same signature as above
```

## Zero-Copy Account Access

### Reading Account Data (Zero-Copy)
```rust
use pinocchio::account_info::AccountInfo;

// BAD - Anchor way (copies data)
// let vault: Account<Vault> = Account::try_from(&account)?;

// GOOD - Pinocchio zero-copy (with validation)
fn read_vault_checked(account: &AccountInfo) -> Result<&Vault, ProgramError> {
    let data = account.borrow_data();

    // Check minimum length
    if data.len() < 1 + std::mem::size_of::<Vault>() {
        return Err(ProgramError::InvalidAccountData);
    }

    // Check discriminator
    if data[0] != VAULT_DISCRIMINATOR {
        return Err(ProgramError::InvalidAccountData);
    }

    // Zero-copy cast
    // SAFETY: We've verified length and discriminator above
    Ok(unsafe {
        &*((data.as_ptr().add(1)) as *const Vault)
    })
}
```

### Writing Account Data (Zero-Copy)
```rust
fn write_vault(account: &AccountInfo) -> Result<&mut Vault, ProgramError> {
    let mut data = account.borrow_mut_data();

    // Check minimum length
    if data.len() < 1 + std::mem::size_of::<Vault>() {
        return Err(ProgramError::InvalidAccountData);
    }

    // Set discriminator (single byte for efficiency)
    data[0] = VAULT_DISCRIMINATOR;

    // Zero-copy mutable reference
    // SAFETY: We've verified length above
    Ok(unsafe {
        &mut *((data.as_mut_ptr().add(1)) as *mut Vault)
    })
}
```

## Single-Byte Discriminators

### Use u8 discriminators (not 8-byte like Anchor)
```rust
// GOOD - Single byte (Pinocchio pattern)
const DISCRIMINATOR_VAULT: u8 = 0;
const DISCRIMINATOR_USER: u8 = 1;
const DISCRIMINATOR_POOL: u8 = 2;

// Account layout:
// [discriminator: 1 byte][data: remaining bytes]

#[repr(C)]
pub struct Vault {
    // No discriminator field in struct!
    pub authority: Pubkey,
    pub bump: u8,
    pub balance: u64,
}
```

## Manual Account Validation (TryFrom Pattern)

### Implement TryFrom for Anchor-style ergonomics
```rust
use std::convert::TryFrom;

pub struct ValidatedVault<'a> {
    pub info: &'a AccountInfo,
    pub data: &'a Vault,
}

impl<'a> TryFrom<&'a AccountInfo> for ValidatedVault<'a> {
    type Error = ProgramError;

    fn try_from(info: &'a AccountInfo) -> Result<Self, Self::Error> {
        // 1. Owner check
        if info.owner() != &crate::ID {
            return Err(ProgramError::IncorrectProgramId);
        }

        // 2. Data length check
        let expected_len = 1 + std::mem::size_of::<Vault>();
        if info.borrow_data().len() != expected_len {
            return Err(ProgramError::InvalidAccountData);
        }

        // 3. Discriminator check
        let data = info.borrow_data();
        if data[0] != DISCRIMINATOR_VAULT {
            return Err(ProgramError::InvalidAccountData);
        }

        // 4. Zero-copy data access
        // SAFETY: Length and discriminator verified above
        let vault_data = unsafe {
            &*((data.as_ptr().add(1)) as *const Vault)
        };

        Ok(ValidatedVault {
            info,
            data: vault_data,
        })
    }
}

// Usage (Anchor-like ergonomics!)
fn initialize(program_id: &Pubkey, accounts: &[AccountInfo], data: &[u8]) -> ProgramResult {
    let vault = ValidatedVault::try_from(&accounts[0])?;
    let authority = &accounts[1];

    // Validate authority is signer
    if !authority.is_signer() {
        return Err(ProgramError::MissingRequiredSignature);
    }

    // Process...
    Ok(())
}
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

// Usage
fn example_usage(
    vault_account: &AccountInfo,
    authority: &AccountInfo,
    vault_bump: u8,
    program_id: &Pubkey,
) -> ProgramResult {
    let seeds: &[&[u8]] = &[b"vault", authority.key().as_ref()];
    validate_pda(vault_account, seeds, vault_bump, program_id)?;
    Ok(())
}
```

## Account Struct Patterns

### Use #[repr(C)] for consistent layout
```rust
#[repr(C)]
#[derive(Copy, Clone)]
pub struct Vault {
    pub authority: Pubkey,    // 32 bytes
    pub bump: u8,             // 1 byte
    pub balance: u64,         // 8 bytes
    pub last_update: i64,     // 8 bytes
    // Total: 49 bytes + 1 byte discriminator = 50 bytes
}

impl Vault {
    pub const SIZE: usize = std::mem::size_of::<Self>();
}
```

### Implement helper methods
```rust
impl Vault {
    /// Calculate required account size
    pub const fn account_size() -> usize {
        1 + Self::SIZE  // discriminator + struct
    }

    /// Initialize a new vault
    pub fn init(authority: Pubkey, bump: u8) -> Self {
        Self {
            authority,
            bump,
            balance: 0,
            last_update: 0,
        }
    }
}
```

## Instruction Data Parsing

### Manual deserialization (no Borsh)
```rust
// Instruction format: [discriminator: 1 byte][data: remaining]

// GOOD - Manual parsing for performance
fn parse_deposit_instruction(data: &[u8]) -> Result<u64, ProgramError> {
    if data.len() != 8 {
        return Err(ProgramError::InvalidInstructionData);
    }

    // Parse u64 amount (little-endian)
    Ok(u64::from_le_bytes([
        data[0], data[1], data[2], data[3],
        data[4], data[5], data[6], data[7],
    ]))
}

// Or use array slicing
fn parse_deposit_fast(data: &[u8]) -> Result<u64, ProgramError> {
    if data.len() != 8 {
        return Err(ProgramError::InvalidInstructionData);
    }

    let amount_bytes: [u8; 8] = data[0..8]
        .try_into()
        .map_err(|_| ProgramError::InvalidInstructionData)?;

    Ok(u64::from_le_bytes(amount_bytes))
}
```

## CPI with Pinocchio

### Manual CPI construction
```rust
use pinocchio::{
    instruction::{AccountMeta, Instruction},
    program::invoke_signed,
};

fn transfer_tokens(
    token_program: &AccountInfo,
    from: &AccountInfo,
    to: &AccountInfo,
    authority: &AccountInfo,
    amount: u64,
    signer_seeds: &[&[&[u8]]],
) -> ProgramResult {
    // SPL Token transfer instruction data: [3 (discriminator), amount (8 bytes)]
    let mut instruction_data = [0u8; 9];
    instruction_data[0] = 3; // Transfer discriminator
    instruction_data[1..9].copy_from_slice(&amount.to_le_bytes());

    let transfer_ix = Instruction {
        program_id: *token_program.key(),
        accounts: vec![
            AccountMeta::new(*from.key(), false),
            AccountMeta::new(*to.key(), false),
            AccountMeta::new_readonly(*authority.key(), true),
        ],
        data: instruction_data.to_vec(),
    };

    let account_infos = &[
        from.clone(),
        to.clone(),
        authority.clone(),
    ];

    invoke_signed(&transfer_ix, account_infos, signer_seeds)
}
```

## Error Handling

### Use ProgramError directly
```rust
use pinocchio::program_error::ProgramError;

// Custom errors via custom code
const ERROR_INSUFFICIENT_FUNDS: u32 = 0;
const ERROR_UNAUTHORIZED: u32 = 1;
const ERROR_INVALID_AMOUNT: u32 = 2;

fn validate_transfer(balance: u64, amount: u64) -> ProgramResult {
    if amount == 0 {
        return Err(ProgramError::Custom(ERROR_INVALID_AMOUNT));
    }
    if balance < amount {
        return Err(ProgramError::Custom(ERROR_INSUFFICIENT_FUNDS));
    }
    Ok(())
}
```

## Performance Optimization

### Minimize allocations
```rust
// BAD - allocates Vec
let seeds = vec![b"vault".as_slice(), authority.as_ref()];

// GOOD - stack-allocated array
let seeds: &[&[u8]] = &[b"vault", authority.as_ref()];
```

### Use const for fixed sizes
```rust
const VAULT_SIZE: usize = 1 + 32 + 1 + 8 + 8; // discriminator + Vault fields

// Use in rent calculation
let rent_lamports = rent.minimum_balance(VAULT_SIZE);
```

### Feature-gate debug code
```rust
#[cfg(feature = "debug")]
{
    pinocchio::msg!("Processing deposit: {}", amount);
}
```

## Account Creation

### Manual account creation with System Program
```rust
use pinocchio::{
    program::invoke_signed,
    sysvars::rent::Rent,
};

fn create_account(
    payer: &AccountInfo,
    new_account: &AccountInfo,
    system_program: &AccountInfo,
    space: usize,
    program_id: &Pubkey,
    signer_seeds: &[&[&[u8]]],
) -> ProgramResult {
    let rent = Rent::get()?;
    let lamports = rent.minimum_balance(space);

    // System program create_account instruction
    // Discriminator: 0, followed by lamports (8), space (8), owner (32)
    let mut ix_data = [0u8; 49];
    ix_data[0..8].copy_from_slice(&lamports.to_le_bytes());
    ix_data[8..16].copy_from_slice(&(space as u64).to_le_bytes());
    ix_data[16..48].copy_from_slice(program_id.as_ref());

    let ix = Instruction {
        program_id: *system_program.key(),
        accounts: vec![
            AccountMeta::new(*payer.key(), true),
            AccountMeta::new(*new_account.key(), true),
        ],
        data: ix_data.to_vec(),
    };

    invoke_signed(&ix, &[payer.clone(), new_account.clone()], signer_seeds)
}
```

## Testing with Pinocchio

### Unit tests with Mollusk
```rust
#[cfg(test)]
mod tests {
    use super::*;
    use mollusk_svm::Mollusk;
    use solana_sdk::{
        account::Account,
        instruction::{AccountMeta, Instruction},
        pubkey::Pubkey,
    };

    #[test]
    fn test_initialize() {
        let program_id = Pubkey::new_unique();
        let mollusk = Mollusk::new(&program_id, "target/deploy/program");

        // Setup accounts
        let authority = Pubkey::new_unique();
        let (vault_pda, bump) = Pubkey::find_program_address(
            &[b"vault", authority.as_ref()],
            &program_id,
        );

        // Create instruction data: [0 (init discriminator), bump]
        let ix_data = vec![0, bump];

        // Create account data
        let vault_account = Account {
            lamports: 1_000_000,
            data: vec![0; Vault::account_size()],
            owner: program_id,
            executable: false,
            rent_epoch: 0,
        };

        let instruction = Instruction {
            program_id,
            accounts: vec![
                AccountMeta::new(vault_pda, false),
                AccountMeta::new_readonly(authority, true),
            ],
            data: ix_data,
        };

        let accounts = vec![
            (vault_pda, vault_account),
            (authority, Account::default()),
        ];

        // Execute
        let result = mollusk.process_instruction(&instruction, &accounts);

        assert!(result.program_result.is_ok());

        // Verify CU usage
        println!("CU consumed: {}", result.compute_units_consumed);
    }
}
```

## When to Use Pinocchio

### Use Pinocchio when:
- Maximum CU efficiency required
- Binary size constraints
- High-frequency program (e.g., DEX)
- You understand unsafe Rust
- Team has Solana expertise

### Don't use Pinocchio when:
- Team is learning Solana
- Development speed more important
- Program complexity is high
- Maintenance burden is a concern

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

## Best Practices

1. **Start with Anchor, optimize with Pinocchio if needed**
2. **Always benchmark CU savings** - measure before/after
3. **Document all unsafe code** - explain safety invariants
4. **Comprehensive testing** - more critical without Anchor safety nets
5. **Use TryFrom pattern** - maintains ergonomics with safety
6. **Store canonical bumps** - don't recalculate PDAs
7. **Single-byte discriminators** - saves space and CU
8. **Feature-gate logs** - minimize production overhead

---

**Remember**: Pinocchio is for optimization, not first implementation. Master Anchor first, then optimize hotspots with Pinocchio.

**Sources:**
- [Pinocchio GitHub](https://github.com/anza-xyz/pinocchio)
- [Pinocchio Guide](https://www.helius.dev/blog/pinocchio)
- [Pinocchio Optimization Patterns](https://www.quicknode.com/guides/solana-development/pinocchio/how-to-build-and-deploy-a-solana-program-using-pinocchio)
