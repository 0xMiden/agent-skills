---
name: rust-client-patterns
description: Enforce coding conventions for the miden-client Rust codebase (rust-client, sqlite-store). Use when editing, reviewing, or creating Rust code in the miden-client workspace — covers error handling, Store trait methods, the Client<AUTH> generic, the Keystore super-trait that constrains the ClientBuilder, builder constructors, lazy reader patterns, and `no_std` organization.
---

# Miden Client Rust Patterns

The crate ships under `crates/rust-client`, with `crates/sqlite-store` as the
native persistence backend and `crates/testing` for test utilities. At v0.15
the WASM/IndexedDB store and the JS web client are no longer part of this
workspace — they live in the separate `0xMiden/web-sdk` repository. The MSRV
tracks `rust-toolchain.toml` in the upstream `miden-client` repository — copy
that channel into the consumer's toolchain file rather than hard-coding a
number that drifts.

## Section Headers

Organize code with section comment headers. Two levels:

**Top-level sections** (96 `=` characters):
```rust
// SECTION NAME
// ================================================================================================
```

**Subsections within impl blocks or traits** (92 `-` characters):
```rust
    // SUBSECTION NAME
    // --------------------------------------------------------------------------------------------
```

Use ALL-CAPS with spaces between words. Apply consistently to organize:
- Module-level sections: `RE-EXPORTS`, `MIDEN CLIENT`, `CLIENT RNG`, `CONSTANTS`
- Trait definitions by domain: `TRANSACTIONS`, `NOTES`, `CHAIN DATA`, `ACCOUNT`, `SYNC`
- Impl block method groupings: `ACCOUNT CREATION`, `ACCOUNT DATA RETRIEVAL`

(These are illustrative of the style; match the section name to the code's
domain rather than copying these verbatim.)

## Error Handling

### Primary Error Enum

Use `thiserror` with `#[derive(Debug, Error)]`:

```rust
#[derive(Debug, Error)]
pub enum ClientError {
    #[error("account with id {0} is already being tracked")]
    AccountAlreadyTracked(AccountId),

    #[error("account error")]
    AccountError(#[from] AccountError),

    #[error("storage error")]
    StoreError(#[from] StoreError),

    #[error("transaction script error")]
    TransactionScriptError(#[source] TransactionScriptError),
}
```

Rules:
- Each variant has `#[error("...")]` with a clear, specific message
- Use `#[from]` for automatic conversion from nested error types (this is the
  common case in `ClientError`, e.g. `AccountError`, `StoreError`,
  `RpcError`, `NoteScreenerError`)
- Use `#[source]` when you want source chaining but *not* an auto-`From`
  (e.g. `TransactionInputError`, `TransactionScriptError`, or struct variants
  with a named `#[source] source` field)
- Include context data (IDs, values) in the error message itself

### ErrorHint for User Guidance

Implement `From<&YourError> for Option<ErrorHint>` to provide actionable help.
Match the variants that have hints and fall back to `None`:

```rust
impl From<&ClientError> for Option<ErrorHint> {
    fn from(err: &ClientError) -> Self {
        match err {
            ClientError::MissingOutputRecipients(recipients) => {
                Some(missing_recipient_hint(recipients))
            },
            _ => None,
        }
    }
}
```

### Error Propagation

Wrap lower-level errors explicitly with `.map_err()`:

```rust
self.store
    .get_addresses_by_account_id(self.account_id)
    .await
    .map_err(ClientError::StoreError)
```

`.map_err(ClientError::StoreError)` is the canonical way to surface a
`StoreError` from a `Store` call inside the client.

Never use `.unwrap()` or `.expect()` in library code. Always propagate with `?` after mapping.

## Store Trait

### Cross-Platform Async Trait

The Store trait must work on both native and WASM. Always use this dual cfg_attr:

```rust
#[cfg_attr(not(target_arch = "wasm32"), async_trait::async_trait)]
#[cfg_attr(target_arch = "wasm32", async_trait::async_trait(?Send))]
pub trait Store: Send + Sync {
    // ...
}
```

