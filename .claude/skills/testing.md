# Testing Solana Programs

## Testing Framework Hierarchy

| Framework | Type | Speed | Best For |
|-----------|------|-------|----------|
| **Mollusk** | Unit (per-IX) | Fastest | Single instruction isolation |
| **LiteSVM** | Unit/Integration | Fast | Multi-instruction flows in-process |
| **Trident** | Fuzz | Variable | Finding edge cases, security testing |
| **Surfpool** | Integration | Medium | Mainnet state locally, realistic scenarios |
| **solana-test-validator** | Integration | Slowest | Full RPC behavior, legacy tests |

## Quick Decision Guide

- **New project, unit tests**: Start with LiteSVM or Mollusk
- **Critical financial logic**: Add Trident fuzz tests
- **Testing against live state**: Use Surfpool
- **Legacy test suite exists**: Keep solana-test-validator

---

## Mollusk (Single-Instruction Unit Testing)

Mollusk is ideal for testing individual instructions in isolation with minimal overhead.

### Setup

```toml
# Cargo.toml
[dev-dependencies]
mollusk-svm = "0.1"
```

### Basic Usage

```rust
#[cfg(test)]
mod tests {
    use mollusk_svm::Mollusk;
    use solana_sdk::{
        account::Account,
        instruction::Instruction,
        pubkey::Pubkey,
    };

    #[test]
    fn test_initialize_instruction() {
        let program_id = Pubkey::new_unique();
        let mollusk = Mollusk::new(&program_id, "target/deploy/my_program");

        let authority = Pubkey::new_unique();
        let (vault_pda, bump) = Pubkey::find_program_address(
            &[b"vault", authority.as_ref()],
            &program_id,
        );

        // Create accounts
        let authority_account = (
            authority,
            Account {
                lamports: 1_000_000_000,
                ..Account::default()
            },
        );

        let vault_account = (
            vault_pda,
            Account {
                lamports: 0,
                owner: system_program::ID,
                ..Account::default()
            },
        );

        // Build instruction
        let mut ix_data = vec![0u8]; // discriminator
        ix_data.extend_from_slice(&1000u64.to_le_bytes());

        let instruction = Instruction {
            program_id,
            accounts: vec![
                AccountMeta::new(authority, true),
                AccountMeta::new(vault_pda, false),
            ],
            data: ix_data,
        };

        // Execute
        let result = mollusk.process_instruction(
            &instruction,
            &[authority_account, vault_account],
        );

        assert!(result.program_result.is_ok());
    }
}
```

### CU Benchmarking with Mollusk

Track compute unit usage over time:

```rust
use mollusk_svm::MolluskComputeUnitBencher;

#[test]
fn bench_instructions() {
    let program_id = Pubkey::new_unique();
    let mollusk = Mollusk::new(&program_id, "target/deploy/my_program");

    let bencher = MolluskComputeUnitBencher::new(mollusk)
        .must_pass(true)
        .out_dir("../target/benches");

    // Bench different instructions
    bencher.bench("initialize", &init_ix, &init_accounts);
    bencher.bench("deposit", &deposit_ix, &deposit_accounts);
    bencher.bench("withdraw", &withdraw_ix, &withdraw_accounts);

    // Generates markdown report with CU usage and deltas vs previous runs
}
```

---

## LiteSVM (Fast Multi-Instruction Testing)

LiteSVM is a lightweight SVM implementation for running multiple instructions in sequence.

### Setup

```toml
# Cargo.toml
[dev-dependencies]
litesvm = "0.3"
```

### Basic Usage

```rust
#[cfg(test)]
mod tests {
    use litesvm::LiteSVM;
    use solana_sdk::{
        signature::{Keypair, Signer},
        transaction::Transaction,
    };

    #[test]
    fn test_program_flow() {
        // Create SVM instance
        let mut svm = LiteSVM::new();

        // Fund authority
        let authority = Keypair::new();
        svm.airdrop(&authority.pubkey(), 10 * LAMPORTS_PER_SOL).unwrap();

        // Deploy program
        let program_id = Pubkey::new_unique();
        let program_bytes = std::fs::read("target/deploy/my_program.so").unwrap();
        svm.add_program(program_id, &program_bytes);

        // Build and send transaction
        let tx = Transaction::new_signed_with_payer(
            &[initialize_ix(&program_id, &authority.pubkey())],
            Some(&authority.pubkey()),
            &[&authority],
            svm.latest_blockhash(),
        );

        let result = svm.send_transaction(tx);
        assert!(result.is_ok());

        // Verify state
        let account = svm.get_account(&vault_pda).unwrap();
        assert!(account.is_some());
    }

    #[test]
    fn test_multi_step_flow() {
        let mut svm = LiteSVM::new();
        let authority = Keypair::new();
        svm.airdrop(&authority.pubkey(), 10 * LAMPORTS_PER_SOL).unwrap();

        // Step 1: Initialize
        let tx1 = build_init_tx(&svm, &authority);
        svm.send_transaction(tx1).unwrap();

        // Step 2: Deposit
        let tx2 = build_deposit_tx(&svm, &authority, 1_000_000_000);
        svm.send_transaction(tx2).unwrap();

        // Step 3: Withdraw
        let tx3 = build_withdraw_tx(&svm, &authority, 500_000_000);
        svm.send_transaction(tx3).unwrap();

        // Verify final state
        let vault = fetch_vault_account(&svm, &authority.pubkey());
        assert_eq!(vault.balance, 500_000_000);
    }
}
```

