---
name: rust-sdk-testing-patterns
description: Guide to testing Miden smart contracts with MockChain (Miden v0.15). Covers test setup, contract building, account/note creation, transaction execution, storage verification, faucet setup, output note verification, block numbering, multi-transaction tests, and asset-bearing notes. Use when writing, editing, or debugging Miden integration tests.
---

# Miden Testing Patterns (MockChain)

These patterns target Miden **v0.15** (`miden-protocol`/`miden-standards`/`miden-testing` 0.15.x). Several types were renamed or removed in the migration to v0.15 (see the notes inline).

The **authoritative working v0.15 example** is the `miden-bank` tutorial (`examples/miden-bank/integration/tests/{init_test,deposit_test,withdraw_test}.rs` plus `integration/src/helpers.rs` in [miden-bank](https://github.com/0xMiden/tutorials/tree/078e2fb289fc299359ce44c3900bb9e59be93d40/examples/miden-bank/integration)). Prefer it over the `project-template` counter test: the `project-template` may still be on the pre-v0.15 line, and its `counter_test.rs` uses the removed pre-v0.15 types `AccountStorageMode` and `AccountType::RegularAccountImmutableCode`, the removed `.storage_mode(...)` builder method, a stale storage-slot naming format, and the bare `RandomCoin::new(... .root())` form — none of which compile under v0.15. Verify any copied snippet compiles under v0.15 before relying on it.

## Test File Setup

Tests go in `integration/tests/`. All tests are async and use MockChain for local execution without a network.

See [miden-bank deposit_test.rs](https://github.com/0xMiden/tutorials/blob/078e2fb289fc299359ce44c3900bb9e59be93d40/examples/miden-bank/integration/tests/deposit_test.rs) for a complete working test covering imports, MockChain setup, contract building, account creation with storage, note creation, transaction execution, and storage verification. The v0.15 imports it relies on are:

```rust
use miden_client::{
    account::{component::{InitStorageData, StorageValueName}, StorageSlotName},
    auth::AuthSchemeId,
    note::{Note, NoteAssets, NoteType, PartialNoteMetadata},
    transaction::RawOutputNote,
    Felt, Word,
};
use miden_client::asset::{Asset, FungibleAsset};
use miden_testing::{Auth, MockChain};
```

## Step-by-Step Test Pattern

### 1. Initialize MockChain Builder

Start from `let mut builder = MockChain::builder();` (see `deposit_test.rs` in [miden-bank](https://github.com/0xMiden/tutorials/blob/078e2fb289fc299359ce44c3900bb9e59be93d40/examples/miden-bank/integration/tests/deposit_test.rs)).

### 2. Create Sender/Wallet Accounts

For a bare wallet use `builder.add_existing_wallet(Auth::BasicAuth { auth_scheme: AuthSchemeId::Falcon512Poseidon2 })`. For wallets with pre-funded assets, use `builder.add_existing_wallet_with_assets(Auth::BasicAuth { auth_scheme: AuthSchemeId::Falcon512Poseidon2 }, [FungibleAsset::new(faucet.id(), 100)?.into()])` (see `deposit_test.rs`).

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

Build each project from its directory with the `build_project_in_dir` helper, e.g. `let bank_package = Arc::new(build_project_in_dir(Path::new("../contracts/bank-account"), true)?);` (see `deposit_test.rs` and `integration/src/helpers.rs::build_project_in_dir`).

### 5. Create Account with Storage

**Storage slot naming convention** (CRITICAL):
```
<package_name>::<interface_segment>::<field_name>
```

The slot name is part of the on-chain storage ABI and is derived by the compiler's `#[component_storage]` macro, **not** from the Rust struct name:
- `<package_name>` is the **bare** package name (`[package].name`), with no `miden:` org prefix.
- `<interface_segment>` is the `[lib].namespace` **interface** segment — the text between the last `/` and the `@` in the namespace — snake_cased. Because it comes from the declared namespace, renaming the Rust struct cannot change the deployed slot name.
- `<field_name>` is the Rust storage field's identifier (not its `description`).

Characters outside `[A-Za-z0-9_]` are replaced with `_` in each segment.

Example: package `bank-account` with `[lib].namespace = "miden:bank-account/bank@0.1.0"` and storage struct `BankStorage` (fields `initialized`, `balances`) yields slots:
- `bank_account::bank::initialized`
- `bank_account::bank::balances`

Note the middle segment is `bank` (the interface segment), **not** `bank_storage` (the struct) and **not** `bank_account`, and there is no `miden_` org prefix. The `miden_counter_account::counter_contract::count_map` style seen in older `project-template` revisions is the **stale pre-v0.15** format (those revisions have no `miden-project.toml` and pin a pre-v0.15 `miden-client`); do not copy it for v0.15.

The component's storage is declared with the v0.15 three-part component macro (`#[component_storage]` struct + `#[component]` trait + `#[component]` impl); the storage struct, not the trait, carries the `#[storage]` fields the slot names derive from. See the `rust-sdk-patterns` skill for the contract side.

**Authoritative pattern** (from `deposit_test.rs`/`init_test.rs`): build a `StorageSlotName`, seed any value slot that has no schema default via `InitStorageData::insert_value`, build the account with `create_testing_account_from_package`, then register it with `builder.add_account(...)`:

```rust
// The bank's `initialized` value slot is `StorageValue<Word>` with no schema default,
// so `AccountComponent::from_package` requires it to be seeded (here a zero Word =
// uninitialized) or it errors with `InitValueNotProvided`. The `balances` map defaults
// to empty and needs no entry.
let initialized_slot = StorageSlotName::new("bank_account::bank::initialized")?;

let mut init_storage_data = InitStorageData::default();
init_storage_data.insert_value(
    StorageValueName::from_slot_name(&initialized_slot),
    Word::default(),
)?;

// `create_testing_account_from_package` builds the AccountComponent from the package +
// init data, then an existing account via `AccountBuilder::new([3u8; 32])
// .account_type(AccountType::Public).with_component(..).with_auth_component(NoAuth)
// .build_existing()`. See `integration/src/helpers.rs`.
let bank_account = create_testing_account_from_package(
    bank_package.clone(),
    AccountCreationConfig { init_storage_data, ..Default::default() },
)?;

builder.add_account(bank_account.clone())?;
```

> v0.15 storage-seeding footgun: `InitStorageData::insert_value(name, value)` takes `value: impl Into<WordValue>`. The numeric `From` impls (`u8`/`u16`/`u32`/`u64`) produce a `WordValue::Atomic(string)` that the slot's schema parses — **not** a felt-positioned `Word`. Only `From<Felt>` yields `[felt, 0, 0, 0]` and `From<Word>`/`From<[Felt; 4]>`/`From<[u32; 4]>` are fully-typed words. For a `StorageValue<Word>` slot (like the bank's `initialized` flag, whose contract reads index `[0]`), seed a `Word` (`Word::default()` for zero). The `insert_value` doc comment claiming `u64` becomes `[0,0,0,felt]` is stale; the code produces an atomic string.

> v0.15 account changes:
> - The old `AccountType` (`FungibleFaucet`/`NonFungibleFaucet`/`RegularAccountImmutableCode`/`RegularAccountUpdatableCode`) is **gone**. `AccountType` is now the visibility enum `{ Private, Public }`. Use `.account_type(AccountType::Public)` (or `::Private`).
> - `AccountStorageMode` was removed and `AccountBuilder` no longer has a `.storage_mode(...)` method. Visibility is set entirely via `.account_type(...)`.
> - Faucet-ness is no longer an account kind; it is determined by the installed components.

If you build the account inline instead of via the helper, `builder.add_account_from_builder(auth, account_builder, AccountState::Exists)` is the v0.15-valid equivalent — it consumes an `AccountBuilder` (configured with `.account_type(AccountType::Public)` and `.with_component(...)`) and registers it. For map slots, seed entries with `init_storage_data.insert_map_entry(slot_name, key, value)?` (three args: `slot_name: impl TryInto<StorageSlotName>`, `key`, `value`).

### 6. Create Notes

The authoritative tests build notes with the `create_testing_note_from_package(package, sender_id, NoteCreationConfig { .. })` helper, which wraps `NoteBuilder` and derives a deterministic serial number from the note-script digest (see `integration/src/helpers.rs`).

If you build a note by hand with `NoteBuilder`, seed the `RandomCoin` from the note-script root:
```rust
use miden_client::{asset::FungibleAsset, crypto::RandomCoin, note::NoteScript, Felt, Word};
use miden_standards::testing::note::NoteBuilder;

let note_script = NoteScript::from_package(note_package.as_ref())?;
let mut note_rng = RandomCoin::new(Word::from(note_script.root()));
let note = NoteBuilder::new(sender.id(), &mut note_rng)
    .package((*note_package).clone())
    .add_assets([FungibleAsset::new(faucet.id(), 50)?.into()])
    .note_storage([Felt::from(42_u32), Felt::from(0_u32)])?
    .build()?;
```

> v0.15: `NoteScript::root()` returns the `NoteScriptRoot` newtype, not a `Word`. `RandomCoin::new` needs a `Word`, so convert the root explicitly with `Word::from(...root())` (equivalently `...root().into()` or `...root().as_word()`). The bare `RandomCoin::new(... .root())` form no longer type-checks under v0.15.

> v0.15: `Felt::new(u64)` is now **fallible** — it returns `Result<Felt, FeltFromIntError>` instead of an infallible value. `note_storage` takes `impl IntoIterator<Item = Felt>` and itself returns `Result`, so a bare `[Felt::new(42), Felt::new(0)]` is `[Result<Felt, _>; 2]` and will not type-check. Prefer the infallible `Felt::from(42_u32)` for in-range literals (`From<u8>/From<u16>/From<u32>` are infallible); for a `u64` use `Felt::new(n)?` or `Felt::new_unchecked(n)` (the form the bank withdraw test uses for note-storage inputs).

### 7. Add to MockChain and Build

Register accounts (`builder.add_account(...)`) and seed notes (`builder.add_output_note(RawOutputNote::Full(note.clone()))`) on the builder, then `let mut mock_chain = builder.build()?;` (see `deposit_test.rs`).

### 8. Execute Transaction

The full execution flow is `build_tx_context` -> `execute()` -> `add_pending_executed_transaction()` -> `prove_next_block()` (see `deposit_test.rs`). The bank tests `apply_delta()` onto the in-memory `Account` after each `execute()` so later local reads see the new state; if you instead re-fetch via `mock_chain.committed_account(...)` after the block is proven you can skip `apply_delta()` (see the multi-transaction note below).

### 9. Execute with Transaction Script

A compiler project with `kind = "tx-script"` compiles to a `TransactionScript`-kind package, **not** an `Executable`. Because of that, `TransactionScript::from_package` and `Package::unwrap_program` do **not** apply to it: `from_package` calls `package.try_into_program()`, which returns `Err` for a non-executable package, and `unwrap_program` asserts the kind is `Executable` and **panics**. Build the script from the package's MAST forest plus its entry export instead:

```rust
use miden_client::transaction::TransactionScript;

let tx_script_package = Arc::new(build_project_in_dir(
    Path::new("../contracts/init-tx-script"),
    true,
)?);

// Locate the entry export ("main"/"run", or the sole export) and build from parts.
// See examples/miden-bank/integration/src/helpers.rs `build_tx_script_from_package`.
let tx_script = build_tx_script_from_package(tx_script_package.as_ref())?;

let executed = mock_chain
    .build_tx_context(account.id(), &[], &[])?
    .tx_script(tx_script)
    .build()?
    .execute()
    .await?;

mock_chain.add_pending_executed_transaction(&executed)?;
mock_chain.prove_next_block()?;

let updated_account = mock_chain.committed_account(account.id())?;
```

The helper essentially does `TransactionScript::from_parts(package.mast.mast_forest().clone(), entrypoint)` after finding the entry procedure's root in the MAST forest.

> v0.15: reserve `TransactionScript::from_package(&package)?` (and the `#[doc(hidden)]` `unwrap_program()`) for packages that are genuinely `Executable`. For `kind = "tx-script"` compiler packages (e.g. the bank's `init-tx-script`, whose `miden-project.toml` declares `kind = "tx-script"`), use `from_parts` / the `build_tx_script_from_package` helper as above — `from_package` returns an error and `unwrap_program()` panics on them.

### 10. Verify Storage State

Read state with `account.storage().get_item(&slot)` / `.get_map_item(&slot, key)` on an in-memory `Account` you keep `apply_delta`-current (the bank tests' approach), or re-fetch the committed account with `mock_chain.committed_account(account.id())?` after `prove_next_block()` and assert on its storage. Map values come back as scalar words in `[value, 0, 0, 0]` layout (see `deposit_test.rs` and `init_test.rs`).

### 11. Verify Output Notes

**Important**: `add_output_note()` is only available on `MockChainBuilder` (before `build()`) — use it to seed the chain with existing notes. To verify output notes from a transaction, use `extend_expected_output_notes()` on `TxContextBuilder`:

```rust
use miden_client::{
    note::{Note, NoteType, PartialNoteMetadata},
    transaction::RawOutputNote,
};

// v0.15: Note::new takes a PartialNoteMetadata (sender + note_type + tag),
// not the old NoteMetadata. Build it with PartialNoteMetadata::new(sender, note_type),
// then optionally `.with_tag(tag)` (the tag defaults to NoteTag::default()).
let partial_metadata = PartialNoteMetadata::new(sender, NoteType::Public).with_tag(tag);
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

`apply_delta()` is needed whenever you keep reading from / reusing the **same in-memory `Account`** across transactions — whether they land in the same block or in separate blocks. The canonical bank tests call `bank_account.apply_delta(&executed.account_delta())?` after every `execute()` (each followed by `add_pending_executed_transaction` + `prove_next_block`) precisely so later local reads like `bank_account.storage().get_map_item(...)` see the latest state. If you instead re-fetch via `mock_chain.committed_account(...)` after `prove_next_block()`, you can skip `apply_delta()`.

See [miden-bank init_test.rs](https://github.com/0xMiden/tutorials/blob/078e2fb289fc299359ce44c3900bb9e59be93d40/examples/miden-bank/integration/tests/init_test.rs) for the init-via-tx-script flow and pre/post storage assertions, and [miden-bank withdraw_test.rs](https://github.com/0xMiden/tutorials/blob/078e2fb289fc299359ce44c3900bb9e59be93d40/examples/miden-bank/integration/tests/withdraw_test.rs) for a complete multi-transaction test demonstrating: initialize bank → deposit assets → withdraw assets (sequential transactions with state verification between steps, plus expected P2ID output-note verification).

See [miden-bank deposit_test.rs](https://github.com/0xMiden/tutorials/blob/078e2fb289fc299359ce44c3900bb9e59be93d40/examples/miden-bank/integration/tests/deposit_test.rs) for an end-to-end asset-bearing note test, including the negative `deposit_exceeds_max_should_fail` / `deposit_without_init_should_fail` cases that assert `tx_context.execute().await.is_err()`.

## MockChain Block Numbering

Genesis is block 0. Each `prove_next_block()` advances the block number by 1. In contract code, `tx::get_block_number()` returns the **reference block** — the last proven block at the time the transaction started, not the block the transaction will be included in.

## Note Construction

Prefer the `create_testing_note_from_package` / `create_note_from_package` helpers (which wrap `NoteBuilder`) for creating notes in tests. If you use `NoteBuilder` directly, start from `NoteBuilder::new(sender.id(), &mut note_rng)`, then configure `.package(...)`, optional `.note_type(...)`, optional `.tag(...)`, optional `.add_assets(...)`, optional `.note_storage(...)?`, optional `.serial_number(...)`, and finally `.build()?`. Seed the `RandomCoin` from `Word::from(NoteScript::from_package(note_package.as_ref())?.root())` (see Step 6).

## Asset-Bearing Note Example

To create a note that carries fungible assets in tests:

1. Create a `FungibleAsset` from a faucet ID and amount, e.g. `FungibleAsset::new(faucet.id(), 50)?`, and wrap into `NoteAssets::new(vec![Asset::Fungible(asset)])?` (or pass via `NoteBuilder::add_assets`).
2. Seed a `RandomCoin` from `Word::from(NoteScript::from_package(note_package.as_ref())?.root())` (the conversion turns the `NoteScriptRoot` into the `Word` that `RandomCoin::new` expects).
3. Pass the asset into the note and any note inputs into `note_storage(...)?`. `note_storage` wants `Item = Felt`; build each input with the infallible `Felt::from(_u32)` for in-range literals (not `Felt::new(u64)`, which is fallible in v0.15), or `Felt::new_unchecked(n)` for u64 inputs (see Step 6).
4. Finish with `.package((*note_package).clone()).build()?` (or use `create_testing_note_from_package` with a `NoteCreationConfig { assets, storage, .. }`).

The faucet must be set up first (see Step 3) and the sender wallet must hold sufficient assets (see Step 2).

## Key Dependencies

See `integration/Cargo.toml` in [miden-bank](https://github.com/0xMiden/tutorials/blob/078e2fb289fc299359ce44c3900bb9e59be93d40/examples/miden-bank/integration/Cargo.toml) for the dependency versions, and confirm it pins the 0.15 line (`miden-client`/`miden-standards`/`miden-testing` 0.15.x, `miden-mast-package` 0.23.x). pre-v0.15 artifacts and serialized `.masl`/`.masp` blobs do not round-trip into v0.15; re-assemble from source.

## Validation Checklist

- [ ] Test function is `async` and uses `#[tokio::test]`
- [ ] Auth uses `AuthSchemeId::Falcon512Poseidon2` (or the equivalent `AuthScheme::Falcon512Poseidon2` — both name the same protocol enum)
- [ ] `AccountBuilder` uses `.account_type(AccountType::Public | ::Private)` and no `.storage_mode(...)` / no `AccountStorageMode`
- [ ] Storage slot names follow `<package_name>::<interface_segment>::<field_name>` (bare package name, `[lib].namespace` interface segment, e.g. `bank_account::bank::balances`) — not the stale pre-v0.15 `miden_..._account::struct::field` format
- [ ] Value slots without a schema default are seeded via `InitStorageData::insert_value(StorageValueName::from_slot_name(&slot), ..)`; `StorageValue<Word>` slots get a `Word` (e.g. `Word::default()`), not a bare integer (numeric `Into<WordValue>` yields an atomic string, not a felt-positioned word)
- [ ] All contracts built before account/note creation
- [ ] `NoteScript::root()` converted with `Word::from(...)` before seeding `RandomCoin`
- [ ] Note-storage felts built with infallible `Felt::from(_u32)` or `Felt::new_unchecked(_u64)` (v0.15 `Felt::new(u64)` returns `Result`, so a bare `[Felt::new(..)]` array does not satisfy `Item = Felt`)
- [ ] `Note::new(...)` is passed a `PartialNoteMetadata` (not `NoteMetadata`)
- [ ] `kind = "tx-script"` packages built with `from_parts` / `build_tx_script_from_package` (not `from_package`/`unwrap_program`, which error/panic on them)
- [ ] `prove_next_block()` called after `add_pending_executed_transaction()`
- [ ] Post-block assertions read state from `mock_chain.committed_account(...)` (or `account.apply_delta(...)` is called when reusing an in-memory `Account` across transactions)
- [ ] Notes added to `MockChainBuilder` via `add_output_note(RawOutputNote::Full(...))` before `build()`
- [ ] Faucet set up before creating assets
