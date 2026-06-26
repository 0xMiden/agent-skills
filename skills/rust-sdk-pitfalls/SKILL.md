---
name: rust-sdk-pitfalls
description: Critical pitfalls and safety rules for Miden Rust SDK development. Covers felt arithmetic security, comparison operators, argument limits, storage naming, no-std setup, asset layout, P2ID roots, NoteType construction, note-to-component call boundaries, and note input immutability. Use when reviewing, debugging, or writing Miden contract code.
---

# Miden SDK Pitfalls

## P1: Felt Arithmetic is Modular (SECURITY CRITICAL)

**Severity**: Critical — can cause loss of funds

Felt subtraction wraps around the prime field modulus (p = 2^64 - 2^32 + 1) instead of panicking. Subtracting more than available silently produces a huge positive number.

```rust
// DANGEROUS — no check before subtraction
let new_balance = current_balance - withdraw_amount;
// If withdraw_amount > current_balance, new_balance ≈ 2^64 (wraps!)

// SAFE — always validate first
assert!(
    current_balance.as_canonical_u64() >= withdraw_amount.as_canonical_u64(),
    "Insufficient balance"
);
let new_balance = current_balance - withdraw_amount;
```

**Rule**: ALWAYS check `.as_canonical_u64()` values before any Felt subtraction.

**Max Felt value**: The maximum valid Felt is `p - 1 = 18446744069414584320`, not `u64::MAX` (`18446744073709551615`). Using `u64::MAX` as a sentinel or boundary value causes silent wraparound.

## P2: Felt Comparison Operators Are Misleading for Quantity Logic

**Severity**: High — silently produces incorrect results

`<`, `>`, `<=`, `>=` on Felt values compare field elements, which differs from natural number ordering. In protocol-level code working with field elements, these comparisons may be intentional. For business logic (balances, amounts, counts), the results are misleading.

```rust
// MISLEADING for business logic — compares field elements
if balance > threshold { ... }

// CORRECT for business logic — compare as integers
if balance.as_canonical_u64() > threshold.as_canonical_u64() { ... }
```

**Rule**: For quantity/business logic, ALWAYS convert to `.as_canonical_u64()` before using comparison operators.

## P3: Function Argument Limit (4 Words / 16 Felts)

**Severity**: Medium — causes compilation errors

Functions can receive at most 4 Words (16 Felts) as arguments.

```rust
// PROBLEM — too many arguments
fn process(a: Word, b: Word, c: Word, d: Word, e: Word) { ... } // > 4 Words!

// SOLUTION — pass fat types by reference
fn process(a: &Word, b: &Word, c: &Word, d: &Word, e: &Word) { ... }
```

## P4: Storage API Is Typed

**Severity**: Medium — old examples no longer compile

The old `Value` / untyped `StorageMap` API is gone. Account storage is now:

- `StorageValue<T>` for a single typed slot
- `StorageMap<K, V>` for typed maps
- `get()` / `set()` methods instead of `.read()` / `.write()`
- `K: WordKey`, `T: WordValue`, `V: WordValue`

```rust
#[component]
struct CounterContract {
    #[storage(description = "single typed slot")]
    counter: StorageValue<Felt>,

    #[storage(description = "typed map")]
    balances: StorageMap<AccountId, Felt>,
}
```

If you need custom keys or values, implement `WordKey` / `WordValue` by converting to and from a single `Word`.

## P5: Storage Slot Naming Convention

**Severity**: Medium — causes silent default-value reads in tests

Storage slot names follow a strict pattern. Getting it wrong often returns the default value silently.

**Pattern**: `[package_name]::[snake_case(component_struct)]::[field_name]`

**Where the namespace segment comes from**: The `#[component]` macro derives the namespace from the Miden project package name — `[package] name` in your component's `miden-project.toml`, NOT from `Cargo.toml`. (Source: the macro loads `miden-project.toml` and uses `metadata.package.name()` as the storage namespace; see the caveat below.)

**Conversion rule**: Any `@version` suffix on the package name is stripped, and characters outside `[A-Za-z0-9_]` are replaced with `_`. Because the project package name is conventionally already `snake_case`, the namespace segment usually equals it verbatim.

| `miden-project.toml` `[package] name` | Component Struct | Field | Storage Slot Name |
|---------------------------------------|------------------|-------|-------------------|
| `counter_account` | `CounterContract` | `count_map` | `counter_account::counter_contract::count_map` |
| `bank_account` | `BankAccount` | `balances` | `bank_account::bank_account::balances` |
| `bank_account` | `BankAccount` | `initialized` | `bank_account::bank_account::initialized` |

**Caveat (toolchain-version dependent)**: This naming is a property of the Rust SDK contract macros, which live in the `miden-base-macros` crate (v0.12.0, part of the v0.15 Rust SDK family alongside `miden`, `miden-base`, and `miden-base-sys`, all 0.12.0; the separate midenc/compiler workspace is versioned 0.8.1). The slot-naming algorithm — `namespace::snake_case(struct)::field`, with non-`[A-Za-z0-9_]` mapped to `_` and `@version` stripped — has been stable, but verify against your installed toolchain rather than assuming a protocol version.

## P6: No-std Environment

**Severity**: Medium -- causes compilation errors

All contract code must be `#![no_std]`. Forgetting this or using std types causes build failures.

