---
name: react-sdk-patterns
description: Complete guide to building Miden frontends with @miden-sdk/react hooks. Covers MidenProvider setup, all query hooks (useAccounts, useAccount, useNotes, useSyncState, useAssetMetadata), all mutation hooks (useCreateWallet, useSend, useMultiSend, useMint, useConsume, useSwap, useTransaction, useCreateFaucet), transaction stages, signer integration, and utility functions. Use when writing, editing, or reviewing Miden React frontend code.
---

# Miden React SDK Patterns

## SDK Choice

ALWAYS use `@miden-sdk/react` hooks. Only fall back to the raw `WasmWebClient` (exported as `WebClient`) via `useMidenClient()` for operations not covered by hooks. The React SDK handles WASM safety (runExclusive), state management (Zustand), auto-sync, and transaction stage tracking automatically.

## MidenProvider Configuration

```tsx
import { MidenProvider } from "@miden-sdk/react";

<MidenProvider
  config={{
    rpcUrl: "testnet",          // "devnet" | "testnet" | "localhost" | "local" | custom URL
    prover: "testnet",          // "local" | "localhost" | "devnet" | "testnet" | custom URL | { primary, fallback }
    autoSyncInterval: 15000,    // ms, set to 0 to disable. Default: 15000
    noteTransportUrl: "...",    // optional: for private note delivery
  }}
  loadingComponent={<Loading />}  // shown during WASM init
  errorComponent={(error) => <Error error={error} />}  // function form receives the Error; a static element does not
>
  <App />
</MidenProvider>
```

| Network | rpcUrl | Use When |
|---------|--------|----------|
| Testnet | `"testnet"` | Recommended for new projects — primary development network |
| Devnet | `"devnet"` | Early-access testing (may lag feature parity with testnet) |
| Localhost | `"localhost"` | Local node at `http://localhost:57291` |

## Query Hooks

Each returns its own result shape plus `isLoading`, `error`, `refetch`.

### useAccounts()
```tsx
const { accounts, wallets, faucets, isLoading, error, refetch } = useAccounts();
// accounts — AccountHeader[] (every tracked account)
// wallets  — mirrors `accounts` (faucet-vs-wallet is not encoded in the account id)
// faucets  — always `[]`
```

An account's faucet-vs-wallet kind is not encoded in the account id, so `wallets` mirrors `accounts` and `faucets` is always empty. Use `accounts` and detect faucets **per-account** via `account.isFaucet()` (load the full `Account` with `useAccount`).

### useAccount(accountId: string)
```tsx
const { account, assets, getBalance, isLoading, error, refetch } = useAccount(accountId);
// account — Account object (.id(), .nonce(), .bech32id(), .isFaucet())
// assets — AssetBalance[] (assetId, amount, symbol?, decimals?)
// getBalance(faucetId) — bigint balance for specific token
```

`account.id()` and `account.nonce()` are methods (call them, then `.toString()` to render). `bech32id()` is installed on the `Account` prototype by the React SDK.

### useNotes(filter?)
```tsx
const { notes, consumableNotes, noteSummaries, consumableNoteSummaries, isLoading, error, refetch } = useNotes();
// notes — InputNoteRecord[] (filtered ONLY by `status`)
// consumableNotes — ConsumableNoteRecord[] (filtered ONLY by `accountId`)
// noteSummaries — NoteSummary[] (id, assets, sender) — also filtered by `sender` and `excludeIds`
// consumableNoteSummaries — NoteSummary[] — also filtered by `sender` and `excludeIds`

// Each filter option only narrows specific fields — destructure the one it affects:

// `status` filters the returned `notes` (the only option that does):
const { notes } = useNotes({ status: "committed" });  // "all" | "consumed" | "committed" | "expected" | "processing"
// `accountId` filters `consumableNotes` (NOT `notes`):
const { consumableNotes } = useNotes({ accountId: "0x..." });
// `sender` filters only the summary arrays (NOT `notes`/`consumableNotes`):
const { noteSummaries, consumableNoteSummaries } = useNotes({ sender: "0x..." });
// `excludeIds` filters only the summary arrays:
const { noteSummaries, consumableNoteSummaries } = useNotes({ excludeIds: ["0xnote1", "0xnote2"] });
```

