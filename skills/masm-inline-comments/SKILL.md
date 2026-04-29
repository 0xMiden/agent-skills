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
exec.native_account::remove_asset
# => [ASSET, note_idx, pad(11)]

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
- Stack state after complex operations: `# => [ptr, ASSET, end_ptr]`
- Purpose of a code block: `# compute the pointer at which we should stop iterating`
- Non-obvious logic or business rules
- TODO items and references to external specs

### 3. Blank line after `# => [...]` trackers

Insert a blank line after a `# => [...]` stack-state tracker, except when the next non-blank line is one of:

- `end` (proc / `while.true` / `if.true` / `repeat.N` closing).
- A control-flow keyword such as `else`, `else.true`, or `else.false`.
- Another `# =>` line that continues the same multi-line stack state.
- A `#` continuation comment that explains the tracker.

This pairs each stack state visually with the operation that produced it and lets the eye skim from one labeled state to the next.

**Good:**

```masm
exec.native_account::remove_asset
# => [ASSET, note_idx, pad(11)]

dupw dup.8 movdn.4
# => [ASSET, note_idx, ASSET, note_idx, pad(11)]
```

**Also OK (no blank line before `end` or control flow):**

```masm
# => [pad(16)]
end
```

## Examples

**Good:**

```masm
# remove the asset from the account
exec.native_account::remove_asset
# => [ASSET, note_idx, pad(11)]

dupw dup.8 movdn.4
# => [ASSET, note_idx, ASSET, note_idx, pad(11)]

exec.output_note::add_asset
# => [ASSET, note_idx, pad(11)]
```

**Avoid:**

```masm
# Swap the top two elements
swap  # swap

# Drop the word
dropw  # drops 4 elements
```
