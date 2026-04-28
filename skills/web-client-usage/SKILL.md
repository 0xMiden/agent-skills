---
name: web-client-usage
description: Conventions for writing JavaScript/TypeScript code that uses the Miden web SDK (`@miden-sdk/miden-sdk`). Use when building apps on Miden, writing integration tests, or calling MidenClient methods — covers initialization, the resource-based API (accounts, transactions, notes, keystore, compile), sync ordering, type conversions, transaction flows, custom contracts, private note transport, and pitfalls.
---

# Web SDK Usage Patterns

This skill targets the `@miden-sdk/miden-sdk` npm package shipped from
[`0xMiden/miden-client`](https://github.com/0xMiden/miden-client). For
React-hook usage, prefer the `react-sdk-patterns` skill — only fall through
to the raw client when a hook does not cover what you need.

## API Overview

The SDK exposes a top-level `MidenClient` whose state is split across typed
**resources**:

| Resource | What it covers |
|----------|----------------|
| `client.accounts` | Wallets, faucets, custom contracts, listing, import/export |
| `client.transactions` | `send` / `mint` / `consume` / `consumeAll` / `swap` / `execute` / `preview` / `waitFor` |
| `client.notes` | Listing, fetching, importing/exporting, private-note transport |
| `client.tags` | Note-tag subscriptions |
| `client.settings` | Persistent client settings |
| `client.compile` | Compiling MASM into account components, tx scripts, note scripts |
| `client.keystore` | Inserting / fetching / removing secret keys |

`MidenClient` is the public surface. The underlying WASM-bound class is still
exported as `WasmWebClient` (alias for the legacy `WebClient`) for low-level
operations the resource API does not yet wrap — reach for it via the wrapped
`#inner` only when you must.

## Client Initialization

### Convenience constructors (recommended)

```typescript
import { MidenClient } from "@miden-sdk/miden-sdk";

// Testnet — autoSync on, testnet RPC + prover + note transport
const client = await MidenClient.createTestnet();

// Devnet equivalent
const client = await MidenClient.createDevnet();
```

Both accept the same `ClientOptions` for overrides:

```typescript
const client = await MidenClient.createTestnet({
  storeName: "my-app-tests",      // isolates the IndexedDB store
  proverUrl: "local",             // prove locally instead of remote
  autoSync: false,                // disable initial sync
});
```

### Generic constructor

```typescript
const client = await MidenClient.create({
  rpcUrl: "https://rpc.testnet.miden.io",  // string URL or "testnet"/"devnet"/"localhost"
  noteTransportUrl: "https://ntx.testnet.miden.io",
  storeName: "my-store",
  seed: new Uint8Array(32),         // optional — deterministic key generation
  proverUrl: "testnet",             // optional — sets a default prover
  autoSync: true,                   // optional — call sync() after init
  keystore: {                       // optional — external HSM/keystore
    getKey: async (pubKey) => { /* return secretKey or null */ },
    insertKey: async (pubKey, secretKey) => { /* persist */ },
    sign: async (pubKey, signingInputs) => { /* return signature */ },
  },
});
```

If `rpcUrl` is omitted, `create()` delegates to `createTestnet()`.

### Lazy / SSR-safe init

Some bundles (Next.js, Capacitor, raw `/lazy` entry) cannot await WASM at
import time. Use `MidenClient.ready()` to wait for WASM in-band — it is
idempotent and shared across callers:

```typescript
await MidenClient.ready();
const client = await MidenClient.createTestnet();
```

### Termination

```typescript
client.terminate();   // free WASM resources, close the store handle
```

After `terminate()`, every method throws — guard against late callbacks on
unmount.

## Sync — Always Sync First

The client's view of the chain is only as fresh as its last sync. **Always
call `sync()` before reading account state or building a transaction that
depends on freshly received notes.**

```typescript
const summary = await client.sync();          // returns SyncSummary
const height  = await client.getSyncHeight(); // current local block number
```

Common patterns:

- Sync before consuming notes (notes must be committed on-chain)
- Sync after submitting a transaction to observe the result
- Pass `waitForConfirmation: true` to a `transactions.send/mint/consume/swap`
  call to let the SDK wait for the tx commit instead of polling manually
- Use `client.waitForIdle()` to flush all queued WASM calls before doing a
  side-effect that must not race with a kernel callback (e.g. clearing an
  in-memory unlock token after a wallet "lock")

`autoSync: true` (default for `createTestnet`/`createDevnet`) only triggers a
single sync at construction time — it is not a polling loop. Use the React
SDK's `useSyncState` or `MidenProvider` `autoSyncInterval` for periodic sync.

## Type Conversions

Type confusion across the WASM boundary is the leading source of bugs.

### `AccountId`

```typescript
const id = AccountId.fromHex("0xabc123...");          // throws on invalid hex
const id = Address.fromBech32("mtst1abc...").accountId();
const hex = id.toString(); // "0x..."
```

Pass `AccountId` (or any account ref the resource accepts: `string` hex,
`Address`, `Account`, `AccountHeader`) to resource methods — never raw
strings to methods that ask for `AccountId` directly.

`AccountId.fromHex` throws on malformed input; wrap in `try/catch` when
accepting user input.

### Amounts — Always `BigInt`

```typescript
BigInt(1000)
1000n           // numeric literal
BigInt("1000")
```

All token amounts in the SDK are `bigint`. Mixing `number` causes silent
precision loss above `Number.MAX_SAFE_INTEGER` and `TypeError` below.

### Visibility & Account Types

```typescript
import { NoteVisibility, AccountType, AuthScheme, StorageMode } from "@miden-sdk/miden-sdk";

NoteVisibility.Public  // "public"
NoteVisibility.Private // "private"

AccountType.MutableWallet
AccountType.ImmutableWallet
AccountType.FungibleFaucet
AccountType.NonFungibleFaucet
AccountType.MutableContract
AccountType.ImmutableContract

AuthScheme.Falcon      // default — Falcon-512 over Poseidon2
AuthScheme.ECDSA       // EcdsaK256Keccak

StorageMode.Public
StorageMode.Private
StorageMode.Network
```

Use `NoteVisibility` (not the legacy `NoteType` enum) and `AuthScheme.Falcon`
for the Poseidon2-based Falcon-512 scheme.

## Account Creation

```typescript
// Wallet — defaults: mutable, private, Falcon
const wallet = await client.accounts.create();

// Wallet with explicit options
const wallet = await client.accounts.create({
  type: AccountType.MutableWallet,
  storage: "private",
  auth: AuthScheme.Falcon,
});

// Faucet
const faucet = await client.accounts.create({
  type: AccountType.FungibleFaucet,
  storage: "public",
  symbol: "DAG",
  decimals: 8,
  maxSupply: 10_000_000n,
});

// Custom contract — requires seed and AuthSecretKey
const component = await client.compile.component({ code: contractMasm, slots: [] });
const contract = await client.accounts.create({
  type: AccountType.MutableContract,
  seed: new Uint8Array(32),
  auth: secretKey,            // AuthSecretKey, not the AuthScheme enum
  components: [component],
});
```

## Transactions

The transactions API is option-bag-based and accepts any account ref
(`Account`, `AccountHeader`, hex string, `AccountId`).

### Send

```typescript
const { txId } = await client.transactions.send({
  account: wallet,                 // sender
  to: "0xrecipient...",            // any account ref
  token: faucet,                   // faucet account ref — identifies the asset
  amount: 100n,
  type: NoteVisibility.Public,     // optional, defaults to "private"
  reclaimAfter: 100,               // optional — sender can reclaim after this block
  timelockUntil: 50,               // optional — recipient can consume after this block
  waitForConfirmation: true,
  timeout: 30_000,
});
```

For private sends where you also need to deliver the note out-of-band, set
`returnNote: true` and the call returns the constructed `Note` object —
incompatible with `reclaimAfter`/`timelockUntil`.

```typescript
const { txId, note } = await client.transactions.send({
  account: wallet,
  to: recipientAddress,
  token: faucet,
  amount: 100n,
  type: NoteVisibility.Private,
  returnNote: true,
});

// Stream the note via the note-transport service
await client.notes.sendPrivate({ note, to: Address.fromBech32("mtst1...") });
```

### Mint

```typescript
const { txId } = await client.transactions.mint({
  account: faucet,                 // faucet executes the mint
  to: targetAccountId,             // recipient
  amount: 1000n,
  type: NoteVisibility.Public,
  waitForConfirmation: true,
});
```

The transaction executes on the **faucet** — a frequent bug is passing the
recipient as `account`.

### Consume

```typescript
// Specific notes
await client.transactions.consume({
  account: wallet,
  notes: [noteId1, noteRecord, "0xnote..."],   // any of: hex, NoteId, InputNoteRecord, Note
  waitForConfirmation: true,
});

// Drain everything consumable for the account
const { txId, consumed, remaining } = await client.transactions.consumeAll({
  account: wallet,
  maxNotes: 50,                    // optional cap
});
```

### Swap

```typescript
await client.transactions.swap({
  account: wallet,
  offered: { token: tokenA, amount: 100n },
  requested: { token: tokenB, amount: 50n },
  type: NoteVisibility.Public,            // swap-note visibility
  paybackType: NoteVisibility.Private,    // payback-note visibility
});
```

### Execute (custom scripts)

```typescript
const script = await client.compile.txScript({
  code: scriptMasm,
  libraries: [{ namespace: "my::lib", code: libMasm, linking: "dynamic" }],
});

await client.transactions.execute({
  account: contract,
  script,
  foreignAccounts: [
    publicAccountId,                    // public — auto-fetched via RPC
    { id: privateContractId, storage: storageRequirements },
  ],
  waitForConfirmation: true,
});
```

**Public foreign accounts are auto-fetched** during execution — only private
foreign accounts must be supplied with their storage requirements.

### Preview (dry run)

`transactions.preview({ operation: "send" | "mint" | "consume" | "swap" | "custom", ... })`
runs the same kernel as the real call but without proving or submitting,
returning a summary suitable for UI confirmation screens.

## Notes

```typescript
await client.notes.list();                            // all input notes
await client.notes.list({ status: "committed" });     // filter
await client.notes.get(noteId);                       // single record
await client.notes.listSent();                        // output notes
await client.notes.listAvailable({ account: wallet });// consumable for an account

// Import/export
await client.notes.import(noteFile);
const file = await client.notes.export(noteId);

// Private-note transport
await client.notes.fetchPrivate();                    // pulls anything addressed to tracked accounts
await client.notes.sendPrivate({ note, to: addr });   // delivers via the transport service
```

## Accounts (querying)

```typescript
await client.accounts.list();                         // tracked accounts
await client.accounts.get(ref);                       // single (returns null if not tracked)
await client.accounts.getOrImport(ref);               // tries get(), falls back to import()
await client.accounts.getDetails(ref);                // header + status + vault summary
await client.accounts.insert({ account, overwrite }); // start tracking an existing account
```

For balance reads, use `getDetails` or — if you need a single asset balance
without loading the full vault — drop into the underlying WASM client's
`accountReader(id)` lazy reader.

## Keystore

```typescript
await client.keystore.insert(accountId, secretKey);
await client.keystore.get(pubKeyCommitment);
await client.keystore.remove(pubKeyCommitment);
await client.keystore.getCommitments(accountId);
await client.keystore.getAccountId(pubKeyCommitment);
```

`keystore.insert` is the single call that both stores the key and registers
its commitment with the account.

## Compile

```typescript
await client.compile.component({ code, slots, supportAllTypes: true });
await client.compile.txScript({ code, libraries });
await client.compile.noteScript({ code, libraries });
```

Note scripts are **MASM libraries with a single `@note_script`-annotated
procedure**, not begin/end programs — `client.compile.noteScript` builds the
correct shape from a procedure body. The same applies to `@auth_script` for
authentication scripts.

## Common Workflows

### Mint and consume (fund a fresh wallet)

```typescript
const wallet = await client.accounts.create();
const faucet = await client.accounts.create({
  type: AccountType.FungibleFaucet,
  storage: "public",
  symbol: "TEST",
  decimals: 8,
  maxSupply: 1_000_000n,
});

await client.transactions.mint({
  account: faucet,
  to: wallet,
  amount: 10_000n,
  type: NoteVisibility.Public,
  waitForConfirmation: true,
});

await client.sync();
await client.transactions.consumeAll({
  account: wallet,
  waitForConfirmation: true,
});
```

### Wait for an external transfer

```typescript
await client.sync();
const before = (await client.notes.listAvailable({ account: wallet })).length;

while (true) {
  await new Promise(r => setTimeout(r, 3000));
  await client.sync();
  const now = (await client.notes.listAvailable({ account: wallet })).length;
  if (now > before) break;
}
```

## Common Pitfalls

1. **Forgetting to sync.** Notes won't appear, balances will be stale, foreign
   accounts will be at the wrong block.
2. **`number` instead of `BigInt`** for amounts — silent precision loss.
3. **Passing strings where the SDK expects an account ref** — pre-parse with
   `AccountId.fromHex()` (and catch its throw).
4. **Consuming notes before they're committed** — sync first, check status.
5. **Submitting `mint` with the recipient as `account`** — mint executes on
   the faucet account, not the target.
6. **Private notes without transport** — must call `notes.sendPrivate()` (or
   pass `returnNote: true` to `transactions.send` and deliver out-of-band).
7. **Holding WASM-owned objects across `terminate()`** — every `Account`,
   `Note`, `AccountId`, `NoteAndArgsArray` etc. owns Rust memory through the
   WASM ArrayBuffer. After `terminate()` they panic with "null pointer
   passed to rust" — drop references on unmount.
8. **Calling `accountReader(...)` in parallel with a write** — the readers
   share the WASM client. Wrap concurrent flows with `client.waitForIdle()`
   or rely on the React SDK's `runExclusive`.