The WASM variant uses `?Send` because WASM is single-threaded.

### Adding a New Store Method

1. Add the method signature to the `Store` trait in `crates/rust-client/src/store/mod.rs`, in the appropriate section
2. Implement it in `SqliteStore` (`crates/sqlite-store/src/lib.rs`) — delegate to a connection method
3. Add to the `StoreError` enum if new error conditions are needed

The native backend in this workspace is `SqliteStore`. The WASM/IndexedDB
`Store` implementation lives in the separate `0xMiden/web-sdk` repo; if a new
method must also exist there, mirror it in that repo.

SqliteStore delegation pattern:
```rust
async fn get_note_tags(&self) -> Result<Vec<NoteTagRecord>, StoreError> {
    self.interact_with_connection(SqliteStore::get_note_tags).await
}

async fn add_note_tag(&self, tag: NoteTagRecord) -> Result<bool, StoreError> {
    self.interact_with_connection(move |conn| SqliteStore::add_note_tag(conn, tag))
        .await
}
```

`interact_with_connection` acquires a pooled connection and runs the provided
closure, returning its `Result`.

### Trait Method Documentation

Every trait method needs a rustdoc comment explaining what it does:

```rust
/// Retrieves stored transactions, filtered by [`TransactionFilter`].
async fn get_transactions(
    &self,
    filter: TransactionFilter,
) -> Result<Vec<TransactionRecord>, StoreError>;
```

### Interior Mutability

All trait methods use `&self`, not `&mut self`. Implementations must use interior mutability (e.g., `Mutex`, `RwLock`, or browser-native locks), because the store's ownership is shared between the executor and the client.

All update operations must be atomic — if an error occurs partway through, roll back all changes.

## Client\<AUTH\> Pattern

### Struct Definition

The client is generic over the authenticator:

```rust
pub struct Client<AUTH> {
    store: Arc<dyn Store>,
    rng: ClientRng,
    rpc_api: Arc<dyn NodeRpcClient>,
    tx_prover: Arc<dyn TransactionProver + Send + Sync>,
    authenticator: Option<Arc<AUTH>>,
    // ...
}
```

Rules:
- Use `Arc<dyn Trait>` for polymorphic dependencies
- Use `Option<Arc<AUTH>>` for the optional authenticator
- Document each field with `///` comments

### Impl Block Constraints

Apply the AUTH constraint per impl block, not on the struct. At v0.15 the
`Client<AUTH>` impl blocks use these bounds:

```rust
// Constructors: `Client::builder()` requires the builder's authenticator bound.
impl<AUTH> Client<AUTH>
where
    AUTH: builder::BuilderAuthenticator,
{
    pub fn builder() -> builder::ClientBuilder<AUTH> { ... }
}

// Access / signing methods need at least TransactionAuthenticator.
impl<AUTH> Client<AUTH>
where
    AUTH: TransactionAuthenticator,
{
    pub fn authenticator(&self) -> Option<&Arc<AUTH>> { ... }
    // in_debug_mode, note_screener, rng, prover, source_manager, ...
}

// Methods that don't touch AUTH at all use an unconstrained block.
impl<AUTH> Client<AUTH> {
    pub fn store_identifier(&self) -> &str { ... }
}
```

(The sync methods sit in their own block bounded by
`AUTH: TransactionAuthenticator + Sync + 'static`.)

There is **no** `impl<AUTH> Client<AUTH> where AUTH: Keystore` block, and
`Client` exposes no key-management methods (no `add_key`/`remove_key`/`get_key`).
Key management is performed directly on the keystore object (see below), not
through `Client`. The only key-related accessor on `Client` is
`authenticator()`, which returns the stored `Option<&Arc<AUTH>>`.

### `Keystore` super-trait

`Keystore` (in `miden_client::keystore`) extends `TransactionAuthenticator`
and adds the unified key-management surface:

