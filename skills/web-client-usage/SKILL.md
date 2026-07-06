---
name: web-client-usage
description: Conventions for writing JavaScript/TypeScript code that uses the Miden web SDK (`@miden-sdk/miden-sdk`). Use when building apps on Miden, writing integration tests, or calling MidenClient methods — covers initialization, the resource-based API (accounts, transactions, notes, keystore, compile), sync ordering, type conversions, transaction flows, custom contracts, private note transport, and pitfalls.
---

# Web SDK Usage Patterns

This skill targets the `@miden-sdk/miden-sdk` npm package published from
[`0xMiden/web-sdk`](https://github.com/0xMiden/web-sdk) (the JS web client; in
0.15 it builds on the `miden-client` Rust crate). For React-hook usage, prefer
the `react-sdk-patterns` skill — only fall through to the raw client when a hook
does not cover what you need.

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

`MidenClient` is the public surface. The underlying WASM-bound class is
exported as `WasmWebClient` (an alias for `WebClient`) for low-level
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
  noteTransportUrl: "https://transport.miden.io",
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

Pass `AccountId` (or any account ref the resource accepts: a hex/bech32
`string`, `Account`, `AccountHeader`, or `AccountId`) to resource methods —
never raw strings to methods that ask for `AccountId` directly. Note that an
`Address` object is **not** an account ref: `AccountRef = string | Account |
AccountHeader | AccountId`, and the resolver only special-cases objects with an
`.id()` method (`Address` exposes `accountId()`, not `id()`), so call
`address.accountId()` first.

`AccountId.fromHex` throws on malformed input; wrap in `try/catch` when
accepting user input.

### Amounts — Always `BigInt`

```typescript
BigInt(1000)
1000n           // numeric literal
BigInt("1000")
```

Amount fields accept `number | bigint` (`SendOptions`/`MintOptions.amount`,
`FaucetOptions.maxSupply`) and are coerced internally with `BigInt(...)`, so
an integer `number` works and does **not** throw. The hazard is pre-conversion
precision loss: a numeric literal above `Number.MAX_SAFE_INTEGER` (2^53) loses
precision before it ever reaches `BigInt()`. Use `bigint` for any value that
might exceed 2^53.

### Visibility & Account Types

```typescript
import { NoteVisibility, AccountType, AuthScheme, StorageMode } from "@miden-sdk/miden-sdk";

NoteVisibility.Public  // "public"
NoteVisibility.Private // "private"

// AccountType is a faucet-kind selector with ONLY two members:
AccountType.FungibleFaucet     // 0
AccountType.NonFungibleFaucet  // 1

AuthScheme.Falcon      // default — Falcon-512 over Poseidon2
AuthScheme.ECDSA       // EcdsaK256Keccak

StorageMode.Public
StorageMode.Private
```

Use `NoteVisibility` strings with the high-level resource APIs — `NoteType` is a
separate enum exported for the low-level WASM APIs and is easy to confuse with
`NoteVisibility`, so do not pass it where a `NoteVisibility` is expected. Use
`AuthScheme.Falcon` for the Poseidon2-based Falcon-512 scheme.

`AccountType` exposes **only** `FungibleFaucet`/`NonFungibleFaucet`. There is no
`MutableWallet`/`ImmutableWallet`/`MutableContract`/`ImmutableContract` member —
those evaluate to `undefined`. Wallets and contracts are not chosen via
`AccountType`: a wallet is the default (omit `type`), and a contract is any
`accounts.create()` call that passes `components` (or `type:
"MutableContract"`/`"ImmutableContract"` as strings). See "Account Creation".

`StorageMode` has only `Public`/`Private`. There is no `StorageMode.Network`
(accessing it yields `undefined`, which silently resolves to private).

## Account Creation

```typescript
// Wallet — the default when no `type` is given (private, Falcon)
const wallet = await client.accounts.create();

// Wallet with explicit options — omit `type` (there is no
// AccountType.*Wallet member; passing one would be undefined → default wallet)
const wallet = await client.accounts.create({
  storage: "private",
  auth: AuthScheme.Falcon,
});

// Faucet — selected via AccountType.FungibleFaucet / NonFungibleFaucet
const faucet = await client.accounts.create({
  type: AccountType.FungibleFaucet,
  storage: "public",
  symbol: "DAG",
  decimals: 8,
  maxSupply: 10_000_000n,
});

// Custom contract — selected by passing `components` (NOT by an AccountType
// member). Requires seed and an AuthSecretKey.
const component = await client.compile.component({ code: contractMasm, slots: [] });
const contract = await client.accounts.create({
  seed: new Uint8Array(32),
  auth: secretKey,            // AuthSecretKey, not the AuthScheme enum
  components: [component],     // presence of `components` routes to a contract
});
```

