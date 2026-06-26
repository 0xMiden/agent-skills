---
name: rust-sdk-testing-patterns
description: Guide to testing Miden smart contracts with MockChain (Miden v0.15). Covers test setup, contract building, account/note creation, transaction execution, storage verification, faucet setup, output note verification, block numbering, multi-transaction tests, and asset-bearing notes. Use when writing, editing, or debugging Miden integration tests.
---

# Miden Testing Patterns (MockChain)

These patterns target Miden **v0.15** (`miden-protocol`/`miden-standards`/`miden-testing` 0.15.x). Several types were renamed or removed in the v0.14 -> v0.15 migration (see the notes inline); the upstream `project-template` may still be on v0.14, so verify any copied snippet compiles under v0.15 before relying on it.

## Test File Setup

Tests go in `integration/tests/`. All tests are async and use MockChain for local execution without a network.

See `integration/tests/counter_test.rs` in [project-template](https://github.com/0xMiden/project-template) for a complete working test covering imports, MockChain setup, contract building, account creation with storage, note creation, transaction execution, and storage verification. The template lags the protocol release: confirm it is on the 0.15 line before copying APIs (older revisions use the removed v0.14 types `AccountStorageMode`, `AuthSchemeId`, and `AccountType::RegularAccountImmutableCode`).

## Step-by-Step Test Pattern

### 1. Initialize MockChain Builder

Start from `let mut builder = MockChain::builder();` (see `integration/tests/counter_test.rs` in [project-template](https://github.com/0xMiden/project-template)).

### 2. Create Sender/Wallet Accounts

See `integration/tests/counter_test.rs` in [project-template](https://github.com/0xMiden/project-template) for the basic wallet pattern. For wallets with pre-funded assets, use `builder.add_existing_wallet_with_assets(Auth::BasicAuth { auth_scheme: AuthScheme::Falcon512Poseidon2 }, [FungibleAsset::new(faucet.id(), 100)?.into()])`.

> v0.15: the auth scheme enum is `AuthScheme` (the v0.14 name `AuthSchemeId` was removed). The variant `Falcon512Poseidon2` is unchanged.

### 3. Set Up Faucets (for fungible assets)
```rust
let faucet = builder.add_existing_basic_faucet(
    Auth::BasicAuth {
        auth_scheme: AuthScheme::Falcon512Poseidon2,
    },
    "TOKEN",     // token symbol
    1000,        // max supply
    Some(10),    // token_supply (None defaults to 0)
)?;
```

The 4th argument is `token_supply: Option<u64>` (an explicit `None` is treated as `0`).

### 4. Build Contracts

See `integration/tests/counter_test.rs` in [project-template](https://github.com/0xMiden/project-template) for the pattern using `build_project_in_dir`.

### 5. Create Account with Storage

**Storage slot naming convention** (CRITICAL):
```
[component_package_or_name]::[snake_case(component_struct)]::[field_name]
```

Examples:
- Package `miden:counter-account`, component `CounterContract`, field `count_map` -> `miden_counter_account::counter_contract::count_map`
- Package `miden:bank-account`, component `BankAccount`, field `balances` -> `miden_bank_account::bank_account::balances`

Rule: Replace characters outside `[A-Za-z0-9_]` with `_` in the package or component name.

Populate `InitStorageData`, build the component from the compiled package, then register the account with `builder.add_account_from_builder(...)`:

```rust
let counter_storage_slot = counter_storage_slot()?;
let mut init_storage_data = InitStorageData::default();
init_storage_data.insert_map_entry(counter_storage_slot.clone(), COUNTER_STORAGE_KEY, 0_u64)?;

let counter_component =
    AccountComponent::from_package(&contract_package, &init_storage_data)?;
let counter_account = builder.add_account_from_builder(
    Auth::BasicAuth {
        auth_scheme: AuthScheme::Falcon512Poseidon2,
    },
    AccountBuilder::new([3_u8; 32])
        .account_type(AccountType::Public)
        .with_component(counter_component),
    AccountState::Exists,
)?;
```

> v0.15 account changes:
> - The old `AccountType` (`FungibleFaucet`/`NonFungibleFaucet`/`RegularAccountImmutableCode`/`RegularAccountUpdatableCode`) is **gone**. `AccountType` is now the visibility enum `{ Private, Public }`. Use `.account_type(AccountType::Public)` (or `::Private`).
> - `AccountStorageMode` was removed and `AccountBuilder` no longer has a `.storage_mode(...)` method. Visibility is set entirely via `.account_type(...)`.
> - Faucet-ness is no longer an account kind; it is determined by the installed components.

For a single-value contract slot (paired with `StorageValue<T>` on-chain) instead of a map:
```rust
let mut init_storage_data = InitStorageData::default();
init_storage_data.insert_value(
    "miden_bank_account::bank_account::initialized",
    0_u64,
)?;
```

### 6. Create Notes

See `integration/tests/counter_test.rs` in [project-template](https://github.com/0xMiden/project-template) for basic note creation with `RandomCoin`, `NoteScript::from_package`, and `NoteBuilder`.

For notes with assets and inputs:
```rust
use miden_client::{asset::FungibleAsset, crypto::RandomCoin, note::NoteScript, Felt};
use miden_standards::testing::note::NoteBuilder;

let mut note_rng =
    RandomCoin::new(NoteScript::from_package(note_package.as_ref())?.root().into());
let note = NoteBuilder::new(sender.id(), &mut note_rng)
    .package((*note_package).clone())
    .add_assets([FungibleAsset::new(faucet.id(), 50)?.into()])
    .note_storage([Felt::new(42), Felt::new(0)])?
    .build()?;
```

> v0.15: `NoteScript::root()` returns the `NoteScriptRoot` newtype, not a `Word`. `RandomCoin::new` needs a `Word`, so convert the root explicitly with `.into()` (equivalently `Word::from(...root())` or `...root().as_word()`). The bare `RandomCoin::new(... .root())` form from v0.14 no longer type-checks.

### 7. Add to MockChain and Build

See `integration/tests/counter_test.rs` in [project-template](https://github.com/0xMiden/project-template) for seeding the note and building the mock chain. `add_account_from_builder(...)` has already registered the account in the builder, so at this stage you usually only need to add notes.

### 8. Execute Transaction

The full execution flow is `build_tx_context` -> `execute()` -> `add_pending_executed_transaction()` -> `prove_next_block()` (see `integration/tests/counter_test.rs` in [project-template](https://github.com/0xMiden/project-template)). For the default MockChain flow, do not call `apply_delta()`; fetch refreshed state from `mock_chain.committed_account(...)` after the block is proven.

### 9. Execute with Transaction Script
```rust
use miden_client::transaction::TransactionScript;

let tx_script_package = Arc::new(build_project_in_dir(
    Path::new("../contracts/my-tx-script"),
    true,
)?);
let tx_script = TransactionScript::from_package(&tx_script_package)?;

let executed = mock_chain
    .build_tx_context(account.clone(), &[], &[])?
    .tx_script(tx_script)
    .build()?
    .execute()
    .await?;

mock_chain.add_pending_executed_transaction(&executed)?;
mock_chain.prove_next_block()?;

let updated_account = mock_chain.committed_account(account.id())?;
```

> v0.15: prefer `TransactionScript::from_package(&package)?` to build a script directly from the compiled package. If you instead use the (`#[doc(hidden)]`) `unwrap_program()`, note it returns a `Program` by value, so do **not** deref/clone it: `TransactionScript::new(tx_script_package.unwrap_program())`.

### 10. Verify Storage State

Read the committed account state with `mock_chain.committed_account(...)` after `prove_next_block()` and assert on the result (see `integration/tests/counter_test.rs` in [project-template](https://github.com/0xMiden/project-template)).

### 11. Verify Output Notes

**Important**: `add_output_note()` is only available on `MockChainBuilder` (before `build()`) — use it to seed the chain with existing notes. To verify output notes from a transaction, use `extend_expected_output_notes()` on `TxContextBuilder`:

```rust
use miden_client::{
    note::{Note, NoteAssets, PartialNoteMetadata, NoteRecipient},
    transaction::RawOutputNote,
};

// v0.15: Note::new takes a PartialNoteMetadata (sender + note_type + tag),
// not the old NoteMetadata. Build it with PartialNoteMetadata::new(sender, note_type).
let partial_metadata = PartialNoteMetadata::new(sender, note_type);
let expected_note = Note::new(expected_assets, partial_metadata, expected_recipient);

let tx_context = mock_chain
    .build_tx_context(account.id(), &[note.id()], &[])?
    .extend_expected_output_notes(vec![RawOutputNote::Full(expected_note)])
    .build()?;

// execute() will verify output notes match
let executed = tx_context.execute().await?;
```

> v0.15 note metadata split:
> - `Note::new(assets, partial_metadata, recipient)` now takes a `PartialNoteMetadata` (sender/type/tag only). The old `NoteMetadata` is no longer accepted here — there is no `Into` on the parameter.
> - For attachment-bearing notes use `Note::with_attachments(assets, partial_metadata, recipient, attachments)` (attachments are `NoteAttachments`).

## Multi-Transaction Test Pattern

For contracts requiring initialization before use, each step usually needs its own `execute()` → `add_pending_executed_transaction()` → `prove_next_block()` cycle. Fetch the committed account or note state from `mock_chain` between steps before building the next context.

`apply_delta()` is only needed in advanced same-block tests that intentionally reuse an in-memory `Account` across multiple transactions before proving the block.

See [miden-bank withdraw_test.rs](https://github.com/0xMiden/tutorials/blob/main/examples/miden-bank/integration/tests/withdraw_test.rs) for a complete multi-transaction test demonstrating: initialize bank → deposit assets → withdraw assets (3 sequential transactions with state verification between each step).

See [miden-bank deposit_test.rs](https://github.com/0xMiden/tutorials/blob/main/examples/miden-bank/integration/tests/deposit_test.rs) for an end-to-end asset-bearing note test.

## MockChain Block Numbering

Genesis is block 0. Each `prove_next_block()` advances the block number by 1. In contract code, `tx::get_block_number()` returns the **reference block** — the last proven block at the time the transaction started, not the block the transaction will be included in.

## Note Construction

Prefer `NoteBuilder` (or mirror its logic with compiled `.masp` package files) for creating notes in tests. Start from `NoteBuilder::new(sender.id(), &mut note_rng)`, then configure `.package(...)`, optional `.add_assets(...)`, optional `.note_storage(...)`, and finally `.build()`. See `integration/tests/counter_test.rs` in [project-template](https://github.com/0xMiden/project-template) for the working pattern.

## Asset-Bearing Note Example

To create a note that carries fungible assets in tests:

1. Create a `FungibleAsset` from a faucet ID and amount.
2. Seed a `RandomCoin` from `NoteScript::from_package(note_package.as_ref())?.root().into()` (the `.into()` converts the `NoteScriptRoot` to the `Word` that `RandomCoin::new` expects).
3. Pass the asset into `NoteBuilder::add_assets(...)` and any note inputs into `note_storage(...)`.
4. Finish with `.package((*note_package).clone()).build()?`.

The faucet must be set up first (see Step 3) and the sender wallet must hold sufficient assets (see Step 2).

## Key Dependencies

See `integration/Cargo.toml` in [project-template](https://github.com/0xMiden/project-template) for the dependency versions, and confirm it pins the 0.15 line (`miden-client`/`miden-protocol`/`miden-standards`/`miden-testing` 0.15.x, `miden-mast-package` 0.23.x). v0.14 artifacts and serialized `.masl`/`.masp` blobs do not round-trip into 0.15; re-assemble from source.

## Validation Checklist

- [ ] Test function is `async` and uses `#[tokio::test]`
- [ ] Auth uses `AuthScheme` (not the removed `AuthSchemeId`)
- [ ] `AccountBuilder` uses `.account_type(AccountType::Public | ::Private)` and no `.storage_mode(...)`
- [ ] Storage slot names follow `package_or_name::component_struct::field_name` pattern
- [ ] All contracts built before account/note creation
- [ ] Account storage seeded via `InitStorageData`
- [ ] `NoteScript::root()` converted with `.into()` before seeding `RandomCoin`
- [ ] `Note::new(...)` is passed a `PartialNoteMetadata` (not `NoteMetadata`)
- [ ] `prove_next_block()` called after `add_pending_executed_transaction()`
- [ ] Post-block assertions read state from `mock_chain.committed_account(...)` or other committed chain views
- [ ] Notes added to `MockChainBuilder` via `add_output_note(RawOutputNote::Full(...))` before `build()`
- [ ] Faucet set up before creating assets
