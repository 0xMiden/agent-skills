---
name: masm-formatting
description: Orchestrator for MASM formatting in miden-base and miden-vm. Consolidates capitalization, the (N) span family, cross-repo doc comment divergences (plural vs singular Inputs/Outputs, Panics if vs # Panics, named vs string assertion errors), the `Cycles:` section, chained u32 assertion guards, and description verbs across the prerequisite MASM skills. Use when editing, reviewing, or creating .masm files.
---

# MASM Formatting

This skill is an orchestrator and gap-filler. It depends on the existing MASM skills and only documents conventions they do not cover plus cross-skill integration guidance. It does not restate prerequisite rules.

## Prerequisites

Consult these skills first.

- `masm-inline-comments` – inline `#` comments: lowercase start, `# => [...]` stack state, avoid commenting obvious operations.
- `masm-doc-comments` – `#!` doc block: Description, Inputs, Outputs, Where, Panics if, Invocation; singular `is` for single items, plural `are` for composites.
- `masm-padding` – `pad(N)` rules for `call` vs `exec` procedures.

## When to Use

Apply this skill alongside the prerequisites whenever you edit a `.masm` file. Reach for it specifically for capitalization questions, `(N)` span notation beyond `pad(N)`, divergences between miden-base style and miden-vm style, the `Cycles:` section, and chained `u32assert2` guards.

## 1. Capitalization (three tiers)

| Kind | Style | Example |
|---|---|---|
| Word (4 felts) or Word-shaped commitment/root/constant | `UPPER_SNAKE_CASE` | `ASSET_KEY`, `ASSET_VALUE`, `FOREIGN_PROC_ROOT`, `EMPTY_WORD` |
| Single felt | `lower_snake_case` | `final_nonce`, `note_idx`, `amount` |
| Multi-felt composite grouped in one name | `lower_snake_case_{part1,part2}` | `account_id_{suffix,prefix}`, `faucet_id_{suffix,prefix}`, `sender_{suffix,prefix}` |

Canonical citations:

- Word-shaped: `miden-base/crates/miden-protocol/asm/protocol/faucet.masm:10` (`[ASSET_KEY, ASSET_VALUE]`), and the matching `Where:` bullets at `miden-base/crates/miden-protocol/asm/protocol/faucet.masm:14-15`.
- Single felt: `miden-base/crates/miden-protocol/asm/protocol/faucet.masm:9` (`[amount]`) and `miden-base/crates/miden-protocol/asm/protocol/native_account.masm:49` (`[final_nonce]`).
- Composite: `miden-base/crates/miden-protocol/asm/protocol/native_account.masm:22`, `miden-base/crates/miden-protocol/asm/protocol/asset.masm:40`, `miden-base/crates/miden-protocol/asm/protocol/active_note.masm:175`.

Composite items always use `are` in `Where:` since they name a group of felts, per `masm-doc-comments`. Brace spacing is inconsistent in source: `miden-base/crates/miden-protocol/asm/protocol/active_account.masm:501` uses `{suffix,prefix}` while `miden-base/crates/miden-protocol/asm/protocol/active_account.masm:537` uses `{suffix, prefix}`. Match the surrounding file; do not reformat existing braces.

## 2. Stack span notation: the `(N)` family

`pad(N)`, owned by `masm-padding`, is one member of a broader span-size family. `(N)` after an item name denotes a span of N felts, not a Word. Words stay UPPERCASE without `(N)`; spans stay lowercase.

| Use | Example | Citation |
|---|---|---|
| Padding span | `pad(12)`, `pad(15)` | `miden-base/crates/miden-protocol/asm/protocol/native_account.masm:65` |
| Known-size felt-array parameter | `foreign_procedure_inputs(16)`, `foreign_procedure_outputs(16)` | `miden-base/crates/miden-protocol/asm/protocol/tx.masm:214-215` |
| Mixed inline tracker with two `(N)` spans | `# => [pad(16), foreign_procedure_inputs(15)]` | `miden-base/crates/miden-protocol/asm/protocol/tx.masm:256` |
| Variable-length remainder | trailing `, ...` | `miden-vm/crates/lib/core/asm/crypto/hashes/poseidon2.masm:118-119` |

## 3. Doc comment divergences across repos

`masm-doc-comments` defines the `#!` block layout. Real source shows miden-base and miden-vm use different but active variants of several sections. Document both, do not flatten.

| Dimension | miden-base (uniform) | miden-vm (mixed across files) |
|---|---|---|
| Inputs/Outputs heading | Plural `Inputs:` / `Outputs:` | Both are active: plural in `mem.masm`, singular `Input:` / `Output:` in `crypto/hashes/poseidon2.masm` |
| Panics heading | `Panics if:` with bullet list | `# Panics` markdown-style heading with prose (`mem.masm`), also `Panics if:` (`math/u128.masm`) |
| `Invocation:` line | Standard (`exec`, `call`, `dynexec`) | Present in newer files (`math/u128.masm`), absent in older `crypto/hashes/*.masm` |
| `Cycles:` section | Rare but present (multi-line form at `miden-base/crates/miden-protocol/asm/protocol/note.masm:28-30`) | Standard on most public procedures |

Row citations:

- Plural (base): `miden-base/crates/miden-protocol/asm/protocol/faucet.masm:9-10`.
- `Panics if:` (base): `miden-base/crates/miden-protocol/asm/protocol/native_account.masm:55-59`.
- `Invocation: exec` (base): `miden-base/crates/miden-protocol/asm/protocol/native_account.masm:61`.
- Plural (vm): `miden-vm/crates/lib/core/asm/mem.masm:9-10`.
- Singular (vm): `miden-vm/crates/lib/core/asm/crypto/hashes/poseidon2.masm:118-119`.
- `# Panics` (vm): `miden-vm/crates/lib/core/asm/mem.masm:17-20`.
- `Invocation: exec` (vm): `miden-vm/crates/lib/core/asm/math/u128.masm:465`.

Recommendation: when writing new miden-protocol-style code (contracts, account components, tutorial MASM), follow the miden-base uniform style. When editing inside an existing miden-vm file, match that file's local style.

## 4. The `Cycles:` section

`Cycles:` is common on public procedures in `miden-vm` and rare but present in `miden-base`. Match the local file or repo style rather than applying a blanket rule. Three observed formats:

Fixed count:

```masm
#! Cycles: 12
```

`miden-vm/crates/lib/core/asm/crypto/hashes/poseidon2.masm:11`.

Linear expression with named variable:

```masm
#! Cycles: 4 + 3 * words, where `words` is the `start_addr - end_addr`
```

`miden-vm/crates/lib/core/asm/crypto/hashes/poseidon2.masm:71`.

Conditional multi-line block introduced by an empty `#! Cycles:` header, then per-condition bullets:

```masm
#! Cycles:
#! - even words: 53 cycles + 3 * words
#! - odd words: 65 cycles + 3 * words
#! where `words` is the `start_addr - end_addr - 1`
```

`miden-vm/crates/lib/core/asm/crypto/hashes/poseidon2.masm:121-124`. The same multi-line form appears in `miden-base` at `miden-base/crates/miden-protocol/asm/protocol/note.masm:28-30`.

A prose variant `Total cycles:` also appears, e.g. `miden-vm/crates/lib/core/asm/mem.masm:22`.

## 5. Assertions and error messages

Two active forms:

- Named constant (dominant in miden-base): `assert.err=ERR_CONSTANT_NAME`. Examples: `miden-base/crates/miden-protocol/asm/protocol/asset.masm:54`, `miden-base/crates/miden-protocol/asm/protocol/active_account.masm:512`.
- Inline string (dominant in miden-vm): `assert.err="lowercase descriptive message"`. Examples: `miden-vm/crates/lib/core/asm/mem.masm:31`, `miden-vm/crates/lib/core/asm/stark/random_coin.masm:102`.

Chained guard pattern, called out in `0xMiden/agent-skills#6`:

```masm
u32assert2 u32lte.MAX_LEAF_SIZE assert.err="invalid leaf: larger than maximum size of 8192"
```

`miden-vm/crates/lib/core/asm/collections/smt.masm:74`. Also:

```masm
u32assert2 u32lt assert.err="comparison failed: norm bound"
```

`miden-vm/crates/lib/core/asm/crypto/dsa/falcon512_poseidon2.masm:676`. Multiple u32 guards are chained on one line followed by a single `assert.err=` attachment.

Doc rule: `Panics if:` bullets describe the condition, not the error identifier. See `miden-base/crates/miden-protocol/asm/protocol/native_account.masm:55-59` which lists `the nonce has already been incremented.` without naming the underlying `ERR_*` constant.

## 6. Description verbs

Doc descriptions start with a capitalized present-tense verb and end the first sentence with a period. Representative canonical verbs:

| Verb | Citation |
|---|---|
| Returns | `miden-base/crates/miden-protocol/asm/protocol/native_account.masm:16` |
| Creates | `miden-base/crates/miden-protocol/asm/protocol/asset.masm:33` |
| Increments | `miden-base/crates/miden-protocol/asm/protocol/native_account.masm:46` |
| Computes | `miden-base/crates/miden-protocol/asm/protocol/native_account.masm:80` |
| Copies | `miden-vm/crates/lib/core/asm/mem.masm:5` |
| Asserts | `miden-vm/crates/lib/core/asm/math/u64.masm:3` |

Multi-sentence elaboration continues on following `#!` lines (see `miden-vm/crates/lib/core/asm/mem.masm:5-7`).

## 7. Inline stack tracker integration

`masm-inline-comments` owns the lowercase-start and comment-only-non-obvious rules. This skill adds one integration rule: an inline `# => [...]` tracker uses the exact same item names and capitalization as the `#!` doc block, including `(N)` span notation.

Canonical progression at `miden-base/crates/miden-protocol/asm/protocol/native_account.masm:63-74`:

```masm
padw padw padw push.0.0.0
# => [pad(15)]

push.ACCOUNT_INCR_NONCE_OFFSET
# => [offset, pad(15)]

syscall.exec_kernel_proc
# => [final_nonce, pad(15)]

swap.15 dropw dropw dropw drop drop drop
# => [final_nonce]
```

Both `offset` (a single felt) and `final_nonce` (the single-felt output named in the doc block at line 49) appear lowercase in the tracker. The `pad(15)` span uses the same `(N)` form as the doc block.

## 8. Compact before / after

A malformed miden-protocol-style procedure followed by its corrected form. Every fix is justified by an adjacent citation.

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

- Capitalized description verb and trailing period: pattern at `miden-base/crates/miden-protocol/asm/protocol/native_account.masm:46`.
- `Input:`/`Output:` rewritten to plural: pattern at `miden-base/crates/miden-protocol/asm/protocol/faucet.masm:9-10` and `miden-base/crates/miden-protocol/asm/protocol/native_account.masm:48-49`.
- `Final_Nonce` rewritten to `final_nonce` (single felt is lowercase): pattern at `miden-base/crates/miden-protocol/asm/protocol/native_account.masm:49`.
- `Where:` sentence now ends with a period: pattern at `miden-base/crates/miden-protocol/asm/protocol/native_account.masm:52-53` (owned by `masm-doc-comments`).
- `Invocation: exec` line added: pattern at `miden-base/crates/miden-protocol/asm/protocol/native_account.masm:61`.
- Inline comment lowercased: rule owned by `masm-inline-comments`.

## References

| Rule | Citations |
|---|---|
| Word UPPERCASE | `miden-base/crates/miden-protocol/asm/protocol/faucet.masm:10,14-15` |
| Single felt lowercase | `miden-base/crates/miden-protocol/asm/protocol/native_account.masm:49` |
| Composite `{part1,part2}` | `miden-base/crates/miden-protocol/asm/protocol/native_account.masm:22`; `miden-base/crates/miden-protocol/asm/protocol/asset.masm:40`; `miden-base/crates/miden-protocol/asm/protocol/active_note.masm:175` |
| Brace spacing variance | `miden-base/crates/miden-protocol/asm/protocol/active_account.masm:501` vs `miden-base/crates/miden-protocol/asm/protocol/active_account.masm:537` |
| `(N)` span family | `miden-base/crates/miden-protocol/asm/protocol/tx.masm:214-215`; `miden-base/crates/miden-protocol/asm/protocol/tx.masm:256` |
| Plural `Inputs:`/`Outputs:` (base) | `miden-base/crates/miden-protocol/asm/protocol/faucet.masm:9-10` |
| Singular `Input:`/`Output:` (vm) | `miden-vm/crates/lib/core/asm/crypto/hashes/poseidon2.masm:118-119` |
| `Panics if:` | `miden-base/crates/miden-protocol/asm/protocol/native_account.masm:55-59` |
| `# Panics` | `miden-vm/crates/lib/core/asm/mem.masm:17-20` |
| `Invocation:` (base) | `miden-base/crates/miden-protocol/asm/protocol/native_account.masm:61` |
| `Invocation:` (vm) | `miden-vm/crates/lib/core/asm/math/u128.masm:465` |
| `Cycles:` fixed | `miden-vm/crates/lib/core/asm/crypto/hashes/poseidon2.masm:11` |
| `Cycles:` linear | `miden-vm/crates/lib/core/asm/crypto/hashes/poseidon2.masm:71` |
| `Cycles:` multi-line | `miden-vm/crates/lib/core/asm/crypto/hashes/poseidon2.masm:121-124`; `miden-base/crates/miden-protocol/asm/protocol/note.masm:28-30` |
| `Total cycles:` prose | `miden-vm/crates/lib/core/asm/mem.masm:22` |
| `assert.err=ERR_*` | `miden-base/crates/miden-protocol/asm/protocol/asset.masm:54` |
| `assert.err="..."` | `miden-vm/crates/lib/core/asm/mem.masm:31`; `miden-vm/crates/lib/core/asm/stark/random_coin.masm:102` |
| Chained u32 guard | `miden-vm/crates/lib/core/asm/collections/smt.masm:74`; `miden-vm/crates/lib/core/asm/crypto/dsa/falcon512_poseidon2.masm:676` |
| Description verbs | `miden-base/crates/miden-protocol/asm/protocol/native_account.masm:16,46,80`; `miden-base/crates/miden-protocol/asm/protocol/asset.masm:33`; `miden-vm/crates/lib/core/asm/mem.masm:5`; `miden-vm/crates/lib/core/asm/math/u64.masm:3` |
| Inline tracker integration | `miden-base/crates/miden-protocol/asm/protocol/native_account.masm:63-74` |