```rust
#[cfg_attr(not(target_arch = "wasm32"), async_trait::async_trait)]
#[cfg_attr(target_arch = "wasm32", async_trait::async_trait(?Send))]
pub trait Keystore: TransactionAuthenticator {
    async fn add_key(&self, key: &AuthSecretKey, account_id: AccountId) -> Result<(), KeyStoreError>;
    async fn remove_key(&self, pub_key: PublicKeyCommitment) -> Result<(), KeyStoreError>;
    async fn get_key(&self, pub_key: PublicKeyCommitment) -> Result<Option<AuthSecretKey>, KeyStoreError>;
    async fn get_account_key_commitments(&self, account_id: &AccountId)
        -> Result<BTreeSet<PublicKeyCommitment>, KeyStoreError>;
    async fn get_account_id_by_key_commitment(&self, pub_key_commitment: PublicKeyCommitment)
        -> Result<Option<AccountId>, KeyStoreError>;
    async fn get_keys_for_account(&self, account_id: &AccountId)
        -> Result<Vec<AuthSecretKey>, KeyStoreError> { ... } // default impl
}
```

A single `keystore.add_key(&secret, account_id)` call both stores the key
and associates it with the given account — there is no separate "insert key" +
"register commitment" step. In this workspace `FilesystemKeyStore` (behind the
`std` feature) is the only `Keystore` impl re-exported from
`crates/rust-client/src/keystore`; the WASM keystore now lives in the
`web-sdk` repo.

The `Keystore` bound reaches the client via the builder: the `ClientBuilder`
is constrained by `BuilderAuthenticator`, a marker super-trait defined as
`Keystore + 'static` (and additionally `From<FilesystemKeyStore>` under the
`std` feature).

### Builder Pattern

Use `ClientBuilder<AUTH>` with `Default` impl and detailed doc comments on each field:

```rust
pub struct ClientBuilder<AUTH> {
    /// An optional custom RPC client. If provided, this takes precedence over `rpc_endpoint`.
    rpc_api: Option<Arc<dyn NodeRpcClient>>,
    /// An optional store provided by the user.
    pub store: Option<StoreBuilder>,
    // ...
}
```

The `store` field holds a `StoreBuilder` (either an already-built
`Arc<dyn Store>` via `StoreBuilder::Store`, or a deferred `StoreFactory` via
`StoreBuilder::Factory`); the `.store(...)` method takes an `Arc<dyn Store>`
and wraps it into `StoreBuilder::Store`.

Provide network-specific constructors: `for_testnet()`, `for_devnet()`,
`for_localhost()`. Each returns `Self` synchronously and pre-fills the RPC
endpoint, default prover, and RNG appropriate for that network. Only `build()`
is async (it constructs the client from the configured components):

```rust
let client = ClientBuilder::for_testnet()
    .store(store)                        // Arc<dyn Store>
    .authenticator(Arc::new(keystore))   // accepts any AUTH: BuilderAuthenticator (i.e. Keystore + 'static)
    .build()
    .await?;
```

## Lazy Reader Patterns

Prefer the lazy readers over loading whole `Account` / `Note` records when
all you need is one field — they avoid materializing storage maps and
asset vaults that a frontend will not display anyway.

```rust
// Account fields without loading the full Account
let reader = client.account_reader(account_id);
let (header, status) = reader.header().await?;
let balance        = reader.get_balance(faucet_id).await?;
let storage_item   = reader.get_storage_item(slot_name).await?;  // slot_name: impl Into<StorageSlotName>
let nonce          = reader.nonce().await?;
let vault_root     = reader.vault_root().await?;
let storage_root   = reader.storage_commitment().await?;
let code_root      = reader.code_commitment().await?;

// Iterator over consumable input notes
let mut notes = client.input_note_reader(consumer_account_id);
while let Some(note) = notes.next().await? {
    // process each InputNoteRecord
}
```

Both readers borrow `&self` and may be invoked concurrently with one another.
They must not be used concurrently with a `Client` write that targets the
same account, so wrap mixed flows in a serializing layer. (The JS wrapper that
serializes WASM calls for browser consumers lives in the `web-sdk` repo.)

## State Sync

`Client::sync_state()` takes no arguments (it borrows `&mut self`). Internally
it runs note-transport sync (`sync_note_transport()`) and then `sync_chain()`;
`sync_chain` is where the building blocks are wired together:

