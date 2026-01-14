---
name: solana-dev
description: End-to-end Solana development playbook (Jan 2026). Covers on-chain programs (Anchor/Pinocchio), frontend (framework-kit, @solana/kit), backend services (Axum, indexers), testing (LiteSVM/Mollusk/Trident/Surfpool), CI/CD automation, deployment workflows, and security hardening. Authoritative patterns for the full Solana stack.
user-invocable: true
---

# Solana Development Skill

## What this Skill is for

Use this Skill when the user asks for:
- Solana dApp UI work (React / Next.js)
- Wallet connection + signing flows
- Transaction building / sending / confirmation UX
- On-chain program development (Anchor or Pinocchio)
- Client SDK generation (typed program clients)
- Local testing (LiteSVM, Mollusk, Trident, Surfpool)
- Backend services (Axum, indexers, webhooks)
- CI/CD and deployment workflows
- Security hardening and audit-style reviews

## Default stack decisions (opinionated)

### 1. UI: framework-kit first
- Use `@solana/client` + `@solana/react-hooks`
- Prefer Wallet Standard discovery/connect via the framework-kit client

### 2. SDK: @solana/kit first
- Prefer Kit types (`Address`, `Signer`, transaction message APIs, codecs)
- Prefer `@solana-program/*` instruction builders over hand-rolled instruction data

### 3. Legacy compatibility: web3.js only at boundaries
- If you must integrate a library that expects web3.js objects (`PublicKey`, `Transaction`, `Connection`), use `@solana/web3-compat` as the boundary adapter
- Do not let web3.js types leak across the entire app; contain them to adapter modules

### 4. Programs
- **Default**: Anchor (fast iteration, IDL generation, mature tooling)
- **Performance/footprint**: Pinocchio when you need CU optimization, minimal binary size, zero dependencies, or fine-grained control

### 5. Testing
- **Default**: LiteSVM or Mollusk for unit tests (fast feedback, runs in-process)
- **Fuzz testing**: Trident for finding edge cases
- **Integration**: Surfpool for realistic cluster state (mainnet/devnet) locally
- **Legacy**: solana-test-validator only when you need specific RPC behaviors

### 6. Backend Services
- **Framework**: Axum 0.8+ (no `#[async_trait]` needed!)
- **Runtime**: Tokio 1.40+
- **Database**: PostgreSQL with sqlx (compile-time checked)
- **Caching**: Redis for RPC response caching

### 7. CI/CD & Deployment
- Automated security checks on every commit
- Verifiable builds for mainnet deployments
- Pre-commit hooks for format, lint, test
- Devnet testing before mainnet

## Operating procedure (how to execute tasks)

### 1. Classify the task layer
- UI/wallet/hook layer
- Client SDK/scripts layer
- Program layer (+ IDL)
- Backend services layer
- Testing/CI layer
- Deployment/infra layer

### 2. Pick the right building blocks
- UI: framework-kit patterns → `frontend-framework-kit.md`
- Scripts/backends: @solana/kit directly → `kit-web3-interop.md`
- Legacy library present: introduce a web3-compat adapter boundary
- High-performance programs: Pinocchio over Anchor → `programs-pinocchio.md`
- Backend APIs: Axum patterns → `backend-async.md`

### 3. Implement with Solana-specific correctness
Always be explicit about:
- cluster + RPC endpoints + websocket endpoints
- fee payer + recent blockhash
- compute budget + prioritization (where relevant)
- expected account owners + signers + writability
- token program variant (SPL Token vs Token-2022) and any extensions
- **Account reloading after CPIs** (critical for correctness)

### 4. Add tests
- Unit test: LiteSVM or Mollusk
- Fuzz test: Trident for critical paths
- Integration test: Surfpool
- For "wallet UX", add mocked hook/provider tests where appropriate

### 5. Security review
Before any deployment:
- Run security checklist from `security.md`
- Verify all account validation
- Check arithmetic safety
- Validate CPI targets

### 6. Deliverables expectations
When you implement changes, provide:
- exact files changed + diffs (or patch-style output)
- commands to install/build/test
- a short "risk notes" section for anything touching signing/fees/CPIs/token transfers

## Progressive disclosure (read when needed)

### Core Development
- Anchor programs: [programs-anchor.md](programs-anchor.md)
- Pinocchio programs: [programs-pinocchio.md](programs-pinocchio.md)
- Testing strategy: [testing.md](testing.md)

### Frontend
- UI + wallet + hooks: [frontend-framework-kit.md](frontend-framework-kit.md)
- Kit ↔ web3.js boundary: [kit-web3-interop.md](kit-web3-interop.md)

### Backend & Infrastructure
- Async Rust services: [backend-async.md](backend-async.md)
- Deployment workflows: [deployment.md](deployment.md)
- CI/CD automation: [ci-cd.md](ci-cd.md)

### Specialized
- IDLs + codegen: [idl-codegen.md](idl-codegen.md)
- Payments: [payments.md](payments.md)
- Security checklist: [security.md](security.md)
- Ecosystem knowledge: [ecosystem.md](ecosystem.md)
- Reference links: [resources.md](resources.md)

## Agent Integration

When working with specialized agents, they reference this skill:
- **solana-architect**: System design, delegates to skill for patterns
- **anchor-specialist**: Uses `programs-anchor.md` for implementation
- **pinocchio-engineer**: Uses `programs-pinocchio.md` for optimization
- **solana-frontend-engineer**: Uses `frontend-framework-kit.md` for UI
- **rust-backend-engineer**: Uses `backend-async.md` for services
- **tech-docs-writer**: Documents based on skill patterns
