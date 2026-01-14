# Frontend with framework-kit (Next.js / React)

## Goals
- One Solana client instance for the app (RPC + WS + wallet connectors)
- Wallet Standard-first discovery/connect
- Minimal "use client" footprint in Next.js (hooks only in leaf components)
- Transaction sending that is observable, cancelable, and UX-friendly
- Beautiful, accessible UI following 2026 design trends

## Recommended dependencies
- `@solana/client`
- `@solana/react-hooks`
- `@solana/kit`
- `@solana-program/system`, `@solana-program/token`, etc. (only what you need)

## Bootstrap recommendation
Prefer `create-solana-dapp` and pick a kit/framework-kit compatible template for new projects.

## Provider setup (Next.js App Router)

Create a single client and provide it via SolanaProvider.

Example `app/providers.tsx`:

```tsx
'use client';

import React from 'react';
import { SolanaProvider } from '@solana/react-hooks';
import { autoDiscover, createClient } from '@solana/client';

const endpoint =
  process.env.NEXT_PUBLIC_SOLANA_RPC_URL ?? 'https://api.devnet.solana.com';

// Some environments prefer an explicit WS endpoint; default to wss derived from https.
const websocketEndpoint =
  process.env.NEXT_PUBLIC_SOLANA_WS_URL ??
  endpoint.replace('https://', 'wss://').replace('http://', 'ws://');

export const solanaClient = createClient({
  endpoint,
  websocketEndpoint,
  walletConnectors: autoDiscover(),
});

export function Providers({ children }: { children: React.ReactNode }) {
  return <SolanaProvider client={solanaClient}>{children}</SolanaProvider>;
}
```

Then wrap `app/layout.tsx` with `<Providers>`.

## Hook usage patterns (high-level)

Prefer framework-kit hooks before writing your own store/subscription logic:

* `useWalletConnection()` for connect/disconnect and wallet discovery
* `useBalance(...)` for lamports balance
* `useSolTransfer(...)` for SOL transfers
* `useSplToken(...)` / token helpers for token balances/transfers
* `useTransactionPool(...)` for managing send + status + retry flows

When you need custom instructions, build them using `@solana-program/*` and send them via the framework-kit transaction helpers.

## Data fetching and subscriptions

* Prefer watchers/subscriptions rather than manual polling
* Clean up subscriptions with abort handles returned by watchers
* For Next.js: keep server components server-side; only leaf components that call hooks should be client components

## Transaction UX checklist

* Disable inputs while a transaction is pending
* Provide a signature immediately after send
* Track confirmation states (processed/confirmed/finalized) based on UX need
* Show actionable errors:
  * user rejected signing
  * insufficient SOL for fees / rent
  * blockhash expired / dropped
  * account already in use / already initialized
  * program error (custom error code)

## When to use ConnectorKit (optional)

If you need a headless connector with composable UI elements and explicit state control, use ConnectorKit.
Typical reasons:
* You want a headless wallet connection core (useful across frameworks)
* You want more control over wallet/account state than a single provider gives
* You need production diagnostics/health checks for wallet sessions

---

# Design Philosophy (2026)

## Core Principles
1. **Clarity Over Cleverness**: Users don't care about flashiness—they care about finding information quickly
2. **Purposeful Motion**: Animation should clarify relationships, not decorate
3. **Cognitive Inclusion**: Design for diverse minds (ADHD, autism, dyslexia)
4. **Accessibility is Non-Negotiable**: 4.5:1 contrast, 24x24px touch targets, keyboard navigation
5. **Performance is UX**: A fast interface feels trustworthy

## 2026 Visual Trends

### Liquid Glass Aesthetic
- Translucent surfaces with depth using `backdrop-filter: blur(12px)`
- Subtle border with `rgba(255, 255, 255, 0.2)`
- Light refraction and layering with `box-shadow`

### Calm UI
- Larger typography (16px+ body, 48px+ headings)
- Generous whitespace (8px grid system)
- Softer edges (`border-radius: 8-16px`)
- Muted, intentional color palettes

### Warm Neutrals
- Soft, "unbleached" backgrounds instead of pure white
- Paper-like tones reduce eye strain

### Avoid Generic AI Aesthetics
- Overused font families (Inter, Roboto, Arial, system fonts)
- Clichéd color schemes (particularly purple gradients on white backgrounds)
- Predictable layouts and component patterns
- Cookie-cutter design that lacks context-specific character

---

# Accessibility Requirements (WCAG 2.2 AA)

## Focus Management
- Always visible focus rings using `focus-visible:ring-2`
- Never remove focus indicators

## Color Contrast
- **4.5:1** for normal text
- **3:1** for large text and UI components

## Touch Targets
- **Minimum**: 24x24px
- **Recommended**: 44x44px for mobile

