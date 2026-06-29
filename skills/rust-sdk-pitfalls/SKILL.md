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

## P3: Direct Call Boundary Passes At Most 16 Stack Felts (4 Words)

**Severity**: Medium — overflow degrades to slower memory transfer; a hard error only fires for FPI imports

A direct cross-context / export call passes its parameters on the MASM operand stack, whose addressable window is 16 felts (4 Words, counting the canonical-ABI output pointer when present). This is NOT a universal compile error: for ordinary component functions, when the flattened parameters exceed 16 flat values *or* 16 stack felts, the compiler automatically transfers them through memory (Canonical-ABI input indirection) instead of failing. A 17-felt or 9×`i64` (18-felt) signature compiles fine — it just gets the slower indirect calling convention.

A hard compile error fires only for **direct FPI imports** that lower to more than 16 operand-stack felts after expanding 64-bit values and result pointers.

```rust
// COMPILES — flattens past 16 felts, so params move through memory (slower, not an error)
fn process(a: Word, b: Word, c: Word, d: Word, e: Word) { ... }

// PREFERRED — keep export / component-method footprints small, or pass aggregates by reference
fn process(a: &Word, b: &Word, c: &Word, d: &Word, e: &Word) { ... }
```

## P4: Storage API Is Typed

**Severity**: Medium — old examples no longer compile

The old `Value` / untyped `StorageMap` API is gone. Account storage is now:

- `StorageValue<T>` for a single typed slot
- `StorageMap<K, V>` for typed maps
- `get()` / `set()` methods instead of `.read()` / `.write()`
- `K: WordKey`, `T: WordValue`, `V: WordValue`

A v0.15 account component is written in **three parts**, and `#[component]` no longer applies to a struct. Annotate the storage struct with `#[component_storage]`, the API `trait` with `#[component]`, and the `impl Trait for Storage` block with `#[component]`:

```rust
// 1. Storage struct — annotated #[component_storage], NOT #[component].
//    Applying #[component] to a struct is now a hard compile error.
#[component_storage]
struct CounterContractStorage {
    #[storage(description = "single typed slot")]
    counter: StorageValue<Felt>,

    #[storage(description = "typed map")]
    balances: StorageMap<Word, Felt>,
}

// 2. API trait — defines the exported interface.
#[component]
trait CounterContract {
    fn get_count(&self) -> Felt;
    fn increment_count(&mut self) -> Felt;
}

// 3. Implementation — the behavior, wired to the storage struct.
#[component]
impl CounterContract for CounterContractStorage {
    fn get_count(&self) -> Felt { self.counter.get() }
    fn increment_count(&mut self) -> Felt {
        let next = self.counter.get() + felt!(1);
        self.counter.set(next);
        next
    }
}
```

If you need custom keys or values, implement `WordKey` / `WordValue` by converting to and from a single `Word`.

## P5: Storage Slot Naming Convention

**Severity**: Medium — causes silent default-value reads in tests

Storage slot names follow a strict pattern. Getting it wrong often returns the default value silently.

**Pattern**: `[package_name]::[namespace_interface_segment]::[field_name]`

**Where the segments come from**: The `#[component_storage]` macro (NOT `#[component]`) processes the `#[storage]` fields and derives slot names. It loads `miden-project.toml` (next to your `Cargo.toml`, NOT `Cargo.toml` itself):

- **First segment** = `[package] name`.
- **Middle segment** = the *interface segment* of the `[lib] namespace` value. The namespace is a fully-qualified component id `namespace:package/interface@version`; the interface segment sits between the last `/` and the `@`. This is deliberately decoupled from the Rust storage-struct name, so renaming the private struct cannot change deployed slot names. The struct name (`BankStorage`, `CounterContractStorage`, …) does NOT appear in the slot name.
- **Last segment** = the `#[storage]` field name.

**Conversion rule**: Each segment is sanitized — any `@version` suffix is stripped, the interface segment is passed through `snake_case`, and characters outside `[A-Za-z0-9_]` are replaced with `_` (an empty or leading-`_` segment is prefixed with `x`). Project package names are conventionally kebab-case (e.g. `counter-contract`, `bank-account`), so the first segment is that name with hyphens replaced by `_` — it does NOT equal the package name verbatim (`counter-contract` → `counter_contract`).

| `[package] name` | `[lib] namespace` | Field | Storage Slot Name |
|------------------|-------------------|-------|-------------------|
| `counter-contract` | `miden:counter-contract/miden-counter-contract@0.1.0` | `count_map` | `counter_contract::miden_counter_contract::count_map` |
| `bank-account` | `miden:bank-account/bank@0.1.0` | `balances` | `bank_account::bank::balances` |
| `bank-account` | `miden:bank-account/bank@0.1.0` | `initialized` | `bank_account::bank::initialized` |

