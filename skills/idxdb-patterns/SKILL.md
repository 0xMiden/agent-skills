---
name: idxdb-patterns
description: Enforce conventions for the IndexedDB/Dexie persistence layer in miden-client v0.14 (idxdb-store crate). Use when editing TypeScript or JavaScript in `crates/idxdb-store/src/ts/` or `crates/idxdb-store/src/js/`, writing Dexie transactions, or modifying the database schema.
---

# IndexedDB Store Patterns (idxdb-store, v0.14)

The v0.14 schema splits account-related tables into `Latest…` /
`Historical…` pairs to support the new account-history pruning feature
(`client.pruneAccountHistory()`). Always check
`crates/idxdb-store/src/ts/schema.ts` for the canonical table list before
adding rows or filters — the table set has grown
(`AccountAuth`, `AccountKeyMapping`, `Addresses`, `Settings`,
`ForeignAccountCode`, `NotesScripts`, `TransactionScripts`,
`PartialBlockchainNodes`, `LatestStorageMapEntries`,
`HistoricalStorageMapEntries`, …) since v0.13.

## Build Workflow

The `idxdb-store` has a dual-file workflow:

- **TypeScript source** lives in `crates/idxdb-store/src/ts/`
- **Generated JavaScript** lives in `crates/idxdb-store/src/js/`
- **Both are committed to git**
- CI compiles TS and compares against committed JS — they must match

After modifying any `.ts` file:
```bash
cd crates/idxdb-store/src
yarn build   # Compiles TS to JS
```

Always commit both the `.ts` source and the generated `.js` output together.

## Database Registry

Use a global `Map` to track database instances by ID:

```typescript
const databaseRegistry = new Map<string, MidenDatabase>();

export function getDatabase(dbId: string): MidenDatabase {
  const db = databaseRegistry.get(dbId);
  if (!db) {
    throw new Error(
      `Database not found for id: ${dbId}. Call openDatabase first.`
    );
  }
  return db;
}

export async function openDatabase(
  network: string,
  clientVersion: string
): Promise<string> {
  const db = new MidenDatabase(network);
  await db.open(clientVersion);
  databaseRegistry.set(network, db);
  return network;
}
```

Rules:
- Every exported function takes `dbId: string` as its first parameter
- Always call `getDatabase(dbId)` at the top of each function — never cache the reference
- The `dbId` is the network name (e.g., `"tests"`, `"testnet"`)

## Schema Interfaces

Define TypeScript interfaces for each table with `I` prefix. Use `Latest…`
/ `Historical…` pairs for anything that participates in account history
(headers, storage slots, storage map entries, vault assets). Account
headers use the slightly older `IAccount` / `IHistoricalAccount` naming
inside `latestAccountHeaders` / `historicalAccountHeaders`:

```typescript
export interface IAccountCode {
  root: string;
  code: Uint8Array;
}

export interface ILatestAccountStorage {
  accountId: string;
  slotName: string;
  slotValue: string;
  slotType: number;
}

export interface IHistoricalAccountStorage {
  accountId: string;
  replacedAtNonce: string;
  slotName: string;
  oldSlotValue: string | null;
  slotType: number;
}

export interface ILatestAccountAsset {
  accountId: string;
  vaultKey: string;     // ASSET_KEY in v0.14 — see `miden-concepts` skill
  asset: string;        // ASSET_VALUE serialized
}

export interface IHistoricalAccountAsset {
  accountId: string;
  replacedAtNonce: string;
  vaultKey: string;
  oldAsset: string | null;
}
```

Rules:
- Use `string` for hex-encoded values (hashes, IDs, nonces, vault keys)
- Use `Uint8Array` for raw binary data
- Use `?` suffix for optional fields, `| null` when the column explicitly
  represents the absence of a previous value (replaced-from in history)
- Use `boolean` for flags, `number` for block heights and slot types
- The asset layer in v0.14 is two-word: `vaultKey` is the `ASSET_KEY` and
  `asset` is the encoded `ASSET_VALUE`. Don't try to fold them back into a
  single hex string.

## Table Enum

Define tables as a TypeScript enum. Match `crates/idxdb-store/src/ts/schema.ts`
exactly — the Rust side imports table names verbatim:

```typescript
enum Table {
  AccountCode = "accountCode",
  LatestAccountStorage = "latestAccountStorage",
  HistoricalAccountStorage = "historicalAccountStorage",
  LatestAccountAssets = "latestAccountAssets",
  HistoricalAccountAssets = "historicalAccountAssets",
  LatestStorageMapEntries = "latestStorageMapEntries",
  HistoricalStorageMapEntries = "historicalStorageMapEntries",
  LatestAccountHeaders = "latestAccountHeaders",
  HistoricalAccountHeaders = "historicalAccountHeaders",
  AccountAuth = "accountAuth",
  AccountKeyMapping = "accountKeyMapping",
  Addresses = "addresses",
  Transactions = "transactions",
  TransactionScripts = "transactionScripts",
  InputNotes = "inputNotes",
  OutputNotes = "outputNotes",
  NotesScripts = "notesScripts",
  StateSync = "stateSync",
  BlockHeaders = "blockHeaders",
  PartialBlockchainNodes = "partialBlockchainNodes",
  Tags = "tags",
  ForeignAccountCode = "foreignAccountCode",
  Settings = "settings",
}
```