### useNoteStream(options?)
```tsx
const { notes, latest, markHandled, markAllHandled, snapshot, isLoading, error } = useNoteStream();
// notes — StreamedNote[] (matching filter criteria)
// latest — most recent StreamedNote (convenience)
// markHandled(noteId) — exclude a note from future renders
// markAllHandled() — exclude all current notes
// snapshot() — capture { ids, timestamp } for cross-phase filtering

// Options:
const { notes } = useNoteStream({ status: "committed", sender: "0x..." });
const { notes } = useNoteStream({ since: Date.now() - 60000 }); // last 60s
const { notes } = useNoteStream({ excludeIds: new Set(["0xnote1"]) });
const { notes } = useNoteStream({ amountFilter: (amount) => amount > 100n });
```

### useSyncState()
```tsx
const { syncHeight, isSyncing, lastSyncTime, sync, error } = useSyncState();
await sync(); // Manual sync
```

### useAssetMetadata(assetIds?: string[])
```tsx
const { assetMetadata } = useAssetMetadata([faucetId]); // takes a string[] (NOT a bare string)
// assetMetadata — Map<string, AssetMetadata>
// Each entry: { assetId, symbol?, decimals? }
const meta = assetMetadata.get(faucetId);
// meta.symbol — "TEST"
// meta.decimals — 8
```

Pass an array even for a single asset — the hook calls `.filter` on its argument, so a bare string throws a runtime `TypeError`.

### useTransactionHistory(options?)
```tsx
const { records, record, status, isLoading, error, refetch } = useTransactionHistory({ id: txId });
// status: "pending" | "committed" | "discarded" | null
```

## Mutation Hooks

Each returns its own action function plus `error` and `reset`. The two families differ in their loading/progress fields:
- **Transaction hooks** (`useSend`, `useMultiSend`, `useMint`, `useConsume`, `useSwap`, `useTransaction`) expose `isLoading` and `stage` (a `TransactionStage`).
- **Account create/import hooks** (`useCreateWallet`, `useCreateFaucet`, `useImportAccount`) expose `isCreating` (or `isImporting` for the latter) and have **no** `stage`.

**Transaction stages**: `"idle"` → `"executing"` → `"proving"` → `"submitting"` → `"complete"`

Import `AuthScheme` for the `authScheme` option (re-exported from the package root):

```tsx
import { AuthScheme } from "@miden-sdk/react";
// AuthScheme.AuthRpoFalcon512 = 2 (default)
// AuthScheme.AuthEcdsaK256Keccak = 1
```

### useCreateWallet()
```tsx
const { createWallet, wallet, isCreating, error, reset } = useCreateWallet();
const account = await createWallet({
  storageMode: "private",                   // "private" | "public". Default: "private"
  authScheme: AuthScheme.AuthRpoFalcon512,  // Default: AuthScheme.AuthRpoFalcon512
  initSeed: seedBytes,                       // optional: Uint8Array for a deterministic account id
});
```

### useCreateFaucet()
```tsx
const { createFaucet, faucet, isCreating, error, reset } = useCreateFaucet();
const account = await createFaucet({
  tokenSymbol: "TEST",
  tokenName: "Test Token",                  // optional: defaults to tokenSymbol
  decimals: 8,                              // Default: 8
  maxSupply: 1000000n,                      // bigint | number
  storageMode: "private",                   // "private" | "public". Default: "private"
  authScheme: AuthScheme.AuthRpoFalcon512,  // Default: AuthScheme.AuthRpoFalcon512
});
```

