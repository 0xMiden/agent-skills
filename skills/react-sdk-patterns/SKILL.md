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

All return `{ data, isLoading, error, refetch }`.

### useAccounts()
```tsx
const { data: accounts } = useAccounts();
// accounts.wallets — regular wallet accounts
// accounts.faucets — token faucet accounts
// accounts.all — everything (AccountHeader[])
```

### useAccount(accountId: string)
```tsx
const { data: account } = useAccount(accountId);
// account.account — Account object (.id, .nonce, .bech32id())
// account.assets — AssetBalance[] (assetId, amount, symbol?, decimals?)
// account.getBalance(faucetId) — bigint balance for specific token
```

### useNotes(filter?)
```tsx
const { data: notes } = useNotes();
// notes.notes — InputNoteRecord[]
// notes.consumableNotes — ConsumableNoteRecord[]
// notes.noteSummaries — NoteSummary[] (id, assets, sender)
// notes.consumableNoteSummaries — NoteSummary[]

// Filter by account:
const { data } = useNotes({ accountId: "0x..." });
// Filter by status:
const { data } = useNotes({ status: "committed" });
```

### useSyncState()
```tsx
const { syncHeight, isSyncing, lastSyncTime, sync, error } = useSyncState();
await sync(); // Manual sync
```

### useAssetMetadata(faucetId: string | string[])
```tsx
const { data: metadata } = useAssetMetadata(faucetId);
// metadata.symbol — "TEST"
// metadata.decimals — 8
```

### useTransactionHistory(options?)
```tsx
const { records, record, status, isLoading } = useTransactionHistory({ id: txId });
// status: "pending" | "committed" | "discarded" | null
```

## Mutation Hooks

All return `{ mutate, data, isLoading, stage, error, reset }`.

**Transaction stages**: `"idle"` → `"executing"` → `"proving"` → `"submitting"` → `"complete"`

### useCreateWallet()
```tsx
const { mutate: createWallet, isLoading } = useCreateWallet();
const account = await createWallet({
  storageMode: "private",  // "private" | "public" | "network". Default: "private"
  mutable: true,           // Default: true
  authScheme: 0,           // 0 = RpoFalcon512, 1 = EcdsaK256Keccak. Default: 0
});
```

### useCreateFaucet()
```tsx
const { mutate: createFaucet } = useCreateFaucet();
const faucet = await createFaucet({
  tokenSymbol: "TEST",
  decimals: 8,             // Default: 8
  maxSupply: 1000000n,     // bigint!
  storageMode: "public",   // Default: "private"
});
```

### useSend()
```tsx
const { mutate: send, stage } = useSend();
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
const { mutate: multiSend } = useMultiSend();
await multiSend({
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
const { mutate: mint } = useMint();
await mint({
  targetAccountId: recipientId,
  faucetId: myFaucetId,
  amount: 10000n,         // bigint!
  noteType: "public",
});
```

### useConsume()
```tsx
const { mutate: consume } = useConsume();
await consume({
  accountId: myAccountId,
  noteIds: [noteId1, noteId2],
});
```

### useSwap()
```tsx
const { mutate: swap } = useSwap();
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
const { mutate: execute } = useTransaction();

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
const { mutate: waitForCommit } = useWaitForCommit();
await waitForCommit({
  transactionId: result.transactionId,
  timeoutMs: 10000,   // Default: 10000
  intervalMs: 1000,    // Default: 1000
});
```

### useWaitForNotes()
```tsx
const { mutate: waitForNotes } = useWaitForNotes();
await waitForNotes({
  accountId: myAccountId,
  minCount: 1,         // Default: 1
  timeoutMs: 10000,
});
```

## Transaction Progress UI

```tsx
function SendButton({ from, to, assetId, amount }) {
  const { mutate: send, stage, isLoading, error } = useSend();

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
