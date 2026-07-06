---
name: idxdb-patterns
description: Enforce conventions for the IndexedDB/Dexie persistence layer of the Miden web client, which lives in the web-sdk repo (idxdb-store crate). Use when editing TypeScript in `crates/idxdb-store/src/ts/`, writing Dexie transactions, or modifying the database schema.
---

# IndexedDB Store Patterns (idxdb-store)

This layer lives in the **web-sdk** repo (`github.com/0xMiden/web-sdk`,
crate `crates/idxdb-store`, package `miden-idxdb-store`), not in the
`miden-client` repo. It is a Dexie-backed `Store` implementation for the
WASM web client.

The schema splits account-related tables into `Latest…` / `Historical…`
pairs to support account-history pruning (`client.pruneAccountHistory()`).
Always check `crates/idxdb-store/src/ts/schema.ts` for the canonical table
list before adding rows or filters — the active set includes `AccountAuth`,
`AccountKeyMapping`, `Addresses`, `Settings`, `ForeignAccountCode`,
`NotesScripts`, `TransactionScripts`, `PartialBlockchainNodes`,
`LatestStorageMapEntries`, `HistoricalStorageMapEntries`, plus the
account-storage / asset / account-header latest/historical pairs.

## Build Workflow

The `idxdb-store` has a dual-file workflow:

- **TypeScript source** lives in `crates/idxdb-store/src/ts/`
- **Generated JavaScript** lives in `crates/idxdb-store/src/js/`
- **Both are committed to git** — the `js` folder is currently *not*
  gitignored (the `#js` entry in `src/.gitignore` is commented out)
- The Rust side imports the generated `.js` modules via
  `#[wasm_bindgen(module = "/src/js/...")]`, so the JS must be kept in
  sync with the TS

After modifying any `.ts` file, regenerate the JS with the canonical
top-level Make target (which runs the package's `build` script through
**pnpm** — this repo is pnpm-only, there is no yarn):

```bash
make rust-client-ts-build   # == pnpm --filter web_store run build
```

The underlying package script is `tsc --build --force ./tsconfig.json`
(`crates/idxdb-store/src/package.json`). Always commit both the `.ts`
source and the regenerated `.js` output together.

## Database Registry

There is no JS object pointer on the Rust side, so open databases are
tracked in a module-level `Map` keyed by network name, in
`crates/idxdb-store/src/ts/schema.ts`:

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
  const success = await db.open(clientVersion);
  if (!success) {
    throw new Error(`Failed to open IndexedDB database: ${network}`);
  }
  databaseRegistry.set(network, db);
  return network;
}
```

Rules:
- Every exported store function takes `dbId: string` as its first parameter
- Call `const db = getDatabase(dbId)` at the top of each function — look it
  up per call rather than holding a long-lived reference across calls
- The `dbId` is the network name (`"mainnet"`, `"devnet"`, `"testnet"`, or
  a custom one); `openDatabase` registers under and returns `network`

## Schema Interfaces

Define TypeScript interfaces for each table with the `I` prefix. Use
`Latest…` / `Historical…` pairs for anything that participates in account
history (storage slots, storage map entries, vault assets, and account
headers). The account-header pair uses `IAccount` (latest, keyed on `id`)
and `IHistoricalAccount` (same fields plus `replacedAtNonce`) inside
`latestAccountHeaders` / `historicalAccountHeaders`:

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
  vaultKey: string;     // ASSET_KEY — see `miden-concepts` skill
  asset: string;        // ASSET_VALUE serialized
}

export interface IHistoricalAccountAsset {
  accountId: string;
  replacedAtNonce: string;
  vaultKey: string;
  oldAsset: string | null;
}

export interface IAccount {
  id: string;           // primary key — NOT `accountId`
  codeRoot: string;
  storageRoot: string;
  vaultRoot: string;
  nonce: string;
  committed: boolean;
  accountSeed?: Uint8Array;
  accountCommitment: string;
  locked: boolean;
  watched: boolean;
}
```

Rules:
- Use `string` for hex-encoded values (hashes, IDs, commitments, nonces,
  vault keys)
- Use `Uint8Array` for raw binary data
- Use `?` suffix for optional fields, `| null` when the column explicitly
  represents the absence of a previous value (e.g. `oldSlotValue`,
  `oldAsset`, `oldValue` in the history tables)
- Use `boolean` for flags, `number` for block heights and slot types
- The LATEST account-header table keys on `id`; the HISTORICAL
  account-header table keys on `accountCommitment` (with `id` and
  `[id+replacedAtNonce]` as secondary indexes). The storage / asset /
  map-entry / foreign-code tables key on `accountId`. Don't confuse the two.
- The asset layer is two-word: `vaultKey` is the `ASSET_KEY` and `asset`
  is the encoded `ASSET_VALUE`. Don't fold them back into a single hex
  string.

## Table Enum

Define tables as a TypeScript enum, in the order they appear in
`crates/idxdb-store/src/ts/schema.ts` — the Rust side imports table-backed
JS functions verbatim:

