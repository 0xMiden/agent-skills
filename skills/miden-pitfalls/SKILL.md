---
name: miden-pitfalls
description: Critical pitfalls and safety rules for Miden Rust SDK development. Covers felt arithmetic security, comparison operators, stack limits, argument limits, array ordering, storage naming, no-std setup, asset layout, and P2ID roots. Use when reviewing, debugging, or writing Miden contract code.
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
assert!(current_balance.as_u64() >= withdraw_amount.as_u64(),
        "Insufficient balance");
let new_balance = current_balance - withdraw_amount;
```

**Rule**: ALWAYS check `.as_u64()` values before any Felt subtraction.

## P2: Felt Comparison Operators Are Wrong for Business Logic

**Severity**: High — silently produces incorrect results

`<`, `>`, `<=`, `>=` on Felt values compare field elements, not natural numbers. The results are mathematically correct in the field but wrong for business logic.

```rust
// WRONG — compares field elements
if balance > threshold { ... }

// CORRECT — compare as integers
if balance.as_u64() > threshold.as_u64() { ... }
```

**Rule**: ALWAYS convert to `.as_u64()` before using comparison operators.

## P3: Stack Limit (16 Elements)

**Severity**: Medium — causes compilation errors

Only 16 stack elements are directly accessible. Too many local variables in a single function trigger "invalid stack index" errors.

```rust
// PROBLEM — too many locals
fn complex_fn(a: Felt, b: Felt, c: Felt, d: Felt) {
    let x = a + b;
    let y = c + d;
    let z = x + y;
    // ... more variables = stack overflow
}

// SOLUTION — break into smaller functions
fn step1(a: Felt, b: Felt) -> Felt { a + b }
fn step2(c: Felt, d: Felt) -> Felt { c + d }
fn combine(x: Felt, y: Felt) -> Felt { x + y }
```

## P4: Function Argument Limit (4 Words / 16 Felts)

**Severity**: Medium — causes compilation errors

Functions can receive at most 4 Words (16 Felts) as arguments.

```rust
// PROBLEM — too many arguments
fn process(a: Word, b: Word, c: Word, d: Word, e: Word) { ... } // > 4 Words!

// SOLUTION — use note inputs for complex data
let inputs = active_note::get_inputs();
// Parse inputs[0..N] into your data structures
```

## P5: Array Ordering Reversal (Rust ↔ MASM)

**Severity**: Medium — causes wrong data interpretation

Arrays passed from Rust are received in reversed order at the MASM level. `Word::from([a, b, c, d])` becomes `[d, c, b, a]` on the stack.

**Rule**: Be consistent with array construction and parsing. When constructing a Word for storage keys, the order you define in Rust is the order you should use when reading back.

## P6: Storage Slot Naming Convention

**Severity**: Medium — causes silent zero returns in tests

Storage slot names follow a strict pattern. Getting it wrong returns zero silently.

**Pattern**: `miden::component::[snake_case(package)]::[field_name]`

**Conversion rule**: Replace `:` and `-` with `_` in the package name from `[package.metadata.component] package = "..."`.

| Package in Cargo.toml | Field | Storage Slot Name |
|----------------------|-------|-------------------|
| `miden:counter-account` | `count_map` | `miden::component::miden_counter_account::count_map` |
| `miden:bank-account` | `balances` | `miden::component::miden_bank_account::balances` |
| `miden:bank-account` | `initialized` | `miden::component::miden_bank_account::initialized` |

## P7: No-std Environment

**Severity**: Medium — causes compilation errors

All contract code must be `#![no_std]`. Forgetting this or using std types causes build failures.

**Required at the top of every contract file:**
```rust
#![no_std]
#![feature(alloc_error_handler)]
```

**For heap allocation (Vec, String, Box):**
```rust
extern crate alloc;
use alloc::vec::Vec;
```

## P8: Asset Word Layout

**Severity**: Medium — creates invalid assets

Fungible assets have a specific Word layout. Getting the order wrong creates invalid assets or reads wrong amounts.

```
Asset Word: [amount, 0, faucet_suffix, faucet_prefix]
              [0]   [1]      [2]            [3]
```

```rust
// Reading amount from an asset
let amount = asset.inner[0];

// Constructing asset key for storage (including faucet identity)
let key = Word::from([
    depositor.prefix,
    depositor.suffix,
    asset.inner[3],  // faucet prefix
    asset.inner[2],  // faucet suffix
]);
```

## P9: P2ID Note Root Hardcoding

**Severity**: Low-Medium — breaks after miden-standards updates

Creating P2ID output notes requires the MAST root digest of the P2ID script. This is typically hardcoded as a constant.

```rust
fn p2id_note_root() -> Digest {
    Digest::from_word(Word::new([
        Felt::from_u64_unchecked(13362761878458161062),
        Felt::from_u64_unchecked(15090726097241769395),
        Felt::from_u64_unchecked(444910447169617901),
        Felt::from_u64_unchecked(3558201871398422326),
    ]))
}
```

**Risk**: If miden-standards updates the P2ID script, this digest becomes invalid and withdrawals silently fail.

**Mitigation**: Use `P2idNote::script_root()` from miden-standards if available, or verify the hardcoded root matches the current version after dependency updates.

## Quick Reference

| Pitfall | One-Line Rule |
|---------|--------------|
| P1 Felt arithmetic | Always `.as_u64()` before subtraction |
| P2 Felt comparison | Always `.as_u64()` for `<` `>` `<=` `>=` |
| P3 Stack limit | Max 16 locals — break large functions |
| P4 Arg limit | Max 4 Words per function — use note inputs |
| P5 Array order | Rust arrays reverse at MASM level |
| P6 Storage names | `miden::component::pkg_name::field` (underscores) |
| P7 No-std | `#![no_std]` + `#![feature(alloc_error_handler)]` |
| P8 Asset layout | `[amount, 0, suffix, prefix]` |
| P9 P2ID root | Verify digest after dependency updates |
