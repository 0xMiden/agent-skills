---
name: masm-formatting
description: Orchestrator for MASM formatting in 0xMiden/protocol and 0xMiden/miden-vm. Consolidates capitalization, the (N) span family, cross-repo doc comment divergences (plural vs singular Inputs/Outputs, Panics if vs # Panics, named vs string assertion errors), the `Cycles:` section, chained u32 assertion guards, and description verbs across the prerequisite MASM skills. Use when editing, reviewing, or creating .masm files.
---

# MASM Formatting

This skill is an orchestrator and gap-filler. It depends on the existing MASM skills and only documents conventions they do not cover plus cross-skill integration guidance. It does not restate prerequisite rules.

## Prerequisites

Consult these skills first.

- `masm-inline-comments` – inline `#` comments: lowercase start, `# => [...]` stack state, avoid commenting obvious operations.
- `masm-doc-comments` – `#!` doc block: Description, Inputs, Outputs, Where, Panics if, Invocation; singular `is` for single items, plural `are` for composites.
- `masm-padding` – `pad(N)` rules for `call` vs `exec` procedures.

## When to Use

Apply this skill alongside the prerequisites whenever you edit a `.masm` file. Reach for it specifically for capitalization questions, `(N)` span notation beyond `pad(N)`, divergences between protocol style and miden-vm style, the `Cycles:` section, and chained `u32assert2` guards.

The conventions below are derived from canonical MASM in [protocol](https://github.com/0xMiden/protocol) and [miden-vm](https://github.com/0xMiden/miden-vm). They apply to anyone writing MASM in the Miden ecosystem, including community contracts, tutorial examples, and library MASM, not just code that lives inside those two repos.

## 1. Capitalization (three tiers)

| Kind | Style | Example |
|---|---|---|
| Word (4 felts) or Word-shaped commitment / root / constant | `UPPER_SNAKE_CASE` | `ASSET_KEY`, `ASSET_VALUE`, `FOREIGN_PROC_ROOT`, `EMPTY_WORD` |
| Single felt | `lower_snake_case` | `final_nonce`, `note_idx`, `amount` |
| Multi-felt composite grouped in one name | `lower_snake_case_{part1,part2}` | `account_id_{suffix,prefix}`, `faucet_id_{suffix,prefix}`, `sender_{suffix,prefix}` |

Composite items always use `are` in `Where:` since they name a group of felts, per `masm-doc-comments`. Brace spacing and part order are inconsistent in source: both `{suffix,prefix}` and `{suffix, prefix}` (with a space) appear, and both `{suffix,prefix}` (suffix-first, the more common form in protocol procs) and `{prefix,suffix}` (prefix-first) appear, sometimes within the same file. Match the surrounding file; do not reformat existing braces.

Use `EMPTY_WORD` to denote the all-zero Word `[0, 0, 0, 0]` in stack trackers, `Where:` bullets, and prose comments (e.g. `# => [..., EMPTY_WORD, ...]`). It is a naming convention used in protocol-style code, not a defined constant in source; the point is to convey *meaning* (absence of data) rather than the literal numeric value.

Canonical references in [protocol](https://github.com/0xMiden/protocol): `crates/miden-protocol/asm/protocol/faucet.masm`, `native_account.masm`, `asset.masm`, `active_note.masm`, `active_account.masm`.

## 2. Stack span notation: the `(N)` family

`pad(N)`, owned by `masm-padding`, is one member of a broader span-size family. `(N)` after an item name denotes a span of N felts, not a Word. Words stay UPPERCASE without `(N)`; spans stay lowercase.

| Use | Example |
|---|---|
| Padding span | `pad(12)`, `pad(15)` |
| Known-size felt-array parameter | `foreign_procedure_inputs(16)`, `foreign_procedure_outputs(16)` |
| Mixed inline tracker with two `(N)` spans | `# => [pad(16), foreign_procedure_inputs(15)]` |
| Variable-length remainder | trailing `, ...` |

Canonical references: `crates/miden-protocol/asm/protocol/tx.masm` in [protocol](https://github.com/0xMiden/protocol) for the `(N)` family in both doc blocks and inline trackers and for the `, ...` trailing-remainder form; `crates/lib/core/asm/crypto/hashes/poseidon2.masm` and `crates/lib/core/asm/stark/deep_queries.masm` in [miden-vm](https://github.com/0xMiden/miden-vm) for further `, ...` examples.

## 3. Doc comment divergences across repos

`masm-doc-comments` defines the `#!` block layout. Real source shows protocol and miden-vm use different but active variants of several sections. Document both, do not flatten.

| Dimension | protocol (uniform) | miden-vm (mixed across files) |
|---|---|---|
| Inputs/Outputs heading | Plural `Inputs:` / `Outputs:` | Both are active: plural in `mem.masm`, singular `Input:` / `Output:` in `crypto/hashes/poseidon2.masm` |
| Panics heading | `Panics if:` with bullet list | `# Panics` markdown-style heading with prose (`mem.masm`), also `Panics if:` (`math/u128.masm`) |
| `Invocation:` line | Standard (`exec`, `call`, `dynexec`) | Present in newer files (`math/u128.masm`), absent in older `crypto/hashes/*.masm` |
| `Cycles:` section | Rare but present (multi-line form in `protocol/note.masm`) | Standard on most public procedures |

Recommendation: when writing new protocol-style code (contracts, account components, tutorial MASM), follow the protocol uniform style. When editing inside an existing miden-vm file, match that file's local style.

Canonical references: `crates/miden-protocol/asm/protocol/faucet.masm` and `native_account.masm` in [protocol](https://github.com/0xMiden/protocol); `crates/lib/core/asm/mem.masm`, `crates/lib/core/asm/crypto/hashes/poseidon2.masm`, and `crates/lib/core/asm/math/u128.masm` in [miden-vm](https://github.com/0xMiden/miden-vm).

## 4. The `Cycles:` Section

`Cycles:` is common on public procedures in `miden-vm` and rare but present in `protocol`. Match the local file or repo style rather than applying a blanket rule. Three observed shapes:

Fixed count:

```masm
#! Cycles: 12
```

Linear expression with a named variable:

```masm
#! Cycles: <fixed> + <coef> * <var>, where `<var>` is <definition>
```

Conditional multi-line block introduced by an empty `#! Cycles:` header, followed by per-condition bullets:

```masm
#! Cycles:
#! - <condition A>: <expr A>
#! - <condition B>: <expr B>
#! where `<var>` is <definition>
```

A prose variant `Total cycles: ...` is also used in some miden-vm files.

Canonical references: `crates/lib/core/asm/crypto/hashes/poseidon2.masm` in [miden-vm](https://github.com/0xMiden/miden-vm) for all three shapes; `crates/miden-protocol/asm/protocol/note.masm` in [protocol](https://github.com/0xMiden/protocol) for the multi-line form outside of stdlib; `crates/lib/core/asm/mem.masm` in miden-vm for the `Total cycles:` prose variant.

## 5. Assertions and Error Messages

Two active forms:

- Named constant (dominant in protocol): `assert.err=ERR_CONSTANT_NAME`.
- Inline string (dominant in miden-vm): `assert.err="lowercase descriptive message"`.

Chained guard pattern, called out in `0xMiden/agent-skills#6`:

```masm
u32assert2 u32lte.MAX_LEAF_SIZE assert.err="invalid leaf: larger than maximum size of 8192"
```

Multiple u32 guards are chained on one line followed by a single `assert.err=` attachment. The same pattern with `u32lt` / `u32gte` and other comparators is also common.

Doc rule: `Panics if:` bullets describe the condition, not the error identifier. A bullet like `the nonce has already been incremented.` is preferred over `ERR_ACCOUNT_NONCE_CAN_ONLY_BE_INCREMENTED_ONCE.`.

Canonical references: `crates/miden-protocol/asm/protocol/asset.masm` and `active_account.masm` in [protocol](https://github.com/0xMiden/protocol) for `assert.err=ERR_*`; `crates/lib/core/asm/mem.masm`, `crates/lib/core/asm/stark/random_coin.masm`, and `crates/lib/core/asm/collections/smt.masm` in [miden-vm](https://github.com/0xMiden/miden-vm) for inline string forms and the chained guard pattern.

## 6. Description Verbs

Doc descriptions start with a capitalized present-tense verb and end the first sentence with a period. Representative canonical verbs used in source: `Returns`, `Creates`, `Increments`, `Computes`, `Copies`, `Asserts`, `Verifies`, `Hashes`, `Adds`, `Removes`. Multi-sentence elaboration continues on following `#!` lines.

Canonical references: `crates/miden-protocol/asm/protocol/native_account.masm` and `asset.masm` in [protocol](https://github.com/0xMiden/protocol); `crates/lib/core/asm/mem.masm` and `crates/lib/core/asm/math/u64.masm` in [miden-vm](https://github.com/0xMiden/miden-vm).

## 7. Inline Stack Tracker Integration

`masm-inline-comments` owns the lowercase-start and comment-only-non-obvious rules. This skill adds one integration rule: an inline `# => [...]` tracker uses the exact same item names, capitalization, and `(N)` span notation as the `#!` doc block.

Synthetic illustration (not a real proc; see Canonical reference below for a real example):

```masm
#! Outputs: [final_nonce]
pub proc example
    # => [final_nonce, pad(15)]
end
```

In the illustration above, the single-felt name `final_nonce` is identical in the doc block and the inline tracker, and the `pad(15)` span follows the `(N)` family rule. The same principle holds in real source: Word-sized items are UPPERCASE in both places, and composite names like `account_id_{suffix,prefix}` decompose into their underlying felts in inline trackers (e.g. `account_id_suffix`, `account_id_prefix`).

Canonical reference: any public proc in `crates/miden-protocol/asm/protocol/native_account.masm` in [protocol](https://github.com/0xMiden/protocol) shows this pattern end-to-end.

## 8. Compact Before / After

A malformed protocol-style procedure followed by its corrected form. Every fix is justified by the rule it applies.

Before:

```masm
#! returns the nonce of the native account
#!
#! Input:  []
#! Output: [Final_Nonce]
#!
#! Where:
#! - Final_Nonce is the nonce of the account
pub proc get_nonce
    # Get the nonce
    push.ACCOUNT_GET_NONCE_OFFSET
    syscall.exec_kernel_proc
end
```

After:

```masm
#! Returns the nonce of the native account.
#!
#! Inputs:  []
#! Outputs: [final_nonce]
#!
#! Where:
#! - final_nonce is the nonce of the account.
#!
#! Invocation: exec
pub proc get_nonce
    # get the nonce
    push.ACCOUNT_GET_NONCE_OFFSET
    syscall.exec_kernel_proc
end
```

Fix log:

- Capitalized description verb and trailing period: §6 (Description Verbs); pattern across procs in `native_account.masm` in [protocol](https://github.com/0xMiden/protocol).
- `Input:`/`Output:` rewritten to plural `Inputs:`/`Outputs:` for protocol-style code: §3 (Doc Comment Divergences); pattern in `faucet.masm` in protocol.
- `Final_Nonce` rewritten to `final_nonce` (single felt is `lower_snake_case`): §1 (Capitalization).
- `Where:` sentence ends with a period: rule owned by `masm-doc-comments`.
- `Invocation: exec` line added: §3 (protocol uniform style includes `Invocation:`).
- Inline comment lowercased: rule owned by `masm-inline-comments`.

## Canonical References

The full set of source files referenced above, by purpose. Repos: [protocol](https://github.com/0xMiden/protocol) and [miden-vm](https://github.com/0xMiden/miden-vm).

| File | What to study |
|---|---|
| `crates/miden-protocol/asm/protocol/native_account.masm` (protocol) | end-to-end protocol-style proc: composite `{suffix,prefix}`, `Panics if:`, `Invocation:`, inline `pad(N)` tracking |
| `crates/miden-protocol/asm/protocol/faucet.masm` (protocol) | Word UPPERCASE in `Inputs:`/`Outputs:`, plural heading style |
| `crates/miden-protocol/asm/protocol/tx.masm` (protocol) | `(N)` span family in both doc blocks and inline trackers |
| `crates/miden-protocol/asm/protocol/asset.masm` (protocol) | `assert.err=ERR_*` named-constant style |
| `crates/miden-protocol/asm/protocol/active_account.masm` (protocol) | brace-spacing variance and additional `assert.err=ERR_*` |
| `crates/miden-protocol/asm/protocol/note.masm` (protocol) | multi-line `Cycles:` outside of stdlib |
| `crates/lib/core/asm/crypto/hashes/poseidon2.masm` (miden-vm) | singular `Input:`/`Output:` and all three `Cycles:` shapes |
| `crates/lib/core/asm/mem.masm` (miden-vm) | plural `Inputs:`/`Outputs:` co-existing in vm, `# Panics` heading, `Total cycles:` prose variant |
| `crates/lib/core/asm/math/u128.masm` (miden-vm) | `Invocation: exec` present in newer vm files |
| `crates/lib/core/asm/collections/smt.masm` (miden-vm) | chained `u32assert2 u32lte.<CONST> assert.err="..."` guard |
| `crates/lib/core/asm/math/u64.masm` (miden-vm) | `Asserts` description verb |
| `crates/lib/core/asm/stark/random_coin.masm` (miden-vm) | inline string `assert.err="..."` style |