**Caveat (toolchain-version dependent)**: This naming is a property of the Rust SDK contract macros, which live in the `miden-base-macros` crate (v0.12.0, part of the v0.15 Rust SDK family alongside `miden` and `miden-base`, all 0.12.0; the separate midenc/compiler workspace is versioned 0.8.1). The slot-naming algorithm — `package_name::snake_case(interface_segment)::field`, with non-`[A-Za-z0-9_]` mapped to `_` and `@version` stripped — has been stable, but verify against your installed toolchain rather than assuming a protocol version.

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

**Clarification (not a v0.15 change)**: This two-word `{key, value}` form is the Rust SDK ABI type, whose own documentation states it matches the protocol/base ABI from before v0.15 — it is the SDK encoding, not a v0.15 protocol redesign. At the protocol layer, `Asset` is an enum `{ Fungible, NonFungible }`, and the vault words are obtained via `to_key_word()` / `to_value_word()`. Reading the fungible amount from `value[0]` is correct on both sides.

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

Creating P2ID output notes requires the MAST root of the P2ID script. The root changes whenever the P2ID script or the assembler/hashing changes (it changed in the v0.15 release), so a hardcoded literal is both stale and unverifiable.

**Source of truth**: Use `P2idNote::script_root()` from `miden-standards` (returns a `NoteScriptRoot`, a `Word` newtype convertible via `.into()`). Derive the root from the dependency rather than embedding a literal, and re-derive after any dependency bump.

```rust
use miden_standards::note::P2idNote;

// script_root() returns a NoteScriptRoot (a Word newtype); convert to Word when needed.
let p2id_root: Word = P2idNote::script_root().into();
```

**If you must embed a constant** (e.g., inside compiler/contract code that cannot call into miden-standards), regenerate it from the current `miden-standards` version and verify it after every update. The four-limb literal below is an ILLUSTRATIVE pre-v0.15 value only — it will NOT match v0.15 and must not be copied as-is:

```rust
// ILLUSTRATIVE pre-v0.15 value ONLY — does NOT match v0.15. Regenerate from
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

**NoteType for P2ID**: P2ID output notes created in contract code should use the private note type value via `NoteType::from(felt!(0))` (see P10). In v0.15 the kernel rejects any note type other than `0` (private) or `1` (public) with `ERR_NOTE_INVALID_TYPE`. See [miden-bank withdraw](https://github.com/0xMiden/tutorials/blob/078e2fb289fc299359ce44c3900bb9e59be93d40/examples/miden-bank/contracts/bank-account/src/lib.rs) for the working pattern.

## P10: NoteType Variants Unavailable in Compiler SDK

**Severity**: Critical -- wrong values panic at runtime, named variants cause compilation errors

Named enum variants (`NoteType::Private`, `NoteType::Public`) don't exist in contract code — the SDK `NoteType` is an unvalidated transparent `Felt` wrapper. Construct via `NoteType::from()`:

| NoteType | Value |
|----------|-------|
| Private (default) | `NoteType::from(felt!(0))` |
| Public | `NoteType::from(felt!(1))` |

**v0.15 encoding changed**: The note-type encoding is now 1-bit — `Private = 0` (and `Private` is the protocol default) and `Public = 1`. Only these two values exist; there is no `Encrypted` type. Because the SDK wrapper does no validation, an out-of-range value (e.g. `felt!(2)` or `felt!(3)`) is not caught at compile time — it is rejected at execution time by the kernel with `ERR_NOTE_INVALID_TYPE` (the kernel asserts `note_type <= 1`). Emitting the old pre-v0.15 value `felt!(2)` for a private note will panic on a v0.15 node.

See [miden-bank bank-account](https://github.com/0xMiden/tutorials/blob/078e2fb289fc299359ce44c3900bb9e59be93d40/examples/miden-bank/contracts/bank-account/src/lib.rs) for `NoteType::from(note_type)` usage.

## P11: Note Scripts Cannot Call Native Account Functions

**Severity**: High -- causes runtime failures

Note scripts cannot call `native_account::add_asset()` or other `native_account::` functions directly. The kernel's `authenticate_account_origin` check rejects these calls from a note context. Instead, note scripts must call an account component method, which then calls `native_account::add_asset()` internally.

See [miden-bank deposit-note](https://github.com/0xMiden/tutorials/blob/078e2fb289fc299359ce44c3900bb9e59be93d40/examples/miden-bank/contracts/deposit-note/src/lib.rs) for the correct pattern: the note script declares the consuming account via `#[account(bank_account::Bank)] pub struct Wallet;` and, inside `#[note_script] fn run(self, _arg: Word, account: &mut Wallet)`, calls `account.deposit(depositor, asset)` on that wrapper. The `deposit()` component method then calls `native_account::add_asset()` internally. It is NOT a free `bank_account::deposit()` call.

## P12: Note Inputs Are Immutable After Creation

**Severity**: Low -- causes incorrect architecture

Note inputs (`active_note::get_storage()`) are baked at note creation time and cannot be modified after creation. Design note input layouts carefully before deployment.
