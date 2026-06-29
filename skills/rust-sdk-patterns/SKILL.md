---
name: rust-sdk-patterns
description: Complete guide to writing Miden smart contracts with the Rust SDK. Covers the three-part #[component_storage]/#[component] account-component pattern, #[note]/#[note_script] notes, #[tx_script] scripts, the #[account(...)] wrapper, storage patterns, native functions, asset handling, cross-component calls, P2ID note creation, and asset receiving via component methods. Use when writing, editing, or reviewing Miden Rust contract code.
---

# Miden Rust SDK Patterns

## Three Contract Types

### Account Component (three-part pattern)
Defines reusable logic and storage for accounts. Accounts are composed of one or more components.

In v0.15 an account component is written as **three parts** (`#[component]` no longer applies to a struct — the compiler rejects it with "`#[component]` no longer applies to structs; annotate the storage struct with `#[component_storage]` instead"):

1. `#[component_storage]` on the **storage struct** — declares typed `#[storage(...)]` fields and derives slot names.
2. `#[component]` on a **trait** — the component's exported API (this is the source of the generated WIT interface).
3. `#[component]` on the **`impl Trait for Storage`** block — the behavior, wired to the guest bindings.

```rust
#![no_std]
#![feature(alloc_error_handler)]
#[macro_use]
extern crate alloc;
use miden::*;

#[component_storage]
struct BankStorage {
    #[storage(description = "initialized")]
    initialized: StorageValue<Word>,
    #[storage(description = "balances")]
    balances: StorageMap<Word, Felt>,
}

#[component]
trait Bank {
    fn initialize(&mut self);
    fn deposit(&mut self, depositor: AccountId, deposit_asset: Asset);
}

#[component]
impl Bank for BankStorage {
    fn initialize(&mut self) { /* read/write self.initialized, etc. */ }
    fn deposit(&mut self, depositor: AccountId, deposit_asset: Asset) { /* ... */ }
}
```

Only the trait's methods are exported to WIT. Inherent (`impl BankStorage`) methods stay private to the contract — use them for helpers like key derivation.

