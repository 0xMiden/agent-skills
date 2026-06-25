---
name: cheap-masm-equivalents
description: Use when writing or reviewing MASM hot paths — prefer the cheaper equivalent instruction: `neq.0` over `push.0 gt` for non-zero checks, `cdrop` over an `if/else` selecting between two values, `dup.N` over `loc_load` for a value still on the stack, `eqw` over hand-rolled element-wise word comparison, `u32gt`/`u32lt` over generic `gt`/`lt` on known-u32 operands.
---

# Prefer Cheap MASM Equivalents

## Rule

Several MASM idioms have a cheap and an expensive form. Use the cheap one when both produce the same result on the inputs the procedure can see:

- Non-zero check: `neq.0` (2 cycles) over a comparison-based check like `push.0 gt` (~17 cycles). `neq.0` lowers to `eqz not`; `push.0 gt` does a full field comparison just to learn "not zero".
- Selecting between two values on a flag: `cdrop` (2 cycles) over an `if.true ... else ... end` branch with the same effect.
- Re-fetch a recently-pushed value: `dup.N` (1-3 cycles) over `loc_load.N` (which costs more) when the value is still on the stack.
- Whole-word equality: the single `eqw` instruction (15 cycles) over a hand-rolled element-wise sequence of `eq`/`and`.
- u32-known operands: `u32lt` (3 cycles) / `u32gt` (4 cycles) over generic `lt` (17 cycles) / `gt` (16 cycles).

Don't apply the cheap form when the operands violate its precondition. `u32lt`/`u32gt` are undefined if either operand is >= 2^32, so the operands must be known (or asserted) to be valid u32s first.

Also note that `gt.0` is only sugar for `push.0 gt`: it parses, but tests `a > 0` via a full field comparison, not `a != 0`. For a genuine non-zero check use `neq.0`; reach for `push.0 gt` / `gt.0` only when you actually want strictly-greater-than-zero.

## Why

MASM cycle costs are not uniform — `gt`/`lt` do full 64-bit comparison work that `neq` skips, so a hot path using the expensive form pays for it on every call. The swaps are semantically equivalent under their preconditions, so the saving is free. The protocol does this in practice: a loop in `account_delta.masm` carries the comment `# we use neq instead of lt for efficiency`. The Miden assembly docs make the same point for branches: an `if.true ... else ... end` incurs non-negligible overhead, so when both branches just select a value (no incompatible side effects), compute both and select with `cdrop`.

## Examples

```masm
# Good: non-zero check, 2 cycles (lowers to `eqz not`)
neq.0

# Bad: full field comparison just to test "not zero", ~17 cycles
push.0 gt
```

```masm
# Good: cdrop for ternary selection (condition on TOP), 2 cycles
# stack: [cond, b, a]
cdrop
# stack: [b if cond else a]   (cond=1 keeps b, cond=0 keeps a; fails if cond > 1)

# Bad: branchy equivalent for the same condition-on-top layout
# stack: [cond, b, a]
if.true
    swap drop   # cond true: keep b
else
    drop        # cond false: keep a
end
# stack: [b if cond else a]
```

Note on `cdrop` stack order: the condition is consumed from the top of the stack and `b` (the value just below it) is kept when the condition is 1, `a` when it is 0. `cdrop` fails if the condition is > 1. `if.true`/`if.false` likewise pop their condition from the top of the stack.
