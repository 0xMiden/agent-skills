---
name: web-client-usage
description: Conventions for writing JavaScript/TypeScript code that uses the Miden web-client SDK (@miden-sdk/miden-sdk). Use when building apps on Miden, writing integration tests, or calling WebClient methods — covers initialization, transactions, notes, sync ordering, type conversions, and common pitfalls.
---

# Web-Client SDK Usage Patterns

## Client Initialization

### Basic

```typescript
const client = await WebClient.createClient(
  rpcUrl?,            // "devnet" | "testnet" | "localhost" | URL string
  noteTransportUrl?,  // for private note streaming
  seed?,              // Uint8Array (32 bytes) for deterministic key generation
  network?            // store name — use different values to isolate IndexedDB stores
);
```

All parameters are optional. Omit `rpcUrl` for testnet defaults.

### External Keystore

For HSM or custom key management:

```typescript
const client = await WebClient.createClientWithExternalKeystore(
  rpcUrl, noteTransportUrl, seed, storeName,
  async (pubKey: Uint8Array) => { /* return secretKey or null */ },
  async (pubKey: Uint8Array, secretKey: Uint8Array) => { /* store key */ },
  async (pubKey: Uint8Array, signingInputs: Uint8Array) => { /* return signature */ }
);
```

### Cleanup

Always call `client.terminate()` when done (e.g., on page unload).

### Multiple Clients

Multiple clients can share the same store (same `network` value) or use separate stores. Sync operations are coordinated via Web Locks across tabs.

## Sync — Always Sync First

**Critical:** Call `syncState()` before querying notes, accounts, or building transactions. The client's local state is only as fresh as the last sync.

```typescript
const summary = await client.syncState();
const blockNum = summary.blockNum();
```

Common sync patterns:
- Sync before consuming notes (notes must be committed on-chain)
- Sync after submitting a transaction (to see the result)
- Sync in a polling loop when waiting for confirmation

```typescript
// Wait for transaction confirmation
await client.submitNewTransaction(accountId, txRequest);
let confirmed = false;
while (!confirmed) {
  await new Promise(r => setTimeout(r, 3000));
  await client.syncState();
  const txs = await client.getTransactions(TransactionFilter.all());
  confirmed = txs.some(tx => tx.id().toHex() === expectedTxId && tx.transactionStatus().isCommitted());
}
```

## Type Conversions

These are the most common sources of bugs. Follow these rules:

### AccountId

```typescript
// From hex string
const id = AccountId.fromHex("0xabc123...");

// From bech32 address
const id = Address.fromBech32("mtst1abc...").accountId();

// Back to string
const hex = id.toString();   // "0x..."
const hex = id.toHex();      // "0x..."
```

Never pass raw strings to methods that expect `AccountId` — always construct the object first.

### Amounts — Always BigInt

```typescript
// Correct
BigInt(1000)
BigInt("1000")

// Wrong — will silently truncate or error
1000        // number, not bigint
"1000"      // string, not bigint
```

All token amounts in the SDK are `bigint`. When displaying, convert with `.toString()`.

### NoteType Enum

```typescript
NoteType.Public    // visible on-chain
NoteType.Private   // encrypted, requires transport
```

Import from the SDK, not as a string. Do not use magic numbers.

### Note Wrapping

Several wrapper types are needed for transaction building:

```typescript
// Wrap a Note into an OutputNote
const outputNote = OutputNote.full(note);

// Wrap into arrays for TransactionRequestBuilder
const outputNoteArray = new OutputNoteArray([outputNote]);
const noteAndArgs = new NoteAndArgs(note, null);  // null = no custom args
const noteAndArgsArray = new NoteAndArgsArray([noteAndArgs]);
```

### FungibleAsset Construction

```typescript
const asset = new FungibleAsset(faucetAccountId, BigInt(amount));
const noteAssets = new NoteAssets([asset]);
```

## Transaction Patterns

### Mint

Mints new tokens from a faucet to a target account:

```typescript
const txRequest = client.newMintTransactionRequest(
  targetAccountId,       // AccountId — who receives tokens
  faucetAccountId,       // AccountId — the faucet minting
  NoteType.Public,       // note visibility
  BigInt(1000)           // amount
);
await client.submitNewTransaction(faucetAccountId, txRequest);
```

Note: The transaction is submitted **from the faucet account**, not the target.

### Send

Sends tokens from one account to another:

```typescript
const txRequest = client.newSendTransactionRequest(
  senderAccountId,       // AccountId
  targetAccountId,       // AccountId
  faucetAccountId,       // AccountId — identifies the token/asset
  NoteType.Public,       // or NoteType.Private
  BigInt(100),           // amount
  null,                  // recallHeight — pass null if not needed
  null                   // timelockHeight — pass null if not needed
);
await client.submitNewTransaction(senderAccountId, txRequest);
```

**Private sends require an extra step** — send the note data to the recipient:

```typescript
const txResult = await client.executeTransaction(senderAccountId, txRequest);
const provenTx = await client.proveTransaction(txResult, prover);
const height = await client.submitProvenTransaction(provenTx, txResult);
await client.applyTransaction(txResult, height);

// Extract the full note and send via transport
const fullNote = txResult.executedTransaction().outputNotes().notes()[0].intoFull();
const recipientAddress = Address.fromBech32("mtst1recipient...");
await client.sendPrivateNote(fullNote, recipientAddress);
```

