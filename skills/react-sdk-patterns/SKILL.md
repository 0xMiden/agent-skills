---
name: react-sdk-patterns
description: Complete guide to building Miden frontends with @miden-sdk/react hooks. Covers MidenProvider setup, all query hooks (useAccounts, useAccount, useNotes, useSyncState, useAssetMetadata), all mutation hooks (useCreateWallet, useSend, useMultiSend, useMint, useConsume, useSwap, useTransaction, useCreateFaucet), transaction stages, signer integration, and utility functions. Use when writing, editing, or reviewing Miden React frontend code.
---

# Miden React SDK Patterns

## SDK Choice

ALWAYS use `@miden-sdk/react` hooks. Only fall back to raw `@miden-sdk/miden-sdk` WebClient via `useMidenClient()` for operations not covered by hooks. The React SDK handles WASM safety (runExclusive), state management (Zustand), auto-sync, and transaction stage tracking automatically.

## MidenProvider Configuration

```tsx
import { MidenProvider } from "@miden-sdk/react";

<MidenProvider
  config={{
    rpcUrl: "devnet",           // "devnet" | "testnet" | "localhost" | custom URL
    prover: "devnet",           // "local" | "devnet" | "testnet" | custom URL
    autoSyncInterval: 15000,    // ms, set to 0 to disable. Default: 15000
    noteTransportUrl: "...",    // optional: for private note delivery
  }}
  loadingComponent={<Loading />}  // shown during WASM init
  errorComponent={<Error />}      // shown on init failure (receives Error prop)
>
  <App />
</MidenProvider>
```

| Network | rpcUrl | Use When |
|---------|--------|----------|
| Devnet | `"devnet"` | Development, testing with fake tokens |
| Testnet | `"testnet"` | Pre-production testing |
| Localhost | `"localhost"` | Local node at `http://localhost:57291` |

## Query Hooks

Each returns its own result shape plus `isLoading`, `error`, `refetch`.

### useAccounts()
```tsx
const { accounts, wallets, faucets, isLoading, error, refetch } = useAccounts();
// wallets — AccountHeader[] (regular wallet accounts)
// faucets — AccountHeader[] (token faucet accounts)
// accounts — AccountHeader[] (everything)
```

### useAccount(accountId: string)
```tsx
const { account, assets, getBalance, isLoading, error, refetch } = useAccount(accountId);
// account — Account object (.id, .nonce, .bech32id())
// assets — AssetBalance[] (assetId, amount, symbol?, decimals?)
// getBalance(faucetId) — bigint balance for specific token
```

### useNotes(filter?)
```tsx
const { notes, consumableNotes, noteSummaries, consumableNoteSummaries, isLoading, error, refetch } = useNotes();
// notes — InputNoteRecord[]
// consumableNotes — ConsumableNoteRecord[]
// noteSummaries — NoteSummary[] (id, assets, sender)
// consumableNoteSummaries — NoteSummary[]

// Filter by account:
const { notes } = useNotes({ accountId: "0x..." });
// Filter by status:
const { notes } = useNotes({ status: "committed" });
```

### useSyncState()
```tsx
const { syncHeight, isSyncing, lastSyncTime, sync, error } = useSyncState();
await sync(); // Manual sync
```

### useAssetMetadata(faucetId: string | string[])
```tsx
const { assetMetadata } = useAssetMetadata(faucetId);
// assetMetadata — Map<string, AssetMetadata>
// Each entry: { assetId, symbol?, decimals? }
const meta = assetMetadata.get(faucetId);
// meta.symbol — "TEST"
// meta.decimals — 8
```

### useTransactionHistory(options?)
```tsx
const { records, record, status, isLoading, error, refetch } = useTransactionHistory({ id: txId });
// status: "pending" | "committed" | "discarded" | null
```

## Mutation Hooks

Each returns its own action function plus `isLoading`, `stage`, `error`, `reset`.

**Transaction stages**: `"idle"` → `"executing"` → `"proving"` → `"submitting"` → `"complete"`

### useCreateWallet()
```tsx
const { createWallet, wallet, isCreating, error, reset } = useCreateWallet();
const account = await createWallet({
  storageMode: "private",  // "private" | "public" | "network". Default: "private"
  mutable: true,           // Default: true
  authScheme: 0,           // 0 = RpoFalcon512, 1 = EcdsaK256Keccak. Default: 0
});
```

### useCreateFaucet()
```tsx
const { createFaucet, faucet, isCreating, error, reset } = useCreateFaucet();
const account = await createFaucet({
  tokenSymbol: "TEST",
  decimals: 8,             // Default: 8
  maxSupply: 1000000n,     // bigint!
  storageMode: "public",   // Default: "private"
});
```

### useSend()
```tsx
const { send, result, isLoading, stage, error, reset } = useSend();
await send({
  from: senderAccountId,
  to: recipientAccountId,
  assetId: faucetId,       // token faucet ID
  amount: 1000n,           // bigint!
  noteType: "private",     // "private" | "public". Default: "private"
  recallHeight: 100,       // optional: sender can reclaim after this block
  timelockHeight: 50,      // optional: recipient can consume after this block
});
```