### Time Manipulation

```rust
#[test]
fn test_time_locked_withdrawal() {
    let mut svm = LiteSVM::new();
    
    // Setup...
    
    // Try to withdraw before lock expires
    let result = svm.send_transaction(withdraw_tx.clone());
    assert!(result.is_err());
    
    // Advance time by 1 day
    svm.warp_to_slot(svm.get_slot() + (24 * 60 * 60 / 400)); // ~400ms slots
    
    // Now withdrawal should succeed
    let result = svm.send_transaction(withdraw_tx);
    assert!(result.is_ok());
}
```

---

## Trident (Fuzz Testing)

Trident uses AFL/libfuzzer to find edge cases in your program logic.

### Setup

```bash
cargo install trident-cli
trident init
```

### Configuration

```toml
# Trident.toml
[fuzz]
fuzzing_with_stats = true
allow_duplicate_txs = false
programs_data = ["target/deploy/my_program.so"]

[fuzz.test]
iterations = 10000
exit_upon_crash = true
```

### Fuzz Test Definition

```rust
// trident-tests/fuzz_tests/fuzz_0/test_fuzz.rs
use trident_fuzz::fuzzing::*;
use my_program::*;

#[derive(Arbitrary, Debug)]
pub struct InitializeData {
    pub amount: u64,
}

impl FuzzDataBuilder<InitializeData> for MyProgramFuzzContext {
    fn build(
        &self,
        u: &mut arbitrary::Unstructured,
    ) -> arbitrary::Result<InitializeData> {
        Ok(InitializeData {
            amount: u.arbitrary()?,
        })
    }
}

#[flow]
impl<'info> InstructionFlow<InitializeData, Initialize<'info>> for MyProgramFuzzContext {
    fn instruction(
        &mut self,
        accounts: &Initialize<'info>,
        data: &InitializeData,
    ) -> Result<()> {
        // Fuzz instruction with arbitrary data
        my_program::instructions::initialize(
            accounts,
            data.amount,
        )
    }
}

#[invariant]
fn vault_balance_invariant(ctx: &MyProgramFuzzContext) -> bool {
    // Invariant: vault balance should never exceed total deposits
    let vault = ctx.get_vault_account();
    let total_deposits = ctx.get_total_deposits();
    vault.balance <= total_deposits
}
```

### Running Fuzz Tests

```bash
# Run fuzz tests
trident fuzz run fuzz_0

# With coverage
trident fuzz run fuzz_0 --coverage

# View crash reports
ls trident-tests/fuzz_tests/fuzz_0/crashes/
```

### Critical Paths to Fuzz

1. **Arithmetic operations**: Overflow, underflow, division
2. **Access control**: Authority checks, signer validation
3. **Token transfers**: Amount calculations, fee handling
4. **PDA derivation**: Seed collision, bump handling
5. **Account state transitions**: Initialization, closure, reallocation

---

## Surfpool (Mainnet State Locally)

Surfpool maintains a local copy of mainnet/devnet state for realistic testing.

### Setup

```bash
# Install Surfpool
cargo install surfpool-cli

# Clone mainnet state for specific accounts
surfpool clone mainnet --accounts ./accounts.json

# Start local environment with cloned state
surfpool start
```

### accounts.json Example

```json
{
  "accounts": [
    "So11111111111111111111111111111111111111112",
    "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
    "YOUR_PROGRAM_ID"
  ],
  "programs": [
    "YOUR_PROGRAM_ID"
  ]
}
```

### Testing Against Live State

```typescript
import { Connection, PublicKey } from '@solana/web3.js';

describe('Integration with Mainnet State', () => {
  const connection = new Connection('http://localhost:8899'); // Surfpool

  it('should interact with real token accounts', async () => {
    // Real USDC mint
    const usdcMint = new PublicKey('EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v');
    
    const mintInfo = await connection.getAccountInfo(usdcMint);
    expect(mintInfo).toBeDefined();
    
    // Your integration tests here...
  });
});
```

---

## CI Integration

### GitHub Actions Workflow

