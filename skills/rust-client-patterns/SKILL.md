---
name: rust-client-patterns
description: Enforce coding conventions for the miden-client Rust codebase (rust-client, sqlite-store). Use when editing, reviewing, or creating Rust code in the miden-client workspace — especially error handling, Store trait methods, Client impl blocks, and module organization.
---

# Miden Client Rust Patterns

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
- Module-level sections: `RE-EXPORTS`, `CLIENT ERROR`, `CONVERSIONS`
- Trait definitions by domain: `TRANSACTIONS`, `NOTES`, `CHAIN DATA`, `ACCOUNTS`, `SYNC`
- Impl block method groupings: `ACCOUNT CREATION`, `SYNC STATE`

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

    #[error("store error")]
    StoreError(#[source] StoreError),
}
```

Rules:
- Each variant has `#[error("...")]` with a clear, specific message
- Use `#[from]` for automatic conversion from nested error types
- Use `#[source]` when you don't want auto-From but want source chaining
- Include context data (IDs, values) in the error message itself

### ErrorHint for User Guidance

Implement `From<&YourError> for Option<ErrorHint>` to provide actionable help:

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
    .insert_account(account, default_address)
    .await
    .map_err(ClientError::StoreError)
```

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
3. Implement it in `WebStore` (`crates/idxdb-store/src/lib.rs`) — delegate to an internal async method
4. Add to the `StoreError` enum if new error conditions are needed

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

WebStore delegation pattern:
```rust
async fn get_note_tags(&self) -> Result<Vec<NoteTagRecord>, StoreError> {
    self.get_note_tags().await
}
```

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

All trait methods use `&self`, not `&mut self`. Implementations must use interior mutability (e.g., `Mutex`, `RwLock`, or browser-native locks).

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

Apply the AUTH constraint per impl block, not on the struct:

```rust
impl<AUTH> Client<AUTH>
where
    AUTH: TransactionAuthenticator + Sync + 'static,
{
    // methods that need authentication
}
```

Methods that don't need AUTH can use unconstrained blocks:

```rust
impl<AUTH> Client<AUTH> {
    // methods that work without authentication
}
```

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

Provide network-specific constructors: `for_testnet()`, `for_devnet()`, `for_localhost()`.

## no_std Compatibility

The `rust-client` crate is `no_std`. Follow these rules:

### Imports

Use `alloc::` and `core::` instead of `std::`:

```rust
#![no_std]

#[macro_use]
extern crate alloc;
use alloc::boxed::Box;
use alloc::string::{String, ToString};
use alloc::vec::Vec;
use core::fmt;

#[cfg(feature = "std")]
extern crate std;
```

### Feature Flags

Gate `std`-only dependencies behind the `std` feature:

```toml
[features]
default = ["std"]
std = [
  "miden-protocol/std",
  "miden-tx/concurrent",
  "tonic/transport",
]
```

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
/// - If the account is new but does not contain the seed.
/// - If the account is already tracked and `overwrite` is `false`.
pub async fn add_account(
    &mut self,
    account: &Account,
    overwrite: bool,
) -> Result<(), ClientError> {
```
