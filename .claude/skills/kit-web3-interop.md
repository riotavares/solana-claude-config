# Kit ↔ web3.js Interop (boundary patterns)

## The rule
- New code: Kit types and Kit-first APIs.
- Legacy dependencies: isolate web3.js-shaped types behind an adapter boundary.

## Preferred bridge: @solana/web3-compat
Use `@solana/web3-compat` when:
- A dependency expects `PublicKey`, `Keypair`, `Transaction`, `VersionedTransaction`, `Connection`, etc.
- You are migrating an existing web3.js codebase incrementally.

### Why this approach works
- web3-compat re-exports web3.js-like types and delegates to Kit where possible.
- It includes helper conversions to move between web3.js and Kit representations.

## Practical boundary layout
Keep these modules separate:

- `src/solana/kit/`:
  - all Kit-first code: addresses, instruction builders, tx assembly, typed codecs, generated clients

- `src/solana/web3/`:
  - adapters for legacy libs (Anchor TS client, older SDKs)
  - conversions between `PublicKey` and Kit `Address`
  - conversions between web3 `TransactionInstruction` and Kit instruction shapes (only at edges)

## Conversion helpers (examples)
Use web3-compat helpers such as:
- `toAddress(...)`
- `toPublicKey(...)`
- `toWeb3Instruction(...)`
- `toKitSigner(...)`

## When you still need @solana/web3.js
Some methods outside web3-compat's compatibility surface may fall back to a legacy web3.js implementation.
If that happens:
- keep `@solana/web3.js` as an explicit dependency
- isolate fallback usage to adapter modules only
- avoid letting `PublicKey` bleed into your core domain types

## Common mistakes to prevent
- Mixing `Address` and `PublicKey` throughout the app (causes type drift and confusion)
- Building transactions in one stack and signing in another without explicit conversion
- Passing web3.js `Connection` into Kit-native code (or vice versa) rather than using a single source of truth

## Decision checklist
If you're about to add web3.js:
1) Is there a Kit-native equivalent? Prefer Kit.
2) Is the only reason a dependency? Use web3-compat at the boundary.
3) Can you generate a Kit-native client (Codama) instead? Prefer codegen.

## Type Conversion Examples

### Address ↔ PublicKey

```typescript
import { address, type Address } from '@solana/kit';
import { PublicKey } from '@solana/web3.js';

// Kit Address → web3.js PublicKey
const kitAddress: Address = address('11111111111111111111111111111111');
const publicKey = new PublicKey(kitAddress);

// web3.js PublicKey → Kit Address
const pk = new PublicKey('11111111111111111111111111111111');
const addr: Address = address(pk.toBase58());
```

### Transaction Interop

```typescript
import { toWeb3Instruction, toKitInstruction } from '@solana/web3-compat';

// Kit instruction → web3.js TransactionInstruction
const web3Ix = toWeb3Instruction(kitInstruction);

// web3.js → Kit (at boundary only)
const kitIx = toKitInstruction(legacyInstruction);
```

## Migration Strategy

### Phase 1: Boundary Adapter
```
src/
├── core/           # Kit-first (new code)
├── adapters/       # web3-compat boundary
└── legacy/         # Old web3.js code (to migrate)
```

### Phase 2: Incremental Migration
- Replace `PublicKey` with `Address` in core modules
- Replace `Connection` with Kit RPC client
- Update transaction building to Kit patterns

### Phase 3: Remove Legacy
- Remove `@solana/web3.js` dependency
- Keep only `@solana/web3-compat` if still needed for external libs
