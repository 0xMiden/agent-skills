---
name: masm-doc-comments
description: Enforce doc comment conventions for Miden Assembly (.masm) procedures. Use when editing, reviewing, or creating .masm procedures, especially when documenting inputs, outputs, panic conditions, or invocation types.
---

# MASM Procedure Doc Comments

## Overview

Every public MASM procedure should have a doc comment block using `#!` prefix with these sections in order:

1. **Description** - What the procedure does
2. **Inputs/Outputs** - Stack state before/after
3. **Where** - Explanation of each stack item (omit when the Description already names every item)
4. **Panics if** - Error conditions (when applicable)
5. **Invocation** - How the procedure is called (`exec`, `call`, `dynexec`, or `syscall`)

## Required Format

```masm
#! Brief description of what the procedure does.
#!
#! Additional context if needed.
#!
#! Inputs:  [item1, item2, WORD_ITEM]
#! Outputs: [result1, RESULT_WORD]
#!
#! Where:
#! - item1 is the description of item1.
#! - item2 is the description of item2.
#! - WORD_ITEM is a 4-element word representing X.
#! - result1 is the description of result1.
#! - RESULT_WORD is the resulting word.
#!
#! Panics if:
#! - condition that causes the procedure to fail.
#! - another error condition.
#!
#! Invocation: exec
pub proc my_procedure
```

## Stack Notation

### Naming Conventions

| Type | Style | Example |
|------|-------|---------|
| Single felt | lowercase with underscores | `note_index`, `amount`, `balance` |
| Word (4 felts) | UPPERCASE with underscores | `ASSET_ID`, `ASSET_VALUE`, `RECIPIENT`, `SCRIPT_ROOT` |
| Multi-felt (2-3) | lowercase with `{parts}` suffix | `account_id_{suffix,prefix}` |
| All-zero Word | `EMPTY_WORD` | `[..., EMPTY_WORD, ...]` |

`EMPTY_WORD` is a naming convention used in stack trackers, `Where:` bullets, and prose comments to denote the all-zero Word `[0, 0, 0, 0]`.

In composite braces, list parts in **stack-top-first order**: `account_id_{suffix,prefix}`, because the suffix sits on top of the stack and the prefix below it. The same applies to other split-128-bit IDs (`sender_{suffix,prefix}`, `faucet_id_{suffix,prefix}`, etc.). Suffix-first dominates protocol source by roughly six to one; write it that way in new code.

Brace *spacing* is genuinely mixed in source — both `{suffix,prefix}` and `{suffix, prefix}` are common — so match the surrounding file rather than reformatting existing braces.

An `Asset` occupies two words and is documented as two items: the `ASSET_ID` word followed by the `ASSET_VALUE` word.

### Stack Order

Items are listed left-to-right, with the **top of stack first**:

```masm
#! Inputs:  [top_item, second_item, THIRD_WORD]
```

### Empty Stack

Use empty brackets for no inputs or outputs:

```masm
#! Inputs:  []
#! Outputs: [result]
```

### Span notation `(N)` family

`(N)` after an item name denotes a span of N felts (not a Word). Spans stay lowercase; Words stay UPPERCASE and never take `(N)`. `pad(N)` (see masm-padding skill) is one member of this family; other spans appear in protocol code:

```masm
#! Inputs:  [first_element, foreign_procedure_inputs(15)]
#! Outputs: [foreign_procedure_outputs(16)]
```

### Padding (for procedures entered at the stack-depth-16 floor)

Procedures entered via `call` (and dynamically-dispatched entry points) must show explicit padding so Inputs and Outputs each sum to 16 elements (see masm-padding skill):

```masm
#! Inputs:  [ASSET_ID, ASSET_VALUE, pad(8)]
#! Outputs: [FINAL_ASSET_VALUE, pad(12)]
#!
#! Invocation: call
```

## Where Section

Define every item from Inputs and Outputs that needs description:

```masm
#! Where:
#! - note_index is the index of the input note.
#! - sender_{suffix,prefix} are the suffix and prefix felts of the sender ID.
#! - ASSET_ID is the asset ID of the asset to be burnt.
#! - ASSET_VALUE is the value of the asset from the vault, which can be the EMPTY_WORD if it isn't present.
#! - balance is the fungible asset balance in the vault.
```

**Rules:**
- Use "is" for single items, "are" for multi-part items
- Start descriptions lowercase (continues the sentence)
- End each line with a period
- Group related items (e.g., all inputs, then all outputs)
- Avoid including low-level details, e.g. how a value is computed.
    - Good: `NOTE_DETAILS_COMMITMENT is the commitment to the note's details.`
    - Avoid: `NOTE_DETAILS_COMMITMENT is the commitment to the note's details computed as hash(RECIPIENT_DIGEST || ASSETS_COMMITMENT).`

### When `Where:` may be omitted

Omit the `Where:` section entirely when the Description already names every Inputs/Outputs item and adding bullets would just repeat that information. Common case: trivial accessor procedures.

```masm
#! Returns the maximum supply.
#!
#! Inputs:  [pad(16)]
#! Outputs: [max_supply, pad(15)]
#!
#! Invocation: call
@account_procedure
pub proc get_max_supply
```

No `Where:` is needed: the single named output `max_supply` is already identified by the Description. If you would otherwise write `#! - max_supply is the maximum supply.`, skip it.

Add `Where:` whenever any item needs description beyond what the Description line conveys — different name, additional constraint, composition, or anything non-obvious.

## Panics Section

Bullets describe the **condition**, not the error identifier. Write `the nonce has already been incremented.`, not `ERR_ACCOUNT_NONCE_CAN_ONLY_BE_INCREMENTED_ONCE.`. The identifier is an implementation detail of the assert; the doc comment should read as English.

