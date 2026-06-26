---
name: rust-sdk-testing-patterns
description: Guide to testing Miden smart contracts with MockChain (Miden v0.15). Covers test setup, contract building, account/note creation, transaction execution, storage verification, faucet setup, output note verification, block numbering, multi-transaction tests, and asset-bearing notes. Use when writing, editing, or debugging Miden integration tests.
---

# Miden Testing Patterns (MockChain)

These patterns target Miden **v0.15** (`miden-protocol`/`miden-standards`/`miden-testing` 0.15.x). Several types were renamed or removed in the v0.14 -> v0.15 migration (see the notes inline); the upstream `project-template` may still be on v0.14, so verify any copied snippet compiles under v0.15 before relying on it.

## Test File Setup

Tests go in `integration/tests/`. All tests are async and use MockChain for local execution without a network.

See `integration/tests/counter_test.rs` in [project-template](https://github.com/0xMiden/project-template) for a complete working test covering imports, MockChain setup, contract building, account creation with storage, note creation, transaction execution, and storage verification. The template lags the protocol release: confirm it is on the 0.15 line before copying APIs (older revisions use the removed v0.14 types `AccountStorageMode` and `AccountType::RegularAccountImmutableCode`, and a stale storage-slot naming format — see Step 5).

## Step-by-Step Test Pattern

### 1. Initialize MockChain Builder

Start from `let mut builder = MockChain::builder();` (see `integration/tests/counter_test.rs` in [project-template](https://github.com/0xMiden/project-template)).

### 2. Create Sender/Wallet Accounts

See `integration/tests/counter_test.rs` in [project-template](https://github.com/0xMiden/project-template) for the basic wallet pattern. For wallets with pre-funded assets, use `builder.add_existing_wallet_with_assets(Auth::BasicAuth { auth_scheme: AuthSchemeId::Falcon512Poseidon2 }, [FungibleAsset::new(faucet.id(), 100)?.into()])`.

> v0.15 auth-scheme naming: `miden_client::auth` re-exports the **same** protocol enum under **two** names — `AuthScheme` (the protocol name) and `AuthSchemeId` (an alias; this is the name the canonical tutorials use). Both compile; the field is `Auth::BasicAuth { auth_scheme }`. The variant `Falcon512Poseidon2` is the same on both. The examples here use `AuthSchemeId::Falcon512Poseidon2` to match the tutorials.

### 3. Set Up Faucets (for fungible assets)
```rust
let faucet = builder.add_existing_basic_faucet(
    Auth::BasicAuth {
        auth_scheme: AuthSchemeId::Falcon512Poseidon2,
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
<package_name>::<interface_segment>::<field_name>
```

The slot name is part of the on-chain storage ABI and is derived by the compiler's `#[component_storage]` macro, **not** from the Rust struct name:
- `<package_name>` is the **bare** package name (`[package].name`), with no `miden:` org prefix.
- `<interface_segment>` is the `[lib].namespace` **interface** segment — the text between the last `/` and the `@` in the namespace — snake_cased. Because it comes from the declared namespace, renaming the Rust struct cannot change the deployed slot name.
- `<field_name>` is the field name.

Characters outside `[A-Za-z0-9_]` are replaced with `_` in each segment.

Example: package `bank-account` with `[lib].namespace = "miden:bank-account/bank@0.1.0"` and struct `BankStorage` (fields `initialized`, `balances`) yields slots:
- `bank_account::bank::initialized`
- `bank_account::bank::balances`

Note the middle segment is `bank` (the interface segment), **not** `bank_storage` (the struct) and **not** `bank_account`, and there is no `miden_` org prefix. The `miden_counter_account::counter_contract::count_map` style seen in older `project-template` revisions is the **stale v0.14** format (those revisions have no `miden-project.toml` and pin `miden-client = "0.14"`); do not copy it for v0.15.

Populate `InitStorageData`, build the component from the compiled package, then register the account with `builder.add_account_from_builder(...)`:

```rust
let counter_storage_slot = counter_storage_slot()?;
let mut init_storage_data = InitStorageData::default();
init_storage_data.insert_map_entry(counter_storage_slot.clone(), COUNTER_STORAGE_KEY, 0_u64)?;

let counter_component =
    AccountComponent::from_package(&contract_package, &init_storage_data)?;
let counter_account = builder.add_account_from_builder(
    Auth::BasicAuth {
        auth_scheme: AuthSchemeId::Falcon512Poseidon2,
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
    "bank_account::bank::initialized",
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
    .note_storage([Felt::from(42_u32), Felt::from(0_u32)])?
    .build()?;
```

> v0.15: `NoteScript::root()` returns the `NoteScriptRoot` newtype, not a `Word`. `RandomCoin::new` needs a `Word`, so convert the root explicitly with `.into()` (equivalently `Word::from(...root())` or `...root().as_word()`). The bare `RandomCoin::new(... .root())` form from v0.14 no longer type-checks.

> v0.15: `Felt::new(u64)` is now **fallible** — it returns `Result<Felt, FeltFromIntError>` instead of an infallible value. `note_storage` takes `impl IntoIterator<Item = Felt>`, so a bare `[Felt::new(42), Felt::new(0)]` is `[Result<Felt, _>; 2]` and will not type-check. Prefer the infallible `Felt::from(42_u32)` for in-range literals (`From<u8>/From<u16>/From<u32>` are infallible); for a `u64` use `Felt::new(n)?` or `Felt::new_unchecked(n)`.

### 7. Add to MockChain and Build

See `integration/tests/counter_test.rs` in [project-template](https://github.com/0xMiden/project-template) for seeding the note and building the mock chain. `add_account_from_builder(...)` has already registered the account in the builder, so at this stage you usually only need to add notes.

### 8. Execute Transaction

The full execution flow is `build_tx_context` -> `execute()` -> `add_pending_executed_transaction()` -> `prove_next_block()` (see `integration/tests/counter_test.rs` in [project-template](https://github.com/0xMiden/project-template)). For the default MockChain flow, do not call `apply_delta()`; fetch refreshed state from `mock_chain.committed_account(...)` after the block is proven.

### 9. Execute with Transaction Script

A compiler project with `kind = "tx-script"` compiles to a `TransactionScript`-kind package, **not** an `Executable`. Because of that, `TransactionScript::from_package` and `Package::unwrap_program` do **not** apply to it: `from_package` calls `package.try_into_program()`, which returns `Err` for a non-executable package, and `unwrap_program` asserts the kind is `Executable` and **panics**. Build the script from the package's MAST forest plus its entry export instead:

```rust
use miden_client::transaction::TransactionScript;

let tx_script_package = Arc::new(build_project_in_dir(
    Path::new("../contracts/my-tx-script"),
    true,
)?);

// Locate the entry export ("main"/"run", or the sole export) and build from parts.
// See examples/miden-bank/integration/src/helpers.rs `build_tx_script_from_package`.
let tx_script = build_tx_script_from_package(tx_script_package.as_ref())?;

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

The helper essentially does `TransactionScript::from_parts(package.mast.mast_forest().clone(), entrypoint)` after finding the entry procedure's root in the MAST forest.

> v0.15: reserve `TransactionScript::from_package(&package)?` (and the `#[doc(hidden)]` `unwrap_program()`) for packages that are genuinely `Executable`. For `kind = "tx-script"` compiler packages, use `from_parts` / the `build_tx_script_from_package` helper as above — `from_package` returns an error and `unwrap_program()` panics on them.

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

`apply_delta()` is needed whenever you keep reading from / reusing the **same in-memory `Account`** across transactions — whether they land in the same block or in separate blocks. The canonical bank tests call `account.apply_delta(&executed.account_delta())?` after every `execute()` (each followed by `add_pending_executed_transaction` + `prove_next_block`) precisely so later local reads like `account.storage().get_map_item(...)` see the latest state. If you instead re-fetch via `mock_chain.committed_account(...)` after `prove_next_block()`, you can skip `apply_delta()` (as the single-transaction counter test does).

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
3. Pass the asset into `NoteBuilder::add_assets(...)` and any note inputs into `note_storage(...)`. `note_storage` wants `Item = Felt`; build each input with the infallible `Felt::from(_u32)` (not `Felt::new(u64)`, which is fallible in v0.15 — see Step 6).
4. Finish with `.package((*note_package).clone()).build()?`.

The faucet must be set up first (see Step 3) and the sender wallet must hold sufficient assets (see Step 2).

## Key Dependencies

See `integration/Cargo.toml` in [project-template](https://github.com/0xMiden/project-template) for the dependency versions, and confirm it pins the 0.15 line (`miden-client`/`miden-protocol`/`miden-standards`/`miden-testing` 0.15.x, `miden-mast-package` 0.23.x). v0.14 artifacts and serialized `.masl`/`.masp` blobs do not round-trip into 0.15; re-assemble from source.

## Validation Checklist

- [ ] Test function is `async` and uses `#[tokio::test]`
- [ ] Auth uses `AuthSchemeId::Falcon512Poseidon2` (or the equivalent `AuthScheme::Falcon512Poseidon2` — both name the same protocol enum)
- [ ] `AccountBuilder` uses `.account_type(AccountType::Public | ::Private)` and no `.storage_mode(...)`
- [ ] Storage slot names follow `<package_name>::<interface_segment>::<field_name>` (bare package name, `[lib].namespace` interface segment, e.g. `bank_account::bank::balances`) — not the stale v0.14 `miden_..._account::struct::field` format
- [ ] All contracts built before account/note creation
- [ ] Account storage seeded via `InitStorageData`
- [ ] `NoteScript::root()` converted with `.into()` before seeding `RandomCoin`
- [ ] Note-storage felts built with infallible `Felt::from(_u32)` (v0.15 `Felt::new(u64)` returns `Result`, so a bare `[Felt::new(..)]` array does not satisfy `Item = Felt`)
- [ ] `Note::new(...)` is passed a `PartialNoteMetadata` (not `NoteMetadata`)
- [ ] `kind = "tx-script"` packages built with `from_parts` / `build_tx_script_from_package` (not `from_package`/`unwrap_program`, which error/panic on them)
- [ ] `prove_next_block()` called after `add_pending_executed_transaction()`
- [ ] Post-block assertions read state from `mock_chain.committed_account(...)` (or `account.apply_delta(...)` is called when reusing an in-memory `Account` across transactions)
- [ ] Notes added to `MockChainBuilder` via `add_output_note(RawOutputNote::Full(...))` before `build()`
- [ ] Faucet set up before creating assets
