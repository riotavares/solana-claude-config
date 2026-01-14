# IDLs + client generation (Codama / Shank / Kinobi)

## Goal
Never hand-maintain multiple program clients by manually re-implementing serializers.
Prefer an IDL-driven, code-generated workflow.

## Codama (preferred)
- Use Codama as the "single program description format" to generate:
  - TypeScript clients (including Kit-friendly output)
  - Rust clients (when available/needed)
  - documentation artifacts

## Anchor → Codama
If the program is Anchor:
1) Produce Anchor IDL from the build
2) Convert Anchor IDL to Codama nodes (nodes-from-anchor)
3) Render a Kit-native TypeScript client (codama renderers)

```bash
# Build and generate IDL
anchor build

# Convert to Codama and generate clients
npx @codama/cli generate \
    --idl target/idl/my_program.json \
    --output clients/ts/
```

## Native Rust → Shank → Codama
If the program is native Rust (not Anchor):
1) Use Shank macros to extract a Shank IDL from annotated Rust
2) Convert Shank IDL to Codama
3) Generate clients via Codama renderers

```rust
// In your program
use shank::ShankAccount;

#[derive(ShankAccount)]
pub struct Vault {
    pub authority: Pubkey,
    pub balance: u64,
    pub bump: u8,
}
```

```bash
# Generate Shank IDL
cargo install shank-cli
shank idl --out-dir idl programs/my-program/src/lib.rs

# Convert to Codama
npx @codama/nodes-from-shank --idl idl/my_program.json

# Generate clients
npx @codama/cli generate
```

## Pinocchio Programs
Pinocchio doesn't auto-generate IDL. Use Shank annotations:

```rust
use shank::{ShankAccount, ShankInstruction};

#[derive(ShankInstruction)]
pub enum MyInstruction {
    #[account(0, writable, signer, name = "authority")]
    #[account(1, writable, name = "vault")]
    Initialize,

    #[account(0, writable, signer, name = "authority")]
    #[account(1, writable, name = "vault")]
    Deposit { amount: u64 },
}
```

## Repository structure recommendation

```
my-program/
├── programs/
│   └── my-program/
│       └── src/
│           └── lib.rs
├── idl/
│   ├── my_program_anchor.json    # Anchor IDL
│   └── my_program_codama.json    # Codama IDL
├── clients/
│   ├── ts/
│   │   └── my-program/
│   │       ├── package.json
│   │       └── src/
│   │           └── generated/     # Codama output
│   └── rust/
│       └── my-program/           # Generated Rust client
└── codama.config.ts              # Codama configuration
```

## Codama Configuration

```typescript
// codama.config.ts
import { createFromIdl } from '@codama/nodes-from-idl';
import { renderJavaScriptVisitor } from '@codama/renderers-js';
import { renderRustVisitor } from '@codama/renderers-rust';

export default {
  idlPath: 'idl/my_program.json',
  outputs: [
    {
      visitor: renderJavaScriptVisitor('clients/ts/src/generated'),
      options: {
        prettierOptions: { semi: true, singleQuote: true },
      },
    },
    {
      visitor: renderRustVisitor('clients/rust/src/generated'),
    },
  ],
};
```

## Generation guardrails
- Codegen outputs should be checked into git if:
  - you need deterministic builds
  - you want users to consume the client without running codegen
- Otherwise, keep codegen in CI and publish artifacts.

## CI Integration

```yaml
# .github/workflows/codegen.yml
name: Generate Clients

on:
  push:
    paths:
      - 'programs/**'
      - 'idl/**'

jobs:
  generate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Build program
        run: anchor build
      
      - name: Generate clients
        run: npx @codama/cli generate
      
      - name: Commit generated code
        uses: stefanzweifel/git-auto-commit-action@v5
        with:
          commit_message: "chore: regenerate clients"
          file_pattern: "clients/**"
```

## Using Generated Clients

### TypeScript (Kit-native)

```typescript
import { 
  getInitializeInstruction,
  getDepositInstruction,
  fetchVault,
} from './generated';

// Build instruction with type safety
const initIx = getInitializeInstruction({
  authority: wallet.publicKey,
  vault: vaultPda,
  systemProgram: SystemProgram.programId,
});

// Fetch and decode accounts
const vault = await fetchVault(rpc, vaultAddress);
console.log(vault.data.balance);
```

### Rust

```rust
use my_program_client::{
    instructions::InitializeBuilder,
    accounts::Vault,
};

let ix = InitializeBuilder::new()
    .authority(authority.pubkey())
    .vault(vault_pda)
    .instruction();
```

## "Do not do this"
- Do not write IDLs by hand unless you have no alternative.
- Do not hand-write Borsh layouts for programs you own; use the IDL/codegen pipeline.
- Do not maintain separate client implementations manually.
- Do not skip IDL version bumps when program changes.

## Best Practices

1. **Version your IDL** alongside your program version
2. **Regenerate on every program change** via CI
3. **Test generated clients** as part of your test suite
4. **Publish clients** to npm/crates.io for external consumption
5. **Document breaking changes** when IDL structure changes
