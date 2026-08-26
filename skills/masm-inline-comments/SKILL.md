---
name: masm-inline-comments
description: Enforce inline commenting conventions for Miden Assembly (.masm) files. Use when editing, reviewing, or creating .masm files.
---

# MASM Inline Commenting Conventions

## Rules

### 1. Inline comments start lowercase

Inline comments (single `#`) should begin with a lowercase letter.

```masm
# good: lowercase start
# remove the asset from the account
exec.native_account::remove_asset
dropw
# => [ASSET_ID, ASSET_VALUE, note_idx, pad(7)]

# Bad: uppercase start (avoid)
# Remove the asset from the account
```

### 2. Don't over-comment obvious operations

Only add comments that provide value. Skip comments for self-explanatory operations.
Only apply this rule to new code you write. Do not remove comments that are present in the code.

**Skip comments for:**
- Simple arithmetic: `add`, `sub`, `mul`, `div`
- Basic stack ops when context is clear: `drop`, `swap`, `dup`
- Standard control flow: `if.true`, `while.true`, `end`

**Do comment:**
- Stack state after complex operations: `# => [ASSET_ID, ASSET_VALUE, note_idx, pad(7)]`
- Purpose of a code block: `# compute the pointer at which we should stop iterating`
- Non-obvious logic or business rules
- TODO items and references to external specs

### 3. Blank line after `# => [...]` trackers

Insert a blank line after a `# => [...]` stack-state tracker, except when the next non-blank line is one of:

- `end` (proc / `while.true` / `if.true` / `repeat.N` closing).
- A control-flow keyword such as `else` (note: `else` is always bare; a false-conditioned branch uses `if.false`, not an `else.*` suffix).
- Another `# =>` line that continues the same multi-line stack state.
- A `#` continuation comment that explains the tracker.

This pairs each stack state visually with the operation that produced it and lets the eye skim from one labeled state to the next.

**Good:**

```masm
dupw.1 dupw.1
# => [ASSET_ID, ASSET_VALUE, ASSET_ID, ASSET_VALUE, note_idx, pad(7)]

# remove the asset from the account
exec.native_account::remove_asset
dropw
# => [ASSET_ID, ASSET_VALUE, note_idx, pad(7)]
```

**Also OK (no blank line before `end` or control flow):**

```masm
# => [pad(16)]
end
```

### 4. Match the doc block

An inline `# => [...]` tracker uses the same item names, capitalization, and `(N)` span notation as the `#!` doc block for the enclosing procedure (see `masm-doc-comments`):

- Single-felt names stay lowercase: `note_idx`, `final_nonce`.
- Word names stay UPPERCASE: `ASSET_ID`, `ASSET_VALUE`, `RECIPIENT`.
- `(N)` spans stay lowercase: `pad(12)`, `foreign_procedure_inputs(15)`.

Composite names like `account_id_{suffix,prefix}` are a doc-block shorthand for a group of felts. In inline trackers they decompose into their individual felts, since each felt occupies one stack slot:

```masm
#! Inputs:  [account_id_{suffix,prefix}, amount]
pub proc transfer
    # => [account_id_suffix, account_id_prefix, amount]
    # ...
end
```

### 5. Reuse existing terminology

Use the vocabulary already established in the surrounding code and doc comments. Do not coin new terms or colloquialisms for a concept that already has a name — a value written to a local is "stored", not "stashed". This applies to inline comments and to constant-header comments.

### 6. Comment the code, not the change

Inline comments explain what the code does for a future reader, not why a particular PR made a change. Avoid PR narrative and framing such as "this is the X that prevents Y"; describe the operation and its purpose as the code stands.

### 7. Accessing a word's individual elements

When a tracker needs to show individual elements of a word, destructure the word and group the elements with brackets:

```masm
# => [ASSET_ID, ASSET_VALUE]
# => [[asset_class_suffix, asset_class_prefix, faucet_id_suffix_and_metadata, faucet_id_prefix], ASSET_VALUE]

dup
# => [asset_class_suffix, ASSET_ID, ASSET_VALUE]
```

An `Asset` occupies two words on the stack — the `ASSET_ID` word followed by the `ASSET_VALUE` word — so a procedure that takes one asset shows both.

## Examples

**Good:**

```masm
dupw.1 dupw.1
# => [ASSET_ID, ASSET_VALUE, ASSET_ID, ASSET_VALUE, note_idx, pad(7)]

# remove the asset from the account
exec.native_account::remove_asset
dropw
# => [ASSET_ID, ASSET_VALUE, note_idx, pad(7)]

exec.output_note::add_asset
# => [pad(16)]
```

**Avoid:**

```masm
# Swap the top two elements
swap  # swap

# Drop the word
dropw  # drops 4 elements
```