### Consume

Consumes notes received by an account:

```typescript
// First: get consumable notes
const consumableNotes = await client.getConsumableNotes(accountId);

// Extract the Note objects
const notes = consumableNotes.map(record => record.inputNoteRecord().toNote());

// Build consume request
const txRequest = client.newConsumeTransactionRequest(notes);
await client.submitNewTransaction(accountId, txRequest);
```

### Swap

Atomic exchange between two assets:

```typescript
const txRequest = client.newSwapTransactionRequest(
  accountId,             // AccountId — initiator
  offeredFaucetId,       // AccountId — asset being offered
  BigInt(100),           // offered amount
  requestedFaucetId,     // AccountId — asset being requested
  BigInt(50),            // requested amount
  NoteType.Public,       // swap note type
  NoteType.Private       // payback note type
);
await client.submitNewTransaction(accountId, txRequest);
```

### Custom Transactions (TransactionRequestBuilder)

For advanced use cases:

```typescript
// Create a P2ID note manually
const note = Note.createP2IDNote(
  senderId,
  receiverId,
  new NoteAssets([new FungibleAsset(faucetId, BigInt(100))]),
  NoteType.Public,
  new Felt(0n)           // aux data
);

const txRequest = new TransactionRequestBuilder()
  .withOwnOutputNotes(new OutputNoteArray([OutputNote.full(note)]))
  .build();

await client.submitNewTransaction(senderId, txRequest);
```

## Transaction Execution Pipeline

Two approaches:

### Simplified (one call)

```typescript
await client.submitNewTransaction(accountId, txRequest);
// or with custom prover:
await client.submitNewTransactionWithProver(accountId, txRequest, prover);
```

### Full control (four steps)

```typescript
const txResult = await client.executeTransaction(accountId, txRequest);
const provenTx = await client.proveTransaction(txResult, prover);
const height = await client.submitProvenTransaction(provenTx, txResult);
await client.applyTransaction(txResult, height);
```

Use the 4-step pipeline when you need to:
- Extract output notes before submitting (private sends)
- Use a custom/remote prover
- Inspect the transaction result before committing

## Querying

### Accounts

```typescript
// List all tracked accounts
const accounts = await client.getAccounts();

// Get specific account
const account = await client.getAccount(accountId);

// Account properties
account.id().toString();
account.vault().getBalance(faucetId);   // bigint
account.vault().fungibleAssets();        // array
account.storage().commitment().toHex();
account.code().commitment().toHex();
```

### Notes

```typescript
// Consumable notes for an account
const notes = await client.getConsumableNotes(accountId);

// All input notes (with filter)
const allNotes = await client.getInputNotes(NoteFilter.all());
const committed = await client.getInputNotes(
  new NoteFilter(NoteFilterTypes.Committed, undefined)
);
const specific = await client.getInputNotes(
  new NoteFilter(NoteFilterTypes.List, [noteId1, noteId2])
);

// Single note
const noteRecord = await client.getInputNote(noteId);
```

### Transactions

```typescript
const allTxs = await client.getTransactions(TransactionFilter.all());
const pending = await client.getTransactions(TransactionFilter.uncommitted());
const specific = await client.getTransactions(TransactionFilter.ids([txId]));

// Transaction status
tx.transactionStatus().isPending();
tx.transactionStatus().isCommitted();
tx.transactionStatus().isDiscarded();
```

## Import/Export

```typescript
// Export full store
const storeData = await client.exportStore();

// Import store
await client.forceImportStore(storeData, "StoreName");

// Export/import individual accounts
const accountFile = await client.exportAccountFile(accountId);
const bytes = Array.from(accountFile.serialize());
// ... later:
const file = AccountFile.deserialize(new Uint8Array(bytes));
await client.importAccountFile(file);

// Import account by ID (from network)
await client.importAccountById(accountId);
```

## Common Workflows

### Mint and Consume (fund an account)

```typescript
// 1. Mint from faucet to target
const mintTx = client.newMintTransactionRequest(
  targetId, faucetId, NoteType.Public, BigInt(1000)
);
await client.submitNewTransaction(faucetId, mintTx);

// 2. Wait for block
await new Promise(r => setTimeout(r, 5000));
await client.syncState();

// 3. Get and consume the minted notes
const notes = await client.getConsumableNotes(targetId);
const noteObjects = notes.map(n => n.inputNoteRecord().toNote());
const consumeTx = client.newConsumeTransactionRequest(noteObjects);
await client.submitNewTransaction(targetId, consumeTx);
```

### Check Balance

```typescript
await client.syncState();
const account = await client.getAccount(accountId);
const balance = account.vault().getBalance(faucetId);
console.log(`Balance: ${balance}`);
```

## Common Pitfalls

1. **Forgetting to sync** — Notes won't appear, balances will be stale
2. **Using `number` instead of `BigInt`** for amounts — silent precision loss
3. **Passing strings where AccountId is expected** — construct with `AccountId.fromHex()`
4. **Consuming notes before they're committed** — sync first, check status
5. **Submitting mint from target instead of faucet** — mint tx executes on the faucet account
6. **Private notes without transport** — must call `sendPrivateNote()` after execute
7. **Not wrapping notes** — `OutputNote.full(note)` and array wrappers are required for builder
8. **Race conditions** — use `AsyncLock.runExclusive()` when multiple operations share the client