### useMultiSend()
```tsx
const { sendMany, result, isLoading, stage, error, reset } = useMultiSend();
await sendMany({
  from: senderAccountId,
  assetId: faucetId,
  recipients: [
    { to: recipient1, amount: 500n },
    { to: recipient2, amount: 300n },
  ],
  noteType: "private",
});
```

### useMint()
```tsx
const { mint, result, isLoading, stage, error, reset } = useMint();
await mint({
  targetAccountId: recipientId,
  faucetId: myFaucetId,
  amount: 10000n,         // bigint!
  noteType: "public",
});
```

### useConsume()
```tsx
const { consume, result, isLoading, stage, error, reset } = useConsume();
await consume({
  accountId: myAccountId,
  noteIds: [noteId1, noteId2],
});
```

### useSwap()
```tsx
const { swap, result, isLoading, stage, error, reset } = useSwap();
await swap({
  accountId: myAccountId,
  offeredFaucetId: tokenA,
  offeredAmount: 100n,
  requestedFaucetId: tokenB,
  requestedAmount: 50n,
  noteType: "private",
  paybackNoteType: "private",
});
```

### useTransaction() — Escape Hatch
```tsx
const { execute, result, isLoading, stage, error, reset } = useTransaction();

// With pre-built TransactionRequest:
await execute({ accountId, request: txRequest });

// With factory function (gets access to client):
await execute({
  accountId,
  request: (client) => client.newSwapTransactionRequest(/* ... */),
});
```

### useWaitForCommit()
```tsx
const { waitForCommit } = useWaitForCommit();
await waitForCommit(result.transactionId, {
  timeoutMs: 10000,   // Default: 10000
  intervalMs: 1000,    // Default: 1000
});
```

### useWaitForNotes()
```tsx
const { waitForConsumableNotes } = useWaitForNotes();
await waitForConsumableNotes({
  accountId: myAccountId,
  minCount: 1,         // Default: 1
  timeoutMs: 10000,
});
```

## Transaction Progress UI

```tsx
function SendButton({ from, to, assetId, amount }) {
  const { send, stage, isLoading, error } = useSend();

  return (
    <div>
      <button onClick={() => send({ from, to, assetId, amount })} disabled={isLoading}>
        {isLoading ? `${stage}...` : "Send"}
      </button>
      {error && <p>Error: {error.message}</p>}
    </div>
  );
}
```

## Signer Integration

### Local Keystore (Default)
No signer provider needed. Keys are managed in the browser via IndexedDB.

### External Signers
Wrap MidenProvider with a signer provider. Three pre-built options:
- `ParaSignerProvider` from `@miden-sdk/para` — EVM wallets
- `TurnkeySignerProvider` from `@miden-sdk/miden-turnkey-react` — passkey auth
- `MidenFiSignerProvider` from `@miden-sdk/wallet-adapter-react` — MidenFi wallet

```tsx
// Example: Para signer wrapping MidenProvider
import { ParaSignerProvider } from "@miden-sdk/para";
<ParaSignerProvider apiKey="..." environment="PRODUCTION">
  <MidenProvider config={...}><App /></MidenProvider>
</ParaSignerProvider>
```

### useSigner() — Unified Interface
```tsx
const { isConnected, connect, disconnect, name } = useSigner();
```

### Custom Signer
Implement `SignerContextValue` interface via `SignerContext.Provider`. Requires: `name`, `storeName` (unique per user for DB isolation), `accountConfig`, `signCb`, `connect`, `disconnect`. See `frontend-source-guide` skill for source references.

## Utility Functions

```tsx
import { formatAssetAmount, parseAssetAmount, getNoteSummary, formatNoteSummary, toBech32AccountId } from "@miden-sdk/react";

formatAssetAmount(1000000n, 8)       // "0.01"
parseAssetAmount("0.01", 8)           // 1000000n
const summary = getNoteSummary(note); // { id, assets, sender }
formatNoteSummary(summary);           // "1.5 TEST"
toBech32AccountId("0x1234...");       // "miden1qy35..."
```

## Direct Client Access

```tsx
const client = useMidenClient(); // throws if not ready
const { runExclusive } = useMiden();

// For operations not covered by hooks:
await runExclusive(async (client) => {
  const header = await client.getBlockHeaderByNumber(100);
});
```

## Type Imports

```tsx
import type {
  MidenConfig, QueryResult, MutationResult, TransactionStage,
  AccountsResult, AccountResult, AssetBalance, NotesResult, NoteSummary,
  SendOptions, MultiSendOptions, MintOptions, ConsumeOptions, SwapOptions,
  CreateWalletOptions, CreateFaucetOptions, ExecuteTransactionOptions,
  TransactionResult, SyncState, WaitForCommitOptions, WaitForNotesOptions,
  Account, AccountId, InputNoteRecord, ConsumableNoteRecord,
  TransactionRecord, TransactionRequest, NoteType, AccountStorageMode,
  SignerContextValue, SignCallback, SignerAccountConfig,
} from "@miden-sdk/react";
```