```rust
let summary = client.sync_state().await?;
```

For custom sync flows the building blocks are public on `Client`:

```rust
use miden_client::sync::{StateSync, StateSyncInput, StateSyncUpdate};

// build_sync_input() collects tracked account headers, all unique note tags,
// all unspent input & output notes, and all uncommitted transactions.
let mut input: StateSyncInput = client.build_sync_input().await?;
input.note_tags.insert(extra_tag); // note_tags is a BTreeSet<NoteTag>

// Construct a StateSync with the same rpc / note-screener the client uses
// internally, fetch the current PartialMmr, and drive the sync. This part
// mirrors Client::sync_chain's body in crates/rust-client/src/sync/mod.rs:
//
//   let state_sync = StateSync::new(rpc_api, Arc::new(note_screener), tx_discard_delta);
//   let mut partial_mmr = client.get_current_partial_mmr().await?;
//   let update: StateSyncUpdate = state_sync.sync_state(&mut partial_mmr, input).await?;
//
// Persist the result. `Client::apply_state_sync` writes the update to the
// store and prunes irrelevant blocks. (Note: unlike `sync_chain`, this
// convenience does NOT re-cache the PartialMmr; if you rely on the in-memory
// MMR cache, replicate sync_chain's cache_partial_mmr step yourself.)
client.apply_state_sync(update).await?;
```

`StateSync::new(rpc_api, note_screener, tx_discard_delta)`,
`Client::get_current_partial_mmr()`, and
`StateSync::sync_state(&mut partial_mmr, input)` are the load-bearing pieces —
copy that construction from `Client::sync_chain` rather than reinventing it.

## no_std Compatibility

The `rust-client` crate is `no_std`. Follow these rules:

### Imports

Declare `#![no_std]` and pull in `alloc` (and `std` only under the feature);
use `alloc::` / `core::` paths instead of `std::`:

```rust
#![no_std]

#[macro_use]
extern crate alloc;
use alloc::boxed::Box;

#[cfg(feature = "std")]
extern crate std;
```

Within modules, import the collection/string/fmt types you need from `alloc`
/ `core` (e.g. `alloc::string::String`, `alloc::vec::Vec`, `core::fmt`).

### Feature Flags

Gate `std`-only dependencies behind the `std` feature. `default = ["std"]`,
and `std` re-exports `concurrent` plus the `std` features of upstream crates:

```toml
[features]
concurrent = ["miden-tx/concurrent"]
default = ["std"]
std = [
  "concurrent",
  "miden-protocol/std",
  "tonic/transport",
  # ...plus tokio, tempfile, and other upstream `std` features
]
```

Note `miden-tx/concurrent` is pulled in indirectly via the `concurrent`
feature rather than listed directly under `std`.

### Platform-Specific Code

Use `#[cfg_attr]` for WASM vs native differences:

```rust
#[cfg_attr(not(target_arch = "wasm32"), async_trait::async_trait)]
#[cfg_attr(target_arch = "wasm32", async_trait::async_trait(?Send))]
```

## Public API Documentation

### Module-Level Docs

Start every module file with `//!` documentation:

```rust
//! Provides the client APIs for synchronizing the client's local state with the Miden
//! network.
//!
//! ## Overview
//!
//! This module handles the synchronization process between the local client and the
//! Miden network.
//!
//! ## Examples
//!
//! ```rust,ignore
//! let sync_summary: SyncSummary = client.sync_state().await?;
//! ```
```

### Function/Method Docs

- One-line summary starting with a verb
- Detailed explanation if non-obvious
- `# Errors` section listing error conditions
- Use backtick references: `[`Type`]`
- Aim for ~100 character line length in doc comments

```rust
/// Adds the provided [`Account`] in the store so it can start being tracked.
///
/// # Errors
///
/// - If the account is new but it does not contain the seed.
/// - If the account is already tracked and `overwrite` is set to `false`.
pub async fn add_account(
    &mut self,
    account: &Account,
    overwrite: bool,
) -> Result<(), ClientError> {
```