### Direct Panics

List conditions from `assert*` statements in the procedure:

```masm
#! Panics if:
#! - flag is false.
proc sample_procedure
    # => [flag]
    assert.err=ERR_FLAG_IS_FALSE
```

### Propagated Panics

When calling other procedures that may panic:

**Simple case (< 4 conditions):** List the specific conditions:

```masm
#! Description, inputs, etc.
#! Panics if:
#! - flag is false.
proc sample_procedure
    # => [flag]
    exec.another_procedure # this procedure may panic
end

proc another_procedure
    # => [flag]
    assert.err=ERR_FLAG_IS_FALSE
end
```

**Complex case (4+ conditions):** Reference the subprocedure's validation rather than re-listing every condition:

```masm
#! Description, inputs, etc.
#! Panics if:
#! - another_procedure validation fails.
proc sample_procedure
    # => [flag_1, flag_2, flag_3, flag_4]
    exec.another_procedure # this procedure may panic
end

#! Description, inputs, etc.
#! Panics if:
#! - flag_1 is false.
#! - flag_2 is false.
#! - flag_3 is false.
#! - flag_4 is false.
proc another_procedure
    # => [flag_1, flag_2, flag_3, flag_4]
    assert.err=ERR_FLAG_1_IS_FALSE
    assert.err=ERR_FLAG_2_IS_FALSE
    assert.err=ERR_FLAG_3_IS_FALSE
    assert.err=ERR_FLAG_4_IS_FALSE
end
```

### No Panics

Omit the "Panics if:" section entirely if the procedure cannot panic.

## Invocation Types

Specify how the procedure should be invoked. The value matches the MASM instruction a caller uses to enter the procedure:

| Value | Used by callers as | When to use |
|---|---|---|
| `exec` | `exec.<proc>` | Standard inline call. Shares the caller's stack and context; no padding requirement. |
| `call` | `call.<proc>` | Cross-context call (e.g. into another account). Enters at stack depth 16 — Inputs/Outputs must show `pad(N)` to total 16 (see masm-padding). |
| `dynexec` | `dynexec` / `dyncall` on a procedure root | Dynamically-dispatched entry point, e.g. the kernel API procedures reached through the transaction kernel's dispatch. |
| `syscall` | `syscall.<proc>` | Rare; a procedure invoked directly as a kernel syscall. |

```masm
#! Invocation: exec
```

```masm
#! Invocation: call
```

For existing procedures, pick the value that matches how callers invoke them: `call` when invoked via `call.<procedure_name>`, `exec` for `exec.<procedure_name>`. `exec` and `call` dominate ordinary module code; `dynexec` dominates the kernel API module (`kernels/transaction/lib/api.masm`), and `syscall` appears only once in protocol source.

### Kernel-internal modules omit the line

Most procedures in the kernel's internal modules (`crates/miden-protocol/asm/kernels/transaction-core/src/*`) carry no `Invocation:` line — `account.masm` has 38 `pub proc` and zero annotations. It is not absolute: `transaction-core/src/tx.masm` annotates six procedures (five `exec`, one `dynexec`). The kernel's public API module (`kernels/transaction/lib/api.masm`) does annotate its procedures. Follow the surrounding module.

## Prose Conventions

Doc comments are read by people unfamiliar with the change that introduced them. Keep the prose general, accurate, and consistent with the rest of the codebase.

### Reuse existing terminology

Use the vocabulary already established in surrounding modules and doc comments. Do not coin new terms or borrow colloquialisms for a concept that already has a name. For example, a value written to a local is "stored" or "saved" (matching `loc_storew_le`), not "stashed"; describe what code does plainly rather than labelling it ("load-bearing", "the real check", and similar).

### Document the procedure, not the change

Describe the procedure as it currently behaves, for a reader who has never seen the PR that added or modified it. Avoid PR narrative, rationale for a recent fix, and framing such as "this is the X that prevents Y". State what the procedure does and what it guarantees.

### Stay at this layer's abstraction

Describe behavior in terms of this procedure and its inputs and outputs. Do not explain how a lower layer (for example the kernel, or a specific syscall) implements or enforces something — that is an implementation detail that may change. Panic bullets in particular state the condition in domain terms ("the asset does not belong to this faucet"), not the mechanism or which layer raises it.

## Validation Checklist

- [ ] Description starts with a capitalized present-tense verb and the first sentence ends with a period. Canonical verbs observed in protocol source: `Returns`, `Gets`, `Computes`, `Burns`, `Creates`, `Increments`, `Copies`, `Asserts`, `Verifies`, `Hashes`, `Adds`, `Removes`.
- [ ] Inputs and Outputs use correct stack notation
- [ ] Where section defines every stack item that needs description beyond the Description line, and is omitted entirely when the Description alone covers every item
- [ ] Words are UPPERCASE, felts are lowercase; composite braces are stack-top-first (`account_id_{suffix,prefix}`), matching the surrounding file's brace spacing
- [ ] Panics section lists direct asserts and propagated errors, describing conditions rather than `ERR_*` identifiers
- [ ] Complex panic propagation references the subprocedure (e.g. "<procedure> validation fails")
- [ ] Invocation type specified (`exec`, `call`, `dynexec`, or `syscall`), or omitted to match a kernel-internal module
- [ ] For `call`: padding shown in Inputs/Outputs (see masm-padding skill)
- [ ] Prose reuses existing terminology; no coined terms or colloquialisms
- [ ] Describes the procedure's current behavior, not the change that introduced it
- [ ] No lower-layer (kernel/syscall) implementation details; panic bullets describe conditions in domain terms
