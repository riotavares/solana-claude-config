# Payments and Commerce

## When payments are in scope
Use this guidance when the user asks about:
- checkout flows, tips, payment buttons
- payment request URLs / QR codes
- fee abstraction / gasless transactions
- merchant integrations

## Commerce Kit (preferred)
Use Commerce Kit as the default for payment experiences:
- drop-in payment UI components (buttons, modals, checkout flows)
- headless primitives for building custom checkout experiences
- React hooks for merchant/payment workflows
- built-in payment verification and confirmation handling
- support for SOL and SPL token payments

### When to use Commerce Kit
- You want a production-ready payment flow with minimal setup
- You need both UI components and headless APIs
- You want built-in best practices for payment verification
- You're building merchant experiences (tipping, checkout, subscriptions)

### Commerce Kit patterns
- Use the provided hooks for payment state management
- Leverage the built-in confirmation tracking (don't roll your own)
- Use the headless APIs when you need custom UI but want the payment logic handled

## Solana Pay

### Payment Request URLs

```typescript
import { createQR, encodeURL } from '@solana/pay';
import { PublicKey, Keypair } from '@solana/web3.js';
import BigNumber from 'bignumber.js';

// Create payment URL
const recipient = new PublicKey('MERCHANT_WALLET');
const reference = new Keypair().publicKey; // Unique per transaction

const url = encodeURL({
  recipient,
  amount: new BigNumber(1.5),     // 1.5 SOL
  reference,                       // For tracking
  label: 'My Store',
  message: 'Order #12345',
  memo: 'order:12345',
});

// Generate QR code
const qr = createQR(url, 512, 'white', 'black');
```

### SPL Token Payments

```typescript
const url = encodeURL({
  recipient,
  amount: new BigNumber(10),
  splToken: new PublicKey('USDC_MINT'),
  reference,
  label: 'My Store',
  message: 'Pay 10 USDC',
});
```

### Transaction Verification

```typescript
import { findReference, validateTransfer } from '@solana/pay';

async function verifyPayment(
  connection: Connection,
  reference: PublicKey,
  expectedAmount: BigNumber,
  recipient: PublicKey
) {
  // Find transaction by reference
  const signatureInfo = await findReference(connection, reference);
  
  // Validate the transfer
  const response = await validateTransfer(
    connection,
    signatureInfo.signature,
    {
      recipient,
      amount: expectedAmount,
      reference,
    }
  );
  
  return response;
}
```

## Kora (Gasless / Fee Abstraction)

Consider Kora when you need:
- sponsored transactions (user doesn't pay gas)
- users paying fees in tokens other than SOL
- a trusted signing / paymaster component

### Gasless Pattern

```typescript
import { Kora } from '@kora/sdk';

const kora = new Kora({
  apiKey: process.env.KORA_API_KEY,
  network: 'mainnet',
});

// Create sponsored transaction
async function sendGasless(
  userWallet: PublicKey,
  instructions: TransactionInstruction[]
) {
  // Build transaction without user paying fees
  const { transaction, sponsorSignature } = await kora.sponsorTransaction({
    feePayer: kora.sponsorWallet,
    instructions,
  });

  // User only signs for their accounts
  const userSignedTx = await wallet.signTransaction(transaction);

  // Submit with sponsor's fee payment
  const signature = await kora.sendTransaction(userSignedTx);

  return signature;
}
```

### Fee Token Abstraction

```typescript
// User pays fees in USDC instead of SOL
const { transaction } = await kora.createFeeAbstractedTransaction({
  feeToken: USDC_MINT,
  feeAmount: new BigNumber(0.01), // 0.01 USDC
  instructions,
  payer: userWallet,
});
```

## UX and Security Checklist for Payments

### Before Transaction
- [ ] Show recipient address clearly (truncated with copy)
- [ ] Show exact amount and token symbol
- [ ] Show estimated fees
- [ ] Display network (devnet vs mainnet)
- [ ] Require explicit confirmation

### Transaction Flow
- [ ] Show signing request in wallet
- [ ] Display pending state after submission
- [ ] Track confirmation status (processed → confirmed → finalized)
- [ ] Provide explorer link immediately

### Error Handling
- [ ] Insufficient balance → Clear message with needed amount
- [ ] User rejected → Graceful cancellation
- [ ] Network congestion → Retry option with priority fee
- [ ] Blockhash expired → Auto-retry with new blockhash
- [ ] Program error → Parse and display meaningful message

### Security
- [ ] Protect against replay (unique references)
- [ ] Confirm settlement via chain query, not callbacks
- [ ] Handle partial failures (sent but not confirmed)
- [ ] Validate amounts server-side for e-commerce

## Checkout Flow Example

```typescript
import { useState } from 'react';
import { useConnection, useWallet } from '@solana/wallet-adapter-react';

export function CheckoutButton({ amount, recipient, onSuccess }) {
  const { connection } = useConnection();
  const { publicKey, sendTransaction } = useWallet();
  const [status, setStatus] = useState<'idle' | 'signing' | 'confirming' | 'success' | 'error'>('idle');

  async function handlePayment() {
    if (!publicKey) return;

    try {
      setStatus('signing');
      
      // Create transfer instruction
      const transaction = new Transaction().add(
        SystemProgram.transfer({
          fromPubkey: publicKey,
          toPubkey: recipient,
          lamports: amount * LAMPORTS_PER_SOL,
        })
      );

      // Get fresh blockhash
      const { blockhash, lastValidBlockHeight } = await connection.getLatestBlockhash();
      transaction.recentBlockhash = blockhash;
      transaction.feePayer = publicKey;

      // Send and wait for confirmation
      setStatus('confirming');
      const signature = await sendTransaction(transaction, connection);
      
      await connection.confirmTransaction({
        signature,
        blockhash,
        lastValidBlockHeight,
      }, 'confirmed');

      setStatus('success');
      onSuccess(signature);
    } catch (error) {
      setStatus('error');
      console.error(error);
    }
  }

  return (
    <button 
      onClick={handlePayment}
      disabled={status === 'signing' || status === 'confirming'}
    >
      {status === 'idle' && `Pay ${amount} SOL`}
      {status === 'signing' && 'Confirm in wallet...'}
      {status === 'confirming' && 'Confirming...'}
      {status === 'success' && 'Payment complete!'}
      {status === 'error' && 'Failed - Try again'}
    </button>
  );
}
```

## Resources

- [Commerce Kit Repository](https://github.com/solana-foundation/commerce-kit)
- [Commerce Kit Documentation](https://commercekit.solana.com/)
- [Solana Pay Specification](https://github.com/solana-labs/solana-pay)
- [Kora Documentation](https://docs.kora.network/)