A contract is whatever `accounts.create()` call carries `components` — the
string forms `type: "MutableContract"` / `"ImmutableContract"` also route to a
contract, but the canonical selector is `components`. There is **no**
`AccountType.MutableContract`; `type: AccountType.MutableContract` is
`undefined` and, without `components`, would silently create a wallet.

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
  type: NoteVisibility.Public,     // optional, but defaults to "public" — see note below
  reclaimAfter: 100,               // optional — sender can reclaim after this block
  timelockUntil: 50,               // optional — recipient can consume after this block
  waitForConfirmation: true,
  timeout: 30_000,
});
```

**`type` defaults to PUBLIC, not private** — for both `send` and `mint`, the
note-type resolver treats an omitted/`undefined` `type` as
`NoteVisibility.Public`. Omitting `type` therefore creates a **public** note (a
privacy hazard). Always pass `type: NoteVisibility.Private` explicitly when a
private note is required.

For private sends where you also need to deliver the note out-of-band, set
`returnNote: true` and the call returns the constructed `Note` object —
incompatible with `reclaimAfter`/`timelockUntil`.

```typescript
const { txId, note } = await client.transactions.send({
  account: wallet,
  to: "mtst1...",                  // account ref: hex/bech32 string, Account,
                                   // AccountHeader, or AccountId (not an Address)
  token: faucet,
  amount: 100n,
  type: NoteVisibility.Private,
  returnNote: true,
});

// Stream the note via the note-transport service.
// `to` accepts a bech32 string, a 0x-hex string, an Account, or an AccountId
// (resolved via resolveAddress). It does NOT accept a pre-parsed Address
// object — that falls through to Address.fromAccountId(addr) and throws.
await client.notes.sendPrivate({ note, to: "mtst1..." });
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
  offer: { token: tokenA, amount: 100n },    // field is `offer`, not `offered`
  request: { token: tokenB, amount: 50n },   // field is `request`, not `requested`
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

`transactions.preview({ operation: "send" | "mint" | "consume" | "swap" | "pswapCreate" | "pswapConsume" | "pswapCancel" | "custom", ... })`
runs the same kernel as the real call but without proving or submitting,
returning a summary suitable for UI confirmation screens. The `pswap*`
operations correspond to the `transactions.pswapCreate` / `pswapConsume` /
`pswapCancel` partial-swap methods.

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
await client.notes.fetchPrivate();                       // pulls anything addressed to tracked accounts
await client.notes.sendPrivate({ note, to: "mtst1..." }); // `to`: bech32 string, 0x-hex string, Account, or AccountId (not a pre-parsed Address); delivers via the transport service
```

## Accounts (querying)

```typescript
await client.accounts.list();                         // tracked accounts
await client.accounts.get(ref);                       // single (returns null if not tracked)
await client.accounts.getOrImport(ref);               // tries get(), falls back to import()
await client.accounts.getDetails(ref);                // { account, vault, storage, code, keys }
await client.accounts.insert({ account, overwrite }); // start tracking an existing account
await client.accounts.getBalance(account, token);     // single-asset balance, returns bigint
```

`getDetails(ref)` returns `{ account, vault, storage, code, keys }` — the full
`Account`, its `AssetVault`, `AccountStorage`, `AccountCode | null`, and the key
commitments (`Word[]`); there is no `status` field.

For a single asset balance without loading the full vault, prefer
`client.accounts.getBalance(account, token)` (returns `bigint`). It wraps the
underlying WASM client's `accountReader(id)` lazy reader, which you can
drop into directly for finer-grained reads.

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
correct shape from a procedure body.

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
2. **`number` literals above 2^53 for amounts.** Amount fields accept
   `number | bigint` and coerce via `BigInt()` (no `TypeError`), but a numeric
   literal above `Number.MAX_SAFE_INTEGER` loses precision *before* coercion.
   Use `bigint` for large amounts.
3. **Omitting `type` and expecting a private note.** `send`/`mint` default
   `type` to **public** — pass `NoteVisibility.Private` explicitly for privacy.
4. **Passing a low-level `AccountId`-only WASM method a raw string** — resource
   methods accept hex/bech32 strings, but pre-parse with `AccountId.fromHex()`
   (and catch its throw) when calling APIs that demand an `AccountId` directly.
5. **Consuming notes before they're committed** — sync first, check status.
6. **Submitting `mint` with the recipient as `account`** — mint executes on
   the faucet account, not the target.
7. **Private notes without transport** — must call `notes.sendPrivate()` (or
   pass `returnNote: true` to `transactions.send` and deliver out-of-band).
8. **Holding WASM-owned objects across `terminate()`** — every `Account`,
   `Note`, `AccountId`, `NoteAndArgsArray` etc. owns Rust memory through the
   WASM ArrayBuffer. After `terminate()` they panic with "null pointer
   passed to rust" — drop references on unmount.
9. **Calling `accountReader(...)` in parallel with a write** — the readers
   share the WASM client. Wrap concurrent flows with `client.waitForIdle()`
   or rely on the React SDK's `runExclusive`.
