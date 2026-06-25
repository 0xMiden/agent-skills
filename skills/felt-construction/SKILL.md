---
name: felt-construction
description: Use when constructing a `Felt` from a numeric value in Rust — avoid silently producing a non-canonical Felt that no longer equals the original input.
---

# Felt Construction From Untrusted Numeric Inputs

## Rule

Do not call `Felt::new_unchecked(x)` when `x` could be greater than or equal to the field order. `new_unchecked` stores any `u64` raw without reduction, so an out-of-range value produces a non-canonical `Felt` that no longer equals the original input on a canonical comparison — a classic source of hard-to-attribute bugs.

Use one of:

- `Felt::from(x)` where `x` is a `u32`, `u16`, or `u8` (infallible).
- `Felt::new(x)` or `Felt::try_from(x)` for `u64` inputs — both are checked and return `Result<Felt, FeltFromIntError>`, rejecting values that are `>= Felt::ORDER`.

If you have independently proven the bound and need the unchecked path, only then reach for `Felt::new_unchecked(x)`, comparing against `Felt::ORDER` (the `u64` field order; there is no `Felt::MODULUS`).

## Why

The field order sits just below `2^64` (`2^64 - 2^32 + 1`), so an out-of-range `u64` is reduced only for a narrow band of large values — most tests pass and production hits the bad input as a value mismatch far from the call. `Felt::from(u32)` cannot exceed the field, and `Felt::new` / `Felt::try_from` force the bound check and surface overflow as a `FeltFromIntError`.

## Examples

```rust
// Good: u32 input, infallible conversion
let f = Felt::from(slot_index as u32);

// Good: untrusted u64 input, checked conversion
let f = Felt::try_from(user_value).map_err(|_| Error::FeltOverflow)?;

// Good: equivalent checked constructor
let f = Felt::new(user_value).map_err(|_| Error::FeltOverflow)?;

// Bad: stores any u64 raw with no reduction; a value >= Felt::ORDER
// yields a non-canonical Felt that does not equal user_value
let f = Felt::new_unchecked(user_value);
```