```typescript
enum Table {
  AccountCode = "accountCode",
  LatestAccountStorage = "latestAccountStorage",
  HistoricalAccountStorage = "historicalAccountStorage",
  LatestAccountAssets = "latestAccountAssets",
  HistoricalAccountAssets = "historicalAccountAssets",
  LatestStorageMapEntries = "latestStorageMapEntries",
  HistoricalStorageMapEntries = "historicalStorageMapEntries",
  AccountAuth = "accountAuth",
  AccountKeyMapping = "accountKeyMapping",
  LatestAccountHeaders = "latestAccountHeaders",
  HistoricalAccountHeaders = "historicalAccountHeaders",
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

The Dexie store schema is defined once, as the `V1_STORES` constant
applied via `this.dexie.version(1).stores(V1_STORES)` in the
`MidenDatabase` constructor. `V1_STORES` is the frozen baseline: index
strings are built with a small `indexes(...)` helper, e.g.
`[Table.LatestAccountStorage]: indexes("[accountId+slotName]", "accountId")`.

The migration system is **not currently in use** — the Miden network
resets on every upgrade, so `ensureClientVersion` nukes the DB (close /
`delete` / re-open) when the running client version is a higher major or
minor than the stored one; same-major.minor patch bumps and downgrades
just persist the new version without resetting (see the semver
`sameMajorMinor` / `!semver.gt(...)` guard in `ensureClientVersion`).
A minor-version bump does trigger it. Adding a table or
changing an index today therefore means:
1. Update the `Table` enum + interface(s) in `schema.ts`
2. Add the table/index to `V1_STORES` (additive, since the DB is nuked on
   version change; once migrations are enabled, `V1_STORES` must be frozen
   and a new `.version(N+1).stores({...}).upgrade(...)` block added instead)
3. Update Rust-side reads/writes, which import the corresponding JS
   functions through `#[wasm_bindgen(module = "/src/js/<file>.js")]`
   (e.g. account functions from `/src/js/accounts.js`, schema/registry
   functions from `/src/js/schema.js`)
4. Run `make rust-client-ts-build` to regenerate the JS, and add a
   schema/migration test in `schema.test.ts`

## Dexie Transactions

### Atomic Operations

When multiple tables must be updated together, wrap in a Dexie transaction.
List every table the transaction touches, and use `Promise.all()` to run
independent operations concurrently (from `applyStateSync` in
`crates/idxdb-store/src/ts/sync.ts`):

```typescript
const tablesToAccess = [
  db.stateSync,
  db.inputNotes,
  db.outputNotes,
  db.notesScripts,
  db.transactions,
  db.transactionScripts,
  db.blockHeaders,
  db.partialBlockchainNodes,
  db.tags,
  db.latestAccountHeaders,
  db.historicalAccountHeaders,
  // ... plus the latest/historical storage, map-entry and asset tables
];

return await db.dexie.transaction("rw", tablesToAccess, async (tx) => {
  await Promise.all([
    /* input/output note upserts */,
    /* transaction upserts */,
    /* per-account applyFullAccountState calls */,
    updateSyncHeight(tx, blockNum),
    updatePartialBlockchainNodes(tx, serializedNodeIds, serializedNodes),
    updateCommittedNoteTags(tx, committedNoteTagSources),
    /* block-header writes */,
  ]);
});
```

Rules:
- Use `"rw"` for read-write transactions
- List all tables that will be accessed in the `tablesToAccess` array
- Use `Promise.all()` inside transactions to parallelize independent operations
- Pass the `tx` transaction object to helper functions that need table access
- Helper write functions (e.g. `upsertInputNote`) commonly take an optional
  `tx?: Transaction`: if supplied they run inside the caller's transaction,
  otherwise they open their own `db.dexie.transaction(...)`

### Table Access Within Transactions

The Dexie `Transaction` type doesn't statically declare table accessors.
`schema.ts` augments `declare module "dexie"` so `tx.inputNotes` etc.
type-check; where that augmentation isn't in scope, type-cast the
transaction (from `updateSyncHeight` in `sync.ts`):

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

Only advance the sync height forward (never regress):

```typescript
if (!current || current.blockNum < blockNum) {
  // Update
}
```

## Error Handling

### logWebStoreError

Use `logWebStoreError()` from `./utils.js` for error logging — it formats
Dexie errors (and walks `error.inner`), then **re-throws** the error:

```typescript
import { logWebStoreError } from "./utils.js";

try {
  // database operation
} catch (error) {
  logWebStoreError(error, "Error while fetching account headers");
}
```

Because `logWebStoreError` always re-throws, code after a `catch` that
calls it (e.g. a trailing `return []`) is effectively unreachable on the
error path — the surrounding `try` body must return the success value.

### Reads return optional / empty

Read functions wrap their body in `try/catch`, returning the queried value
on success. The fallback after the catch (empty array, `null`, or
`undefined`) documents intent but is unreachable because `logWebStoreError`
re-throws (from `getAccountIds` in `crates/idxdb-store/src/ts/accounts.ts`):