```yaml
name: Test Solana Program

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install Rust
        uses: dtolnay/rust-action@stable

      - name: Install Solana CLI
        run: |
          sh -c "$(curl -sSfL https://release.anza.xyz/stable/install)"
          echo "$HOME/.local/share/solana/install/active_release/bin" >> $GITHUB_PATH

      - name: Cache dependencies
        uses: Swatinem/rust-cache@v2

      - name: Build program
        run: cargo build-sbf

      - name: Run unit tests (Mollusk/LiteSVM)
        run: cargo test --lib

      - name: Run integration tests
        run: cargo test --test '*'

  fuzz:
    runs-on: ubuntu-latest
    needs: test
    steps:
      - uses: actions/checkout@v4

      - name: Install Trident
        run: cargo install trident-cli

      - name: Run fuzz tests (limited iterations for CI)
        run: trident fuzz run-all --iterations 1000

      - name: Upload crash reports
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: fuzz-crashes
          path: trident-tests/fuzz_tests/**/crashes/

  benchmark:
    runs-on: ubuntu-latest
    needs: test
    steps:
      - uses: actions/checkout@v4

      - name: Run CU benchmarks
        run: cargo test --test bench -- --nocapture

      - name: Upload benchmark results
        uses: actions/upload-artifact@v4
        with:
          name: cu-benchmarks
          path: target/benches/*.md
```

---

## Test Organization

### Recommended Structure

```
programs/my-program/
├── src/
│   └── lib.rs
├── tests/
│   ├── common/
│   │   └── mod.rs          # Shared test utilities
│   ├── unit/
│   │   ├── initialize.rs   # Mollusk unit tests
│   │   ├── deposit.rs
│   │   └── withdraw.rs
│   └── integration/
│       └── full_flow.rs    # LiteSVM integration tests
└── trident-tests/
    └── fuzz_tests/
        └── fuzz_0/
            └── test_fuzz.rs
```

### Common Test Utilities

```rust
// tests/common/mod.rs
use solana_sdk::{pubkey::Pubkey, signature::Keypair};

pub fn setup_test_accounts(program_id: &Pubkey) -> TestAccounts {
    let authority = Keypair::new();
    let (vault_pda, bump) = Pubkey::find_program_address(
        &[b"vault", authority.pubkey().as_ref()],
        program_id,
    );
    
    TestAccounts {
        authority,
        vault_pda,
        vault_bump: bump,
    }
}

pub struct TestAccounts {
    pub authority: Keypair,
    pub vault_pda: Pubkey,
    pub vault_bump: u8,
}
```

---

## Coverage and Reporting

### Code Coverage with cargo-llvm-cov

```bash
# Install
cargo install cargo-llvm-cov

# Run with coverage
cargo llvm-cov --lib --html

# Open report
open target/llvm-cov/html/index.html
```

### Minimum Coverage Requirements

- **Critical paths** (token transfers, access control): 100%
- **Happy paths**: 90%+
- **Error paths**: 80%+
- **Edge cases**: Handled by fuzz testing

---

## Best Practices

### 1. Test Both Success and Failure

```rust
#[test]
fn test_unauthorized_withdrawal() {
    let mut svm = LiteSVM::new();
    let owner = Keypair::new();
    let attacker = Keypair::new();
    
    // Setup vault owned by `owner`
    
    // Attacker tries to withdraw
    let tx = build_withdraw_tx(&svm, &attacker, vault_pda, 100);
    let result = svm.send_transaction(tx);
    
    assert!(result.is_err());
    assert!(result.unwrap_err().to_string().contains("Unauthorized"));
}
```

### 2. Test Boundary Conditions

```rust
#[test]
fn test_deposit_boundary_conditions() {
    let mut svm = LiteSVM::new();
    
    // Zero amount
    let result = deposit(&mut svm, 0);
    assert!(result.is_err());
    
    // Max u64
    let result = deposit(&mut svm, u64::MAX);
    assert!(result.is_err()); // Should overflow or hit rent limits
    
    // Just under overflow threshold
    let result = deposit(&mut svm, u64::MAX - 1000);
    assert!(result.is_err());
}
```

### 3. Test State Consistency

```rust
#[test]
fn test_state_consistency_after_error() {
    let mut svm = LiteSVM::new();
    let authority = Keypair::new();
    
    // Setup with initial balance
    initialize_vault(&mut svm, &authority, 1000);
    
    let state_before = get_vault_state(&svm, &authority);
    
    // Attempt invalid operation
    let result = withdraw(&mut svm, &authority, 9999);
    assert!(result.is_err());
    
    // State should be unchanged
    let state_after = get_vault_state(&svm, &authority);
    assert_eq!(state_before, state_after);
}
```

### 4. Use Property-Based Testing

```rust
use proptest::prelude::*;

proptest! {
    #[test]
    fn deposit_then_withdraw_preserves_balance(
        deposit_amount in 1u64..1_000_000_000,
        withdraw_amount in 1u64..1_000_000_000,
    ) {
        prop_assume!(withdraw_amount <= deposit_amount);
        
        let mut svm = setup_test_svm();
        let authority = Keypair::new();
        
        deposit(&mut svm, &authority, deposit_amount).unwrap();
        withdraw(&mut svm, &authority, withdraw_amount).unwrap();
        
        let final_balance = get_balance(&svm, &authority);
        assert_eq!(final_balance, deposit_amount - withdraw_amount);
    }
}
```

---

**Sources:**
- [Mollusk GitHub](https://github.com/buffalojoec/mollusk)
- [LiteSVM GitHub](https://github.com/LiteSVM/litesvm)
- [Trident Documentation](https://ackee.xyz/trident/docs/latest/)
- [Surfpool Guide](https://www.helius.dev/blog/surfpool)