## ARIA
- Proper labels (`aria-label`, `aria-labelledby`)
- Error associations (`aria-describedby`)
- Live regions for dynamic content (`aria-live`)

## Keyboard Navigation
- Full tab navigation
- Roving tabindex for lists
- Skip links to main content

## Motion
- Always respect `prefers-reduced-motion`
- Use `useReducedMotion()` hook in Framer Motion

```tsx
import { useReducedMotion } from 'framer-motion';

function AnimatedComponent() {
  const shouldReduceMotion = useReducedMotion();
  
  return (
    <motion.div
      animate={{ opacity: 1, y: 0 }}
      transition={{ duration: shouldReduceMotion ? 0 : 0.3 }}
    />
  );
}
```

---

# Component Patterns

## Tailwind 4.0 @theme Syntax

Use the new @theme directive for design tokens:

```css
@theme {
  --spacing-*: /* 8px grid system */
  --font-size-*: /* modular scale 1.25 ratio */
  --color-primary: oklch(0.7 0.15 280);
  --color-solana-purple: #9945FF;
  --color-solana-green: #14F195;
}
```

## Wallet Connection Component

```tsx
'use client';

import { useWalletConnection } from '@solana/react-hooks';
import { useState } from 'react';

export function WalletButton() {
  const { wallet, connect, disconnect, connecting, connected } = useWalletConnection();
  const [showDropdown, setShowDropdown] = useState(false);

  if (!connected) {
    return (
      <button
        onClick={connect}
        disabled={connecting}
        className="px-4 py-2 bg-solana-purple text-white rounded-lg 
                   hover:bg-opacity-90 disabled:opacity-50
                   focus-visible:ring-2 focus-visible:ring-offset-2"
        aria-busy={connecting}
      >
        {connecting ? 'Connecting...' : 'Connect Wallet'}
      </button>
    );
  }

  return (
    <div className="relative">
      <button
        onClick={() => setShowDropdown(!showDropdown)}
        className="flex items-center gap-2 px-4 py-2 bg-surface rounded-lg"
        aria-expanded={showDropdown}
        aria-haspopup="menu"
      >
        <span className="w-2 h-2 bg-green-500 rounded-full" aria-hidden="true" />
        <span className="font-mono text-sm">
          {wallet?.publicKey.slice(0, 4)}...{wallet?.publicKey.slice(-4)}
        </span>
      </button>
      
      {showDropdown && (
        <div
          role="menu"
          className="absolute top-full mt-2 right-0 bg-surface rounded-lg shadow-lg p-2"
        >
          <button
            role="menuitem"
            onClick={() => navigator.clipboard.writeText(wallet?.publicKey || '')}
            className="w-full text-left px-4 py-2 hover:bg-white/10 rounded"
          >
            Copy Address
          </button>
          <button
            role="menuitem"
            onClick={disconnect}
            className="w-full text-left px-4 py-2 hover:bg-white/10 rounded text-red-400"
          >
            Disconnect
          </button>
        </div>
      )}
    </div>
  );
}
```

## Transaction Flow Component

```tsx
'use client';

import { AnimatePresence, motion } from 'framer-motion';

type TransactionState = 'idle' | 'signing' | 'confirming' | 'success' | 'error';

interface TransactionDialogProps {
  state: TransactionState;
  signature?: string;
  error?: string;
  onClose: () => void;
}

export function TransactionDialog({ state, signature, error, onClose }: TransactionDialogProps) {
  if (state === 'idle') return null;

  return (
    <div
      role="dialog"
      aria-labelledby="tx-title"
      aria-describedby="tx-description"
      className="fixed inset-0 flex items-center justify-center bg-black/50"
    >
      <motion.div
        initial={{ opacity: 0, scale: 0.95 }}
        animate={{ opacity: 1, scale: 1 }}
        exit={{ opacity: 0, scale: 0.95 }}
        className="bg-surface rounded-2xl p-6 max-w-md w-full mx-4"
      >
        <h2 id="tx-title" className="text-xl font-semibold mb-4">
          {state === 'signing' && 'Waiting for Signature'}
          {state === 'confirming' && 'Confirming Transaction'}
          {state === 'success' && 'Transaction Successful'}
          {state === 'error' && 'Transaction Failed'}
        </h2>
        
        <p id="tx-description" className="text-gray-400 mb-4">
          {state === 'signing' && 'Please approve the transaction in your wallet'}
          {state === 'confirming' && 'Transaction is being confirmed on the network'}
          {state === 'success' && 'Your transaction has been confirmed'}
          {state === 'error' && error}
        </p>

        <AnimatePresence mode="wait">
          {state === 'signing' && (
            <motion.div
              key="signing"
              initial={{ opacity: 0 }}
              animate={{ opacity: 1 }}
              exit={{ opacity: 0 }}
              className="flex justify-center"
            >
              <div className="animate-spin w-8 h-8 border-2 border-solana-purple border-t-transparent rounded-full" />
            </motion.div>
          )}

          {state === 'success' && signature && (
            <motion.a
              key="success"
              initial={{ opacity: 0, y: 10 }}
              animate={{ opacity: 1, y: 0 }}
              href={`https://solscan.io/tx/${signature}`}
              target="_blank"
              rel="noopener noreferrer"
              className="inline-flex items-center gap-2 text-solana-green hover:underline"
            >
              View on Solscan ↗
            </motion.a>
          )}
        </AnimatePresence>

        {(state === 'success' || state === 'error') && (
          <button
            onClick={onClose}
            className="mt-4 w-full py-2 bg-white/10 rounded-lg hover:bg-white/20 
                       focus-visible:ring-2 focus-visible:ring-offset-2"
          >
            Close
          </button>
        )}
      </motion.div>
    </div>
  );
}
```

## Token Balance Display

```tsx
'use client';