### useImportAccount()
```tsx
const { importAccount, account, isImporting, error, reset } = useImportAccount();

// Import by account ID (network lookup):
const account = await importAccount({ type: "id", accountId: "0x..." });

// Import from file:
const account = await importAccount({ type: "file", file: accountFileOrBytes });

// Import from seed:
const account = await importAccount({
  type: "seed",
  seed: seedBytes,
  authScheme: AuthScheme.AuthRpoFalcon512,  // optional. Default: AuthScheme.AuthRpoFalcon512
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
  sendAll: true,           // optional: send entire balance (ignores amount)
  attachment: [1n, 2n],    // optional: arbitrary data attached to the note
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
    { to: recipient2, amount: 300n, noteType: "public" },          // per-recipient override
    { to: recipient3, amount: 200n, attachment: [1n, 2n, 3n] },    // per-recipient attachment
  ],
  noteType: "private",     // default for all recipients
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
  notes: [noteId1, noteId2],   // accepts: hex string IDs, NoteId, InputNoteRecord, or Note
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
await waitForCommit(result.txId, {  // useSend returns { txId, note }; other hooks use { transactionId }
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

### useSessionAccount(options)
```tsx
const { initialize, sessionAccountId, isReady, step, error, reset } = useSessionAccount({
  fund: async (sessionId) => {
    // Called after session wallet is created — fund it here
    await send({ from: mainWallet, to: sessionId, assetId: faucetId, amount: 100n });
  },
  assetId: faucetId,              // optional: for note filtering
  walletOptions: {                // optional: session wallet creation options
    storageMode: "private",                   // "private" | "public"
    authScheme: AuthScheme.AuthRpoFalcon512,
  },
  pollIntervalMs: 3000,           // optional: funding detection interval. Default: 3000
});
// Steps: "idle" → "creating" → "funding" → "consuming" → "ready"
// Call initialize() to start the flow. isReady becomes true when fully funded.
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
- `ParaSignerProvider` from `@miden-sdk/use-miden-para-react` — EVM wallets
- `TurnkeySignerProvider` from `@miden-sdk/miden-turnkey-react` — passkey auth
- `MidenFiSignerProvider` from `@miden-sdk/miden-wallet-adapter-react` — MidenFi wallet

These three packages live in external repos (not in web-sdk), so confirm the exact published names against the current Para/Turnkey/MidenFi integration docs before installing. The v0.15 example app (`packages/react-sdk/examples/wallet/src/main.tsx`) imports them as above; some web-sdk docs alias the Para package as `@miden-sdk/para`.

```tsx
// Example: Para signer wrapping MidenProvider
import { ParaSignerProvider } from "@miden-sdk/use-miden-para-react";
<ParaSignerProvider apiKey="..." environment="BETA">
  <MidenProvider config={...}><App /></MidenProvider>
</ParaSignerProvider>
```

### useSigner() — Unified Interface
Returns `SignerContextValue | null` — `null` in local-keystore mode (no signer provider mounted). Guard before destructuring.
```tsx
const signer = useSigner();
if (!signer) return null; // local keystore mode
const { isConnected, connect, disconnect, name } = signer;
```

### Custom Signer
Implement `SignerContextValue` interface via `SignerContext.Provider`. Requires: `name`, `storeName` (unique per user for DB isolation), `accountConfig`, `signCb`, `isConnected`, `connect`, `disconnect`. See `frontend-source-guide` skill for source references.

## Utility Functions

```tsx
import { formatAssetAmount, parseAssetAmount, getNoteSummary, formatNoteSummary, toBech32AccountId } from "@miden-sdk/react";

formatAssetAmount(1000000n, 8)       // "0.01"
parseAssetAmount("0.01", 8)           // 1000000n
const summary = getNoteSummary(note); // { id, assets, sender }
formatNoteSummary(summary);           // "1.5 TEST from mtst1..." (the " from <sender>" suffix is appended whenever the summary has a sender)
toBech32AccountId("0x1234...");       // "mtst1..." (testnet HRP; defaults to testnet)
```

The HRP is inferred from the configured `rpcUrl` and defaults to testnet: mainnet=`mm`, testnet=`mtst` (default), devnet=`mdev` — there is no `miden` HRP.

## Direct Client Access

```tsx
const client = useMidenClient(); // throws if not ready
const { runExclusive } = useMiden();

// For operations not covered by hooks (use methods on the WebClient itself —
// e.g. getSyncHeight, getAccount, getTransactions; getBlockHeaderByNumber lives on RpcClient, not here):
await runExclusive(async () => {
  const height = await client.getSyncHeight();
});
```

## Type Imports

```tsx
import { AuthScheme } from "@miden-sdk/react"; // value (enum), not just a type

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