See [miden-bank bank-account](https://github.com/0xMiden/tutorials/blob/078e2fb289fc299359ce44c3900bb9e59be93d40/examples/miden-bank/contracts/bank-account/src/lib.rs) for a complete working example demonstrating the three-part pattern, typed `StorageValue<Word>` / `StorageMap<Word, Felt>`, `get()`/`set()`, felt arithmetic, and private inherent helpers.

**Project metadata for accounts:** See `contracts/bank-account/miden-project.toml` in [miden-bank](https://github.com/0xMiden/tutorials/tree/078e2fb289fc299359ce44c3900bb9e59be93d40/examples/miden-bank/contracts/bank-account) for the `[lib] kind = "account-component"`, the `namespace`, and the `supported-types`. The `Cargo.toml` only needs `crate-type = ["cdylib"]` and the `miden` dependency.

### Note Script (`#[note]` / `#[note_script]`)
Executes when a note is consumed by an account. Can call component methods on the consuming account.

A note is two parts: a `#[note]` struct (the note inputs type) and a `#[note]` `impl` block containing exactly one `#[note_script]` entrypoint. The entrypoint takes `self` **by value**, exactly one `Word` argument, and optionally a single reference to an `#[account(...)]` wrapper (`&MyAccount` or `&mut MyAccount`). The consuming account is declared separately with `#[account(...)]`.

```rust
#![no_std]
#![feature(alloc_error_handler)]
use miden::*;

// The native (active) account this note runs against: exposes the
// bank-account `Bank` component's methods on the wrapper.
#[account(bank_account::Bank)]
pub struct Wallet;

#[note]
struct DepositNote;

#[note]
impl DepositNote {
    #[note_script]
    fn run(self, _arg: Word, account: &mut Wallet) {
        let depositor = active_note::get_sender();
        for asset in active_note::get_assets() {
            account.deposit(depositor, asset);
        }
    }
}
```

See [miden-bank deposit-note](https://github.com/0xMiden/tutorials/blob/078e2fb289fc299359ce44c3900bb9e59be93d40/examples/miden-bank/contracts/deposit-note/src/lib.rs) for a working example demonstrating `#[note]`, `#[note_script]`, the `#[account(...)]` wrapper, and a cross-component call.

**Project metadata for notes:** See `contracts/deposit-note/miden-project.toml` in [miden-bank](https://github.com/0xMiden/tutorials/tree/078e2fb289fc299359ce44c3900bb9e59be93d40/examples/miden-bank/contracts/deposit-note) for `[lib] kind = "note"`, the `namespace`, the path dependency on the called component, and the cross-component `[package.metadata.miden.dependencies]` WIT entry.

### Transaction Script (`#[tx_script]`)
One-off logic executed in the context of an account. Used for initialization, admin operations, etc.

`#[tx_script]` annotates a free `fn run`. Its signature is `fn run(arg: Word)` or `fn run(arg: Word, account: &mut MyAccount)` where `MyAccount` is an `#[account(...)]` wrapper. There is no `crate::bindings::Account` — you declare the account wrapper yourself and the macro instantiates it as the active account.

```rust
#![no_std]
#![feature(alloc_error_handler)]
use miden::*;

// The account this tx-script runs against: the bank-account `Bank` component.
#[account(bank_account::Bank)]
pub struct Wallet;

#[tx_script]
fn run(_arg: Word, account: &mut Wallet) {
    account.initialize();
}
```

See [miden-bank init-tx-script](https://github.com/0xMiden/tutorials/blob/078e2fb289fc299359ce44c3900bb9e59be93d40/examples/miden-bank/contracts/init-tx-script/src/lib.rs) for the working example.

**Project metadata for tx scripts:** Like a note, but `[lib] kind = "tx-script"` and `namespace = "miden:base/transaction-script@1.0.0"`. See `contracts/init-tx-script/miden-project.toml` in [miden-bank](https://github.com/0xMiden/tutorials/tree/078e2fb289fc299359ce44c3900bb9e59be93d40/examples/miden-bank/contracts/init-tx-script).

## Storage Slot Naming

Storage slot names are part of the on-chain storage ABI and are derived as:

```
<package_name_snake>::<interface_segment_snake>::<field_name>
```

The **middle segment is the interface segment of the `[lib].namespace`** in `miden-project.toml` (the part between the last `/` and `@`), snake-cased — **not** the snake-cased struct name. This deliberately decouples slot names from private Rust renames.

Example: package `bank-account` + `namespace = "miden:bank-account/bank@0.1.0"` + field `balances` derives slot `bank_account::bank::balances` (see [miden-bank deposit_test.rs](https://github.com/0xMiden/tutorials/blob/078e2fb289fc299359ce44c3900bb9e59be93d40/examples/miden-bank/integration/tests/deposit_test.rs), `bank_storage_slots()`). Note the version suffix (`@0.1.0`) is ignored so the slot name stays stable. Slots are derived from the slot name; there is no `slot(...)` attribute. See the rust-sdk-pitfalls skill (P5) for more on slot naming.

## Storage Types

| Type | Usage | Read | Write |
|------|-------|------|-------|
| `StorageValue<T>` | Single typed slot (flags, counters, IDs) | `.get() -> T` | `.set(T) -> T` |
| `StorageMap<K, V>` | Typed key-value mapping (balances, records) | `.get(K) -> V` | `.set(K, V) -> V` |

## Native Function Modules

| Module | Key Functions | Purpose |
|--------|--------------|---------|
| `native_account::` | `add_asset(Asset) -> Word`, `remove_asset(Asset) -> Word`, `incr_nonce() -> Felt`, `get_id() -> AccountId` | Modify current account vault/nonce |
| `active_account::` | `get_id() -> AccountId`, `get_balance(Word) -> Felt` | Query current account (`get_balance` takes the asset key word, not an AccountId) |
| `active_note::` | `get_storage() -> Vec<Felt>`, `get_assets() -> Vec<Asset>`, `get_sender() -> AccountId` | Query note being consumed |
| `note::` | `build_recipient(Word, Word, Vec<Felt>) -> Recipient` | Build note recipients from serial number, script root, and note storage |
| `output_note::` | `create(Tag, NoteType, Recipient) -> NoteIdx`, `add_asset(Asset, NoteIdx)` | Create output notes |
| `faucet::` | `create_fungible_asset(Felt) -> Asset`, `mint(Asset)`, `burn(Asset)` | Asset minting |
| `tx::` | `get_block_number() -> Felt`, `get_block_timestamp() -> Felt` | Transaction context |
| Intrinsics | `assert(Felt)`, `assertz(Felt)`, `assert_eq(Felt, Felt)` | Validation (`assert` fails unless the felt equals 1; `assertz` fails unless it equals 0) |

## Asset Handling

`Asset` is a two-word value (`key` + `value`):

**Constructor**: `Asset::new(key, value)` builds an Asset from its vault key word and value word (the arguments are `impl Into<Word>`, so e.g. `Asset::new(key_word, value_word)` or from `[Felt; 4]`).

See [miden-bank bank-account](https://github.com/0xMiden/tutorials/blob/078e2fb289fc299359ce44c3900bb9e59be93d40/examples/miden-bank/contracts/bank-account/src/lib.rs) for complete asset handling patterns including deposit, withdrawal, and balance tracking.

```rust
pub struct Asset {
    pub key: Word,
    pub value: Word,
}
```

For fungible assets, the amount lives in `asset.value[0]`. The asset class / vault identity lives in `asset.key`.

```rust
// Access fungible amount
let amount = asset.value[0];

// Keep the asset key if you need to persist or compare the asset class
let asset_key = asset.key;

// Add asset to account vault (only from component methods, not note scripts — see pitfall P11)
native_account::add_asset(asset);

// Remove asset from account vault (Asset is Copy, no clone needed)
native_account::remove_asset(asset);
```

## P2ID Output Note Creation

To send assets to another account, create a P2ID (Pay-to-ID) output note. See [miden-bank bank-account](https://github.com/0xMiden/tutorials/blob/078e2fb289fc299359ce44c3900bb9e59be93d40/examples/miden-bank/contracts/bank-account/src/lib.rs) `create_p2id_note()` for a complete working implementation (builds the recipient with `note::build_recipient`, creates the note with `output_note::create`, then `native_account::remove_asset` + `output_note::add_asset`).

## Cross-Component Dependencies

To call another component's methods from a note or tx script, declare the dependency in your `miden-project.toml` in **two places**:

- `[dependencies]` — a normal path (or registry) dependency on the component crate.
- `[package.metadata.miden.dependencies]` — the generated WIT for the component, e.g. `bank-account = { wit = "../bank-account/target/generated-wit/" }`. The WIT is produced by building the dependency component first.

See `contracts/deposit-note/miden-project.toml` in [miden-bank](https://github.com/0xMiden/tutorials/tree/078e2fb289fc299359ce44c3900bb9e59be93d40/examples/miden-bank/contracts/deposit-note) for a working example showing both sections.

Then expose the dependency's methods on the consuming account by declaring an `#[account(package::Interface)]` wrapper (e.g. `#[account(bank_account::Bank)] pub struct Wallet;`) and calling methods on the injected `account` parameter. The package name is the dependency's Rust-style name (`-` replaced with `_`) and `Interface` is its exported WIT interface in UpperCamelCase. See `contracts/deposit-note/src/lib.rs` in [miden-bank](https://github.com/0xMiden/tutorials/blob/078e2fb289fc299359ce44c3900bb9e59be93d40/examples/miden-bank/contracts/deposit-note/src/lib.rs).

## Common Type Conversions

```rust
// Felt from integer
let f = felt!(42);                     // preferred for literals in contract code
let f = Felt::new(42).unwrap();        // fallible: Felt::new returns Result<Felt, _> in v0.15
let f = Felt::new_unchecked(42);       // infallible, non-reducing form
let f = Felt::from_u32(42);            // infallible (u32 always fits)
let f = Felt::from_canonical_checked(42).unwrap(); // returns Option<Felt>

// Word from Felts
let w = Word::from([f0, f1, f2, f3]);
let w = Word::new([f0, f1, f2, f3]);
let w = Word::from([0_u32, 0, 0, 1]);
let w = Word::try_from([0_u64, 0, 0, 1]).unwrap();

// Inspect a Word
let limbs: [Felt; 4] = w.into_elements();
let bytes: [u8; 32] = w.as_bytes();
let hex = w.to_hex();

// Felt to u64 (for comparisons and arithmetic safety)
let n: u64 = f.as_canonical_u64();
```

## No-std Requirements

Every contract file must start with `#![no_std]` and `#![feature(alloc_error_handler)]`. See any contract under `contracts/` in [miden-bank](https://github.com/0xMiden/tutorials/tree/078e2fb289fc299359ce44c3900bb9e59be93d40/examples/miden-bank/contracts) for the pattern.

If you need heap allocation (Vec, String, etc.):
```rust
extern crate alloc;
use alloc::vec::Vec;
```

The bank-account contract uses `#[macro_use] extern crate alloc;` so the `vec!` macro is available (it builds note-recipient inputs with `vec![...]`).

## Asset Receiving via Component Methods

Note scripts cannot call `native_account::add_asset()` directly (see pitfall P11). The canonical pattern is for an account component to expose a public (trait) method that wraps `native_account::add_asset()`, and the note script calls that method through the `#[account(...)]` wrapper.

See [miden-bank bank-account deposit()](https://github.com/0xMiden/tutorials/blob/078e2fb289fc299359ce44c3900bb9e59be93d40/examples/miden-bank/contracts/bank-account/src/lib.rs) for the component side: the `deposit()` trait method validates the deposit, updates storage, and calls `native_account::add_asset()`.

See [miden-bank deposit-note](https://github.com/0xMiden/tutorials/blob/078e2fb289fc299359ce44c3900bb9e59be93d40/examples/miden-bank/contracts/deposit-note/src/lib.rs) for the note side: the note declares `#[account(bank_account::Bank)] pub struct Wallet;` and, inside `#[note_script] fn run(self, _arg: Word, account: &mut Wallet)`, calls `account.deposit(depositor, asset)` on that wrapper. It is **not** a free `bank_account::deposit()` call.

## Validation Checklist

- [ ] `#![no_std]` and `#![feature(alloc_error_handler)]` at top of every contract
- [ ] Account components use the three-part pattern: `#[component_storage]` struct + `#[component]` trait + `#[component]` impl (never `#[component]` on a struct)
- [ ] `crate-type = ["cdylib"]` in `Cargo.toml`
- [ ] Correct `[lib] kind` in `miden-project.toml` (`account-component` / `note` / `tx-script`) with the matching `namespace`
- [ ] Typed storage uses `StorageValue<T>` / `StorageMap<K, V>` with `get()` / `set()`; slot names derive from `<package>::<namespace-interface>::<field>`
- [ ] Notes/tx-scripts that call a component declare an `#[account(package::Interface)]` wrapper and call methods on the injected `account`
- [ ] Cross-component deps declared in `miden-project.toml` under both `[dependencies]` (path) and `[package.metadata.miden.dependencies]` (wit)
- [ ] Felt arithmetic validated before subtraction (see rust-sdk-pitfalls skill)
- [ ] Felt comparisons use `.as_canonical_u64()` (see rust-sdk-pitfalls skill)