**Required at the top of every contract file:** See any contract under `contracts/` in [project-template](https://github.com/0xMiden/project-template) for the correct pattern (`#![no_std]` + `#![feature(alloc_error_handler)]`).

**For heap allocation (Vec, String, Box):**
```rust
extern crate alloc;
use alloc::vec::Vec;
```

## P7: Rust SDK `Asset` Is Two Words (Key + Value)

**Severity**: Medium — old `asset.inner[...]` code is stale

In the Rust SDK (`miden::Asset` / `miden_base_sys::bindings::Asset`), an `Asset` is encoded as two words:

```rust
pub struct Asset {
    pub key: Word,
    pub value: Word,
}
```

```rust
// Reading the amount from a fungible asset
let amount = asset.value[0];

// Persisting or comparing the asset class
let asset_key = asset.key;
```

Do not assume the old single-word asset layout. Use `asset.key` and `asset.value`, or protocol helpers, instead of reconstructing from old `asset.inner[...]` offsets.

**Clarification (not a v0.15 change)**: This two-word `{key, value}` form is the Rust SDK ABI type, whose own documentation states it matches the v0.14 protocol/base ABI — it is the SDK encoding, not a v0.15 protocol redesign. At the protocol layer, `Asset` is an enum `{ Fungible, NonFungible }`, and the vault words are obtained via `to_key_word()` / `to_value_word()`. Reading the fungible amount from `value[0]` is correct on both sides.

## P8: `Recipient::compute` Was Removed

**Severity**: Medium — causes compilation errors after upgrading

Building recipients now goes through the note binding:

```rust
extern crate alloc;
use alloc::vec;

let recipient = note::build_recipient(
    serial_num,
    script_root,
    vec![recipient_id.suffix, recipient_id.prefix],
);
```

`note::build_recipient` is retained in the Rust SDK as a friendly alias; it forwards to the underlying host function (`miden::protocol::note::compute_and_store_recipient`), which computes and stores the recipient in one step. You can call either name.

## P9: P2ID Note Root — Prefer `script_root()`, Do Not Hardcode

**Severity**: Low-Medium — breaks after miden-standards updates

Creating P2ID output notes requires the MAST root of the P2ID script. The root changes whenever the P2ID script or the assembler/hashing changes (it did between v0.14 and v0.15), so a hardcoded literal is both stale and unverifiable.

**Source of truth**: Use `P2idNote::script_root()` from `miden-standards` (returns a `NoteScriptRoot`, a `Word` newtype convertible via `.into()`). Derive the root from the dependency rather than embedding a literal, and re-derive after any dependency bump.

```rust
use miden_standards::note::P2idNote;

// script_root() returns a NoteScriptRoot (a Word newtype); convert to Word when needed.
let p2id_root: Word = P2idNote::script_root().into();
```

**If you must embed a constant** (e.g., inside compiler/contract code that cannot call into miden-standards), regenerate it from the current `miden-standards` version and verify it after every update. The four-limb literal below is an ILLUSTRATIVE v0.14-era value only — it will NOT match v0.15 and must not be copied as-is:

```rust
// ILLUSTRATIVE v0.14 value ONLY — does NOT match v0.15. Regenerate from
// P2idNote::script_root() for your pinned miden-standards version.
fn p2id_note_root() -> Word {
    Word::try_from([
        13362761878458161062_u64,
        15090726097241769395_u64,
        444910447169617901_u64,
        3558201871398422326_u64,
    ])
    .unwrap()
}
```

**Risk**: If miden-standards updates the P2ID script, any hardcoded digest becomes invalid and withdrawals silently fail.

**NoteType for P2ID**: P2ID output notes created in contract code should use the private note type value via `NoteType::from(felt!(0))` (see P10). In v0.15 the kernel rejects any note type other than `0` (private) or `1` (public) with `ERR_NOTE_INVALID_TYPE`. See [miden-bank withdraw](https://github.com/0xMiden/tutorials/blob/main/examples/miden-bank/contracts/bank-account/src/lib.rs) for the working pattern.

## P10: NoteType Variants Unavailable in Compiler SDK

**Severity**: Critical -- wrong values panic at runtime, named variants cause compilation errors

Named enum variants (`NoteType::Private`, `NoteType::Public`) don't exist in contract code — the SDK `NoteType` is an unvalidated transparent `Felt` wrapper. Construct via `NoteType::from()`:

| NoteType | Value |
|----------|-------|
| Private (default) | `NoteType::from(felt!(0))` |
| Public | `NoteType::from(felt!(1))` |

**v0.15 encoding changed**: The note-type encoding is now 1-bit — `Private = 0` (and `Private` is the protocol default) and `Public = 1`. Only these two values exist; there is no `Encrypted` type. Because the SDK wrapper does no validation, an out-of-range value (e.g. `felt!(2)` or `felt!(3)`) is not caught at compile time — it is rejected at execution time by the kernel with `ERR_NOTE_INVALID_TYPE` (the kernel asserts `note_type <= 1`). Emitting the old v0.14 value `felt!(2)` for a private note will panic on a v0.15 node.

See [miden-bank bank-account](https://github.com/0xMiden/tutorials/blob/main/examples/miden-bank/contracts/bank-account/src/lib.rs) for `NoteType::from(note_type)` usage.

## P11: Note Scripts Cannot Call Native Account Functions

**Severity**: High -- causes runtime failures

Note scripts cannot call `native_account::add_asset()` or other `native_account::` functions directly. The kernel's `authenticate_account_origin` check rejects these calls from a note context. Instead, note scripts must call an account component method, which then calls `native_account::add_asset()` internally.

See [miden-bank deposit-note](https://github.com/0xMiden/tutorials/blob/main/examples/miden-bank/contracts/deposit-note/src/lib.rs) for the correct pattern: the note script calls `bank_account::deposit()`, which internally calls `native_account::add_asset()`.

## P12: Note Inputs Are Immutable After Creation

**Severity**: Low -- causes incorrect architecture

Note inputs (`active_note::get_storage()`) are baked at note creation time and cannot be modified after creation. Design note input layouts carefully before deployment.
