---
name: rust-sdk-pitfalls
description: Critical pitfalls and safety rules for Miden Rust SDK development. Covers felt arithmetic security, comparison operators, argument limits, storage naming, no-std setup, asset layout, and P2ID roots. Use when reviewing, debugging, or writing Miden contract code.
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

## P2: Felt Comparison Operators Are Misleading for Quantity Logic

**Severity**: High — silently produces incorrect results

`<`, `>`, `<=`, `>=` on Felt values compare field elements, which differs from natural number ordering. In protocol-level code working with field elements, these comparisons may be intentional. For business logic (balances, amounts, counts), the results are misleading.

```rust
// MISLEADING for business logic — compares field elements
if balance > threshold { ... }

// CORRECT for business logic — compare as integers
if balance.as_u64() > threshold.as_u64() { ... }
```

**Rule**: For quantity/business logic, ALWAYS convert to `.as_u64()` before using comparison operators.

## P3: Function Argument Limit (4 Words / 16 Felts)

**Severity**: Medium — causes compilation errors

Functions can receive at most 4 Words (16 Felts) as arguments.

```rust
// PROBLEM — too many arguments
fn process(a: Word, b: Word, c: Word, d: Word, e: Word) { ... } // > 4 Words!

// SOLUTION — pass fat types by reference
fn process(a: &Word, b: &Word, c: &Word, d: &Word, e: &Word) { ... }
```

## P4: Storage Slot Naming Convention

**Severity**: Medium — causes silent zero returns in tests

Storage slot names follow a strict pattern. Getting it wrong returns zero silently.

**Pattern**: `miden::component::[snake_case(package)]::[field_name]`

**Conversion rule**: Replace `:` and `-` with `_` in the package name from `[package.metadata.component] package = "..."`.

| Package in Cargo.toml | Field | Storage Slot Name |
|----------------------|-------|-------------------|
| `miden:counter-account` | `count_map` | `miden::component::miden_counter_account::count_map` |
| `miden:bank-account` | `balances` | `miden::component::miden_bank_account::balances` |
| `miden:bank-account` | `initialized` | `miden::component::miden_bank_account::initialized` |

## P5: No-std Environment

**Severity**: Medium -- causes compilation errors

All contract code must be `#![no_std]`. Forgetting this or using std types causes build failures.

**Required at the top of every contract file:** See any contract in [contracts/](../../../contracts/) for the correct pattern (`#![no_std]` + `#![feature(alloc_error_handler)]`).

**For heap allocation (Vec, String, Box):**
```rust
extern crate alloc;
use alloc::vec::Vec;
```

## P6: Asset Word Layout

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

## P7: P2ID Note Root Hardcoding

**Severity**: Low-Medium — breaks after miden-standards updates

Creating P2ID output notes requires the MAST root digest of the P2ID script. This is typically hardcoded as a constant.

For any note that is being created within the compiler code, the MAST root digest is needed. Below you find the example of a P2ID note

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
| P2 Felt comparison | Always `.as_u64()` for `<` `>` `<=` `>=` in business logic |
| P3 Arg limit | Max 4 Words per function — pass by reference |
| P4 Storage names | `miden::component::pkg_name::field` (underscores) |
| P5 No-std | `#![no_std]` + `#![feature(alloc_error_handler)]` |
| P6 Asset layout | `[amount, 0, suffix, prefix]` |
| P7 P2ID root | Verify digest after dependency updates |
