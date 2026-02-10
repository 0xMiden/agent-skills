---
name: idxdb-patterns
description: Enforce conventions for the IndexedDB/Dexie persistence layer in miden-client (idxdb-store crate). Use when editing TypeScript or JavaScript in crates/idxdb-store/src/ts/ or crates/idxdb-store/src/js/, writing Dexie transactions, or modifying the database schema.
---

# IndexedDB Store Patterns (idxdb-store)

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

Define TypeScript interfaces for each table with `I` prefix:

```typescript
export interface IAccount {
  id: string;
  codeRoot: string;
  storageRoot: string;
  vaultRoot: string;
  nonce: string;
  committed: boolean;
  accountSeed?: Uint8Array;
  accountCommitment: string;
  locked: boolean;
}

export interface IAccountCode {
  root: string;
  code: Uint8Array;
}
```

Rules:
- Use `string` for hex-encoded values (hashes, IDs, nonces)
- Use `Uint8Array` for raw binary data
- Use `?` suffix for optional fields (e.g., `accountSeed?`)
- Use `boolean` for flags, `number` for block heights

## Table Enum

Define tables as a TypeScript enum:

```typescript
enum Table {
  AccountCode = "accountCode",
  AccountStorage = "accountStorage",
  AccountAssets = "accountAssets",
  Accounts = "accounts",
  InputNotes = "inputNotes",
  OutputNotes = "outputNotes",
  Transactions = "transactions",
  BlockHeaders = "blockHeaders",
  StateSync = "stateSync",
  Tags = "tags",
}
```

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
    const tracked = await db.trackedAccounts.toArray();
    return tracked.map((entry) => entry.id);
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
const records = await db.accounts.toArray();

// Iterate with side effects
await db.accounts.each((record) => { /* ... */ });

// Get by primary key
const record = await db.accounts.get(id);

// Filter with where clause
const filtered = await db.accounts.where("committed").equals(1).toArray();
```

### Deduplication

When multiple versions of a record exist, keep the latest (highest nonce):

```typescript
const latestRecordsMap: Map<string, IAccount> = new Map();

await db.accounts.each((record) => {
  const existing = latestRecordsMap.get(record.id);
  if (!existing || BigInt(record.nonce) > BigInt(existing.nonce)) {
    latestRecordsMap.set(record.id, record);
  }
});

const latestRecords = Array.from(latestRecordsMap.values());
```

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
