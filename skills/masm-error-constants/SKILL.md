---
name: masm-error-constants
description: Use when adding or editing MASM `assert*` instructions — give every assertion a descriptive error (a named `ERR_*` constant or an inline string).
---

# MASM Error Constants

## Rule

Every MASM assertion should carry a descriptive error message. There are two valid forms in v0.15 source:

- **Named `ERR_*` constant** — preferred for reusable or shared errors, and the dominant style in protocol/standards code:

```masm
assert.err=ERR_NOTE_NOT_FOUND
assert_eqw.err=ERR_COMMITMENT_MISMATCH
```

- **Inline string literal** with `.err="..."` — valid and common, especially in core-lib/stdlib and for one-off, local checks:

```masm
u32assert2.err="number of storage map elements should fit into a u32"
assert.err="number of storage map elements must be a multiple of 8"
```

Prefer a named constant when the same error is raised in more than one place, when it is part of a module's documented error surface, or when Rust code needs to match on it (named `ERR_*` constants are codegen'd into Rust bindings — see `masm-rust-constant-parity`). Reach for an inline string for a purely local, single-site check where a named constant would only add indirection.

When you define a named error constant, it must:

- Use the `ERR_` prefix.
- Live in the file's dedicated errors section (see `masm-constants` skill).
- Have a descriptive string value, not a bare numeric code: `const ERR_NOTE_NOT_FOUND = "note not found"`.
- Be unique per distinct failure condition — do not share one `ERR_` across two unrelated asserts.

## Why

A bare `assert` traps with the default error code `0`, which tells the debugger nothing about which check failed; a descriptive error message (a named `ERR_*` constant or an inline string) ties each trap site to a specific failure mode. (The error string is hashed into a field element; an omitted code defaults to `0`.) Distinct constants per condition also let tests pin the expected error rather than matching a generic failure.

## Examples

```masm
# Good
const ERR_NOTE_NOT_FOUND = "note not found"
const ERR_COMMITMENT_MISMATCH = "stored commitment does not match recomputed value"

proc verify_note
    # ...
    assert.err=ERR_NOTE_NOT_FOUND
    # ...
    assert_eqw.err=ERR_COMMITMENT_MISMATCH
end

# Bad: bare assertion
proc verify_note
    assert
end

# Bad: shared generic constant for unrelated cases
const ERR_INVALID = "invalid"
assert.err=ERR_INVALID   # used in 6 places, each meaning something different
```
