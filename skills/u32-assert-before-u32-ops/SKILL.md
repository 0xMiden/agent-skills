---
name: u32-assert-before-u32-ops
description: Use when writing MASM `u32*` instructions on values from user input or untrusted sources — ensure the operands are valid u32s first.
---

# Validate u32 Operands Before u32 Instructions

## Rule

MASM's `u32*` instructions require their operands to be valid `u32` values (i.e. fit in 32 bits, `<= u32::MAX`). Applied to a non-u32 value the behavior is *undefined*: in the current VM such an op typically traps with the generic error (`operation expected u32 values, but got values: ...`) rather than telling you *which* precondition was violated — but trapping is not guaranteed (it may instead wrap/truncate and silently poison the proof).

Before applying any `u32*` instruction to a value that is not already known to be a valid u32 (e.g. it came from the stack as input, was read from memory, or arose from a non-u32 arithmetic op), assert the bound with a descriptive error:

```masm
u32assert            # assert the one value on top of the stack
u32assert2           # assert the two values on top of the stack
u32assertw           # assert the four elements of the word on top of the stack
```

All three accept a named error via `.err=`, e.g. `u32assert2.err=ERR_NOT_U32`.

If the operand is already known-valid (just produced by another `u32*` op, or a value loaded from a slot whose layout is u32 by construction), skip the assert.

## Why

`u32*` instructions are tuned for the precondition that operands fit in 32 bits. Passing an out-of-range operand to a `u32*` op is undefined behavior: the current VM almost always traps with the generic `operation expected u32 values, but got values: ...` error, but trapping is not guaranteed — it may instead wrap/truncate and silently poison the proof, and even when it traps it does not name the precondition or the call site. Asserting first with `u32assert*.err=` turns this fragile/undefined path into a deterministic, named failure mode, so a non-u32 input fails loudly and diagnosably at the point where the assumption is introduced.

## Examples

```masm
# Good: assert u32 before the u32 op
u32assert.err=ERR_VALUE_NOT_U32
u32overflowing_add

# Good: both operands at once
u32assert2.err=ERR_VALUES_NOT_U32
u32lt

# Bad: u32 op on untrusted input
u32overflowing_add   # if an operand exceeds 2^32 the behavior is undefined; the current
                     # VM typically traps with the generic "operation expected u32 values"
                     # error instead of a named one (but may instead wrap and poison the proof)
```