import { useMemo } from 'react';

interface TokenBalanceProps {
  amount: bigint;
  decimals: number;
  symbol: string;
}

export function TokenBalance({ amount, decimals, symbol }: TokenBalanceProps) {
  const formattedAmount = useMemo(() => {
    const divisor = BigInt(10 ** decimals);
    const whole = amount / divisor;
    const fraction = amount % divisor;
    
    // Format with K, M suffixes for large numbers
    if (whole >= 1_000_000n) {
      return `${(Number(whole) / 1_000_000).toFixed(2)}M`;
    }
    if (whole >= 1_000n) {
      return `${(Number(whole) / 1_000).toFixed(2)}K`;
    }
    
    const fractionStr = fraction.toString().padStart(decimals, '0').slice(0, 4);
    return `${whole}.${fractionStr}`;
  }, [amount, decimals]);

  return (
    <span className="font-mono tabular-nums">
      {formattedAmount} <span className="text-gray-400">{symbol}</span>
    </span>
  );
}
```

## Address Display with Copy

```tsx
'use client';

import { useState } from 'react';

interface AddressDisplayProps {
  address: string;
  truncate?: boolean;
}

export function AddressDisplay({ address, truncate = true }: AddressDisplayProps) {
  const [copied, setCopied] = useState(false);

  const displayAddress = truncate
    ? `${address.slice(0, 4)}...${address.slice(-4)}`
    : address;

  const handleCopy = async () => {
    await navigator.clipboard.writeText(address);
    setCopied(true);
    setTimeout(() => setCopied(false), 2000);
  };

  return (
    <button
      onClick={handleCopy}
      className="inline-flex items-center gap-2 font-mono text-sm 
                 hover:text-solana-green transition-colors
                 focus-visible:ring-2 focus-visible:ring-offset-2"
      aria-label={`Copy address ${address}`}
    >
      <span>{displayAddress}</span>
      <span className="text-xs" aria-live="polite">
        {copied ? '✓' : '📋'}
      </span>
    </button>
  );
}
```

---

# Animation Guidelines

| Type | Duration | Use Case |
|------|----------|----------|
| Micro | 150-200ms | Hover states, button feedback |
| Small | 200-300ms | Tooltips, dropdowns |
| Medium | 300-400ms | Modals, cards |
| Large | 400-600ms | Page transitions |

Always use `useReducedMotion()` hook and respect `prefers-reduced-motion`.

---

# Critical Rules

## NEVER
- Skip accessibility (`alt`, `aria-*`, semantic HTML)
- Use raw color values (use semantic tokens)
- Use `any` in TypeScript
- Animate layout properties (use transforms/opacity)
- Forget loading/error/empty states
- Hardcode text strings

## ALWAYS
- Test on mobile first
- Test with keyboard navigation
- Handle all states (loading, error, empty, success)
- Use semantic HTML (`button` for actions, `a` for links)
- Provide visible focus indicators
- Validate input client and server side
- Provide immediate feedback for user actions

---

# Form Validation with Solana Types

```tsx
import { z } from 'zod';

// Solana-specific validators
const solanaAddressSchema = z.string()
  .length(44, 'Solana addresses are 44 characters')
  .regex(/^[1-9A-HJ-NP-Za-km-z]+$/, 'Invalid base58 character');

const tokenAmountSchema = z.object({
  amount: z.string()
    .regex(/^\d*\.?\d+$/, 'Invalid amount')
    .refine((val) => parseFloat(val) > 0, 'Amount must be positive'),
  decimals: z.number().int().min(0).max(18),
});

export const transferSchema = z.object({
  recipient: solanaAddressSchema,
  amount: tokenAmountSchema,
});
```

---

**Sources:**
- [framework-kit Repository](https://github.com/solana-foundation/framework-kit)
- [ConnectorKit](https://github.com/civic-io/connector-kit)
- [create-solana-dapp](https://github.com/solana-developers/create-solana-dapp)