```typescript
export async function getAccountIds(dbId: string) {
  try {
    const db = getDatabase(dbId);
    const records = await db.latestAccountHeaders.toArray();
    return records.map((entry) => entry.id);   // header rows key on `id`
  } catch (error) {
    logWebStoreError(error, "Error while fetching account IDs");
  }
  return [];
}
```

## Data Operations

### Querying

Use Dexie's query API. Patterns actually used in the store:

```typescript
// Get all records
const records = await db.latestAccountHeaders.toArray();

// Get by primary key (e.g. stateSync row id 1)
const current = await db.stateSync.get(1);

// Look up a header by its `id` index (header PK is `id`)
const record = await db.latestAccountHeaders
  .where("id")
  .equals(accountId)
  .first();

// Filter a single-field index, optionally narrowing with `.and(...)`
const slots = await db.latestAccountStorages
  .where("accountId")
  .equals(accountId)
  .and((record) => nameSet.has(record.slotName))
  .toArray();

// Match multiple keys against one index
const codes = await db.accountCodes.where("root").anyOf(codeRoots).toArray();
```

For compound indexes, use the **bracket-string** index name and pass the
key parts as an array to `.equals(...)` (from `applyTransactionDelta`):

```typescript
const oldSlot = await db.latestAccountStorages
  .where("[accountId+slotName]")
  .equals([accountId, slot.slotName])
  .first();
```

### Latest vs Historical

For account state, the `latest…` tables hold the current row (keyed by
`accountId`, or the compound `[accountId+slotName]` / `[accountId+vaultKey]`
/ `[accountId+slotName+key]`); the matching `historical…` tables hold the
value that was replaced, keyed by `[accountId+replacedAtNonce…]` with the
prior value in `oldSlotValue` / `oldAsset` / `oldValue` (`null` when no
previous value existed). The write path is **archive-then-replace**: read
the current latest row, `put` it into historical under the new nonce, then
`put` the new value into latest (see `applyTransactionDelta` /
`applyFullAccountState`).

Undo restores from history back to latest, keyed by the compound nonce
index; a non-null old value overwrites latest, a `null` old value deletes
the latest row (from `restoreSlotsFromHistorical` in `accounts.ts`):

```typescript
const oldSlots = await db.historicalAccountStorages
  .where("[accountId+replacedAtNonce]")
  .equals([accountId, nonce])
  .toArray();

for (const slot of oldSlots) {
  if (slot.oldSlotValue !== null) {
    await db.latestAccountStorages.put({ /* ...restore old value... */ });
  } else {
    await db.latestAccountStorages
      .where("[accountId+slotName]")
      .equals([accountId, slot.slotName])
      .delete();
  }
}
```

`client.pruneAccountHistory()` (web-client `pruneAccountHistory`, backed by
the JS `pruneAccountHistory` in `accounts.ts`) drops `historical…` rows
whose `replacedAtNonce <= upToNonce` and any orphaned account code. Write
functions must keep the latest row authoritative regardless of how much
history has been pruned.

### Serialization Conventions

- Hex strings for cryptographic values (hashes, IDs, commitments, vault keys)
- `uint8ArrayToBase64()` (from `./utils.js`) when a `Uint8Array` must be
  returned to Rust as a base64 string (e.g. serialized account code, seeds)
- `Uint8Array` for direct binary storage in a table column
- Default empty strings for optional string fields when reading out:
  `record.storageRoot || ""`
- `BigInt()` for nonce comparisons and sorting (nonces are stored as
  strings, so lexicographic / index-range ordering would be wrong)

## Upsert Pattern

Dexie `Table.put()` is itself an upsert: it inserts or replaces by primary
key. The store builds a plain data object and calls `.put()` — it does not
read-then-branch. Convert `null` to `undefined` so Dexie omits the field
from indexes (a `null` in a compound index is a real value; an absent
field is skipped). From `upsertInputNote` in
`crates/idxdb-store/src/ts/notes.ts`:

```typescript
export async function upsertInputNote(
  dbId: string,
  detailsCommitment: string,
  noteId: string | undefined,
  // ... more params
  consumedBlockHeight?: number | null,
  consumedTxOrder?: number | null,
  consumerAccountId?: string | null,
  tx?: Transaction
) {
  const db = getDatabase(dbId);
  const doWork = async (t: Transaction) => {
    try {
      const data = {
        detailsCommitment,
        noteId: noteId ?? undefined,
        // null -> undefined so Dexie omits these from compound indexes
        consumedBlockHeight: consumedBlockHeight ?? undefined,
        consumedTxOrder: consumedTxOrder ?? undefined,
        consumerAccountId: consumerAccountId ?? undefined,
        // ... remaining fields
      };
      await t.inputNotes.put(data);
      await t.notesScripts.put({ scriptRoot, serializedNoteScript });
    } catch (error) {
      logWebStoreError(error, `Error inserting note: ${detailsCommitment}`);
    }
  };
  // Run inside the caller's tx if provided, else open one.
  if (tx) return doWork(tx);
  return db.dexie.transaction("rw", db.inputNotes, db.notesScripts, doWork);
}
```
