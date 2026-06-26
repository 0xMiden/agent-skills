---
name: rust-sdk-patterns
description: Complete guide to writing Miden smart contracts with the Rust SDK. Covers #[component], #[note], #[tx_script] macros, storage patterns, native functions, asset handling, cross-component calls, P2ID note creation, and asset receiving via component methods. Use when writing, editing, or reviewing Miden Rust contract code.
---

# Miden Rust SDK Patterns

## Three Contract Types

### Account Component (`#[component]`)
Defines reusable logic and storage for accounts. Accounts are composed of one or more components.

See `contracts/counter-account/src/lib.rs` in [project-template](https://github.com/0xMiden/project-template) for a working example demonstrating `#[component]`, typed `StorageMap<Word, Felt>`, `get()`/`set()`, and felt arithmetic.

**Cargo.toml for accounts:** See `contracts/counter-account/Cargo.toml` in [project-template](https://github.com/0xMiden/project-template) for the required `crate-type`, `miden` dependency, `component` metadata, and `project-kind`.

### Note Script (`#[note]`)
Executes when a note is consumed by an account. Can call component methods on the consuming account.

See `contracts/increment-note/src/lib.rs` in [project-template](https://github.com/0xMiden/project-template) for a working example demonstrating `#[note]`, `#[note_script]`, and cross-component calls.

**Cargo.toml for notes:** See `contracts/increment-note/Cargo.toml` in [project-template](https://github.com/0xMiden/project-template) for the required `miden` deps, cross-component dependencies, wit deps, and `project-kind = "note-script"`.

### Transaction Script (`#[tx_script]`)
One-off logic executed in the context of an account. Used for initialization, admin operations, etc.

```rust
#![no_std]
#![feature(alloc_error_handler)]
use miden::*;
use crate::bindings::Account;

#[tx_script]
fn run(_arg: Word, account: &mut Account) {
    account.initialize();
}
```

**In your `Cargo.toml`:** Same as an account component, but with `project-kind = "tx-script"`.

## Storage Types

| Type | Usage | Read | Write |
|------|-------|------|-------|
| `StorageValue<T>` | Single typed slot (flags, counters, IDs) | `.get() -> T` | `.set(T) -> T` |
| `StorageMap<K, V>` | Typed key-value mapping (balances, records) | `.get(K) -> V` | `.set(K, V) -> V` |

## Native Function Modules

| Module | Key Functions | Purpose |
|--------|--------------|---------|
| `native_account::` | `add_asset(Asset)`, `remove_asset(Asset)`, `incr_nonce()` | Modify account vault/nonce |
| `active_account::` | `get_id() -> AccountId`, `get_balance(Word) -> Felt` | Query current account (`get_balance` takes the asset key word, not an AccountId) |
| `active_note::` | `get_storage() -> Vec<Felt>`, `get_assets() -> Vec<Asset>`, `get_sender() -> AccountId` | Query note being consumed |
| `note::` | `build_recipient(Word, Word, Vec<Felt>) -> Recipient` | Build note recipients from serial number, script root, and note storage |
| `output_note::` | `create(Tag, NoteType, Recipient) -> NoteIdx`, `add_asset(Asset, NoteIdx)` | Create output notes |
| `faucet::` | `create_fungible_asset(Felt) -> Asset`, `mint(Asset)`, `burn(Asset)` | Asset minting |
| `tx::` | `get_block_number() -> Felt`, `get_block_timestamp() -> Felt` | Transaction context |
| Intrinsics | `assert(Felt)`, `assertz(Felt)`, `assert_eq(Felt, Felt)` | Validation (`assert` fails unless the felt equals 1) |

## Asset Handling

`Asset` is a two-word value (`key` + `value`):

**Constructor**: `Asset::new(key, value)` builds an Asset from its vault key word and value word (e.g. `Asset::new(key_word, value_word)`).

See [miden-bank bank-account](https://github.com/0xMiden/tutorials/blob/main/examples/miden-bank/contracts/bank-account/src/lib.rs) for complete asset handling patterns including deposit, withdrawal, and balance tracking.

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

To send assets to another account, create a P2ID (Pay-to-ID) output note. See [miden-bank bank-account](https://github.com/0xMiden/tutorials/blob/main/examples/miden-bank/contracts/bank-account/src/lib.rs) `create_p2id_note()` for a complete working implementation.

## Cross-Component Dependencies

To call another component's methods from a note or tx script, two sections are needed in your `Cargo.toml`. See `contracts/increment-note/Cargo.toml` in [project-template](https://github.com/0xMiden/project-template) for a working example showing both `[package.metadata.miden.dependencies]` and `[package.metadata.component.target.dependencies]`.

Then import the bindings in your Rust code. See `contracts/increment-note/src/lib.rs` in [project-template](https://github.com/0xMiden/project-template) (around line 13) for the import pattern: `use crate::bindings::miden::target_component::target_component;`

## Common Type Conversions

```rust
// Felt from integer
let f = felt!(42);                     // preferred for literals in contract code
let f = Felt::new(42).unwrap();        // fallible: Felt::new returns Result<Felt, _> in v0.15
let f = Felt::new_unchecked(42);       // infallible, non-reducing form
let f = Felt::from_u32(42);            // infallible (u32 always fits)
let f = Felt::from_canonical_checked(42).unwrap();

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

Every contract file must start with `#![no_std]` and `#![feature(alloc_error_handler)]`. See any contract under `contracts/` in [project-template](https://github.com/0xMiden/project-template) for the pattern.

If you need heap allocation (Vec, String, etc.):
```rust
extern crate alloc;
use alloc::vec::Vec;
```

## Asset Receiving via Component Methods

Note scripts cannot call `native_account::add_asset()` directly (see pitfall P11). The canonical pattern is for an account component to expose a public method that wraps `native_account::add_asset()`, and note scripts call that method via cross-component bindings.

See [miden-bank bank-account deposit()](https://github.com/0xMiden/tutorials/blob/main/examples/miden-bank/contracts/bank-account/src/lib.rs) for the component side: the `deposit()` method validates the deposit, updates storage, and calls `native_account::add_asset()`.

See [miden-bank deposit-note](https://github.com/0xMiden/tutorials/blob/main/examples/miden-bank/contracts/deposit-note/src/lib.rs) for the note side: the note script calls `bank_account::deposit()` via generated bindings.

## Validation Checklist

- [ ] `#![no_std]` and `#![feature(alloc_error_handler)]` at top of every contract
- [ ] `crate-type = ["cdylib"]` in your `Cargo.toml`
- [ ] Correct `project-kind` in `[package.metadata.miden]`
- [ ] Typed storage uses `StorageValue<T>` / `StorageMap<K, V>` with `get()` / `set()`
- [ ] Cross-component deps in both `[package.metadata.miden.dependencies]` and `[package.metadata.component.target.dependencies]`
- [ ] Felt arithmetic validated before subtraction (see rust-sdk-pitfalls skill)
- [ ] Felt comparisons use `.as_canonical_u64()` (see rust-sdk-pitfalls skill)