Adding a new table requires a coordinated update in three places:
1. `Table` enum + interface in `schema.ts`
2. The Dexie store version (`MidenDatabase.version(...).stores({...})`),
   bumping the version number and writing a migration if the change is
   not purely additive
3. Rust-side reads/writes that import the table name through
   `#[wasm_bindgen(module = "/src/js/schema.js")]`

## Dexie Transactions

### Atomic Operations

When multiple tables must be updated together, wrap in a Dexie transaction:

```typescript
const tablesToAccess = [
  db.stateSync,
  db.inputNotes,
  db.outputNotes,
  db.transactions,
  db.blockHeaders,
  db.partialBlockchainNodes,
  db.tags,
];

return await db.dexie.transaction("rw", tablesToAccess, async (tx) => {
  await Promise.all([
    inputNotesWriteOp,
    outputNotesWriteOp,
    transactionWriteOp,
    accountUpdatesWriteOp,
    updateSyncHeight(tx, blockNum),
    updatePartialBlockchainNodes(tx, serializedNodeIds, serializedNodes),
  ]);
});
```

Rules:
- Use `"rw"` for read-write transactions
- List all tables that will be accessed in the `tablesToAccess` array
- Use `Promise.all()` inside transactions to parallelize independent operations
- Pass the `tx` transaction object to helper functions that need table access

### Table Access Within Transactions

Type-cast the transaction to access tables:

```typescript
async function updateSyncHeight(tx: Transaction, blockNum: number) {
  try {
    const current = await (
      tx as Transaction & { stateSync: Dexie.Table<IStateSync, number> }
    ).stateSync.get(1);
    if (!current || current.blockNum < blockNum) {
      await (
        tx as Transaction & { stateSync: Dexie.Table<IStateSync, number> }
      ).stateSync.update(1, { blockNum: blockNum });
    }
  } catch (error) {
    logWebStoreError(error, "Failed to update sync height");
  }
}
```

### Forward-Only Updates

Only update state forward (never regress):

```typescript
if (!current || current.blockNum < blockNum) {
  // Update
}
```

## Error Handling

### logWebStoreError

Always use `logWebStoreError()` for error logging — never `console.error()` directly:

```typescript
import { logWebStoreError } from "./utils.js";

try {
  // database operation
} catch (error) {
  logWebStoreError(error, "Error while fetching account headers");
}
```

### Graceful Degradation

Functions that query data should return empty arrays or `null` on error, not throw:

```typescript
export async function getAccountIds(dbId: string) {
  try {
    const db = getDatabase(dbId);
    const headers = await db.latestAccountHeaders.toArray();
    return headers.map((entry) => entry.accountId);
  } catch (error) {
    logWebStoreError(error, "Error while fetching account IDs");
  }
  return [];  // Return empty on error
}
```

## Data Operations

### Querying

Use Dexie's query API:

```typescript
// Get all records
const records = await db.latestAccountHeaders.toArray();

// Iterate with side effects
await db.latestAccountHeaders.each((record) => { /* ... */ });

// Get by primary key
const record = await db.latestAccountHeaders.get(accountId);

// Filter with where clause
const filtered = await db.latestAccountHeaders
  .where("storageMode")
  .equals("public")
  .toArray();
```

### Latest vs Historical

For account state, "latest" tables hold the current row keyed by
`accountId` (or `accountId + slotName / vaultKey`); the matching
"historical" tables hold previous rows keyed by `replacedAtNonce`. To
find the row in effect at a specific nonce:

```typescript
// Walk back through the historical rows until one was replaced strictly
// after the target nonce, then take its `oldSlotValue` (or fall back to
// the latest row if no historical row covers the target nonce).
const targetNonce = BigInt("123");
const historical = await db.historicalAccountStorage
  .where(["accountId", "slotName"])
  .equals([accountId, slotName])
  .filter((r) => BigInt(r.replacedAtNonce) > targetNonce)
  .first();

const value = historical
  ? historical.oldSlotValue
  : (await db.latestAccountStorage.get([accountId, slotName]))?.slotValue;
```

`client.pruneAccountHistory()` is the public entry point that drops
`Historical…` rows below a configured retention floor — write functions
must keep the latest row authoritative regardless of what history is
present.

### Serialization Conventions

- Hex strings for cryptographic values (hashes, IDs, commitments)
- Base64 for binary data (seeds, serialized objects): use `uint8ArrayToBase64()`
- `Uint8Array` for direct binary storage
- Default empty strings for optional string fields: `record.storageRoot || ""`
- `BigInt()` for nonce comparisons (stored as strings)

## Upsert Pattern

For operations that may create or update records:

```typescript
export async function upsertInputNote(
  dbId: string,
  noteId: string,
  noteAssets: Uint8Array | undefined,
  // ... more params
) {
  const db = getDatabase(dbId);
  const existing = await db.inputNotes.get(noteId);

  if (existing) {
    // Merge: only update fields that have new values
    await db.inputNotes.update(noteId, {
      ...(noteAssets && { noteAssets }),
      // ... only non-undefined fields
    });
  } else {
    await db.inputNotes.put({
      noteId,
      noteAssets,
      // ... all fields
    });
  }
}
```
