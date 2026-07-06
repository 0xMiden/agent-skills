---
name: advice-provider-hygiene
description: Use when writing kernel, account, or note MASM code that reads from or writes to the advice provider (advice stack / advice map) — validate advice data.
---

# Advice Provider Hygiene

## Rules

The advice provider is untrusted input supplied by the (potentially adversarial) prover. Any kernel, account, or note procedure that touches it must follow three rules.

### 1. Validate advice data against a commitment

Before consuming data loaded from the advice provider:

1. Read the data from the advice stack / advice map into memory.
2. Compute its hash with Poseidon2 over the loaded region.
3. Assert the computed hash equals an expected commitment that the kernel already trusts — on-chain storage, a prior input, or a value already on the stack.

Do not consume advice data before this check passes. The advice provider's only role is to supply witness data for commitments the kernel has already received.

### 2. Key advice map entries by content hash

When inserting into the advice map, the key must be a hash of the value it indexes (or a derived commitment of the same data):

- Use `Poseidon2(value)` (or whichever commitment matches the consumer's check) as the key.
- Do not hard-code keys like `0x0000_0000_0000_0001`, `ADVICE_KEY_NOTE_DATA`, or per-procedure magic constants.

Readers retrieve the entry by recomputing the same hash from data they already trust; rule 1's commitment check binds the lookup result to that trusted hash.

### 3. Missing advice is an error

A missing advice-map entry, an empty advice stack, or an absent required value is an error — not a default. `adv.push_mapval` / `adv.push_mapvaln` already abort execution when the key is missing (the VM returns `MapKeyNotFound`), so don't paper over it with a fallback. When you branch on presence yourself, surface the failure with `assert.err=ERR_...`. Don't substitute zero / empty / a fallback and continue.

## Why

The advice provider is filled by a potentially adversarial prover. Validating every value against a commitment the kernel already trusts, keying map entries by content hash, and erroring on missing entries are what stop untrusted advice from silently corrupting the kernel's invariants.

## Examples

Advice data is tied to a commitment by piping it into memory. There are two mechanisms, and they differ in whether the commitment check happens for you.

### Piping words to memory — you must validate

`adv_pipe`, `adv_loadw`, and `mem::pipe_double_words_to_memory` copy advice data into memory but do *not* check it against any commitment. Hash the loaded region with Poseidon2 yourself and assert it equals a commitment the kernel already trusts.

```masm
# Good: pipe words while hashing, then assert against a trusted commitment.
# This is the input-note-assets path from the transaction prologue:
# pipe_double_words_to_memory runs `adv_pipe exec.poseidon2::permute` internally
# (no commitment check of its own), then you squeeze and assert.
exec.poseidon2::init_no_padding
exec.mem::pipe_double_words_to_memory
exec.poseidon2::squeeze_digest
# => [COMPUTED_ASSETS_COMMITMENT, ...]
exec.memory::get_input_note_assets_commitment
assert_eqw.err=ERR_PROLOGUE_PROVIDED_INPUT_ASSETS_INFO_DOES_NOT_MATCH_ITS_COMMITMENT

# Good: drive the adv_pipe + permute loop yourself, squeeze, then assert.
# (The block-data prologue squeezes a SUB_COMMITMENT here, merges it with the
#  trusted NOTE_ROOT to form the block commitment, and only THEN asserts — i.e.
#  the squeezed digest is combined with trusted data before the equality check.)
exec.poseidon2::init_no_padding
adv_pipe exec.poseidon2::permute
# ... one `adv_pipe exec.poseidon2::permute` per piped block ...
exec.poseidon2::squeeze_digest
# => [SUB_COMMITMENT, ...]  (combine with trusted data as needed, e.g. merge a root)
# ... eventually ...
exec.memory::get_block_commitment
assert_eqw.err=ERR_PROLOGUE_GLOBAL_INPUTS_PROVIDED_DO_NOT_MATCH_BLOCK_COMMITMENT

# Bad: pipe advice into memory and use it without the hash/assert step
adv_pipe
# ... data could be anything the prover supplied
```

### Piping a preimage to memory — validated for you

`mem::pipe_preimage_to_memory` takes the commitment on the stack, copies the preimage from the advice stack into memory, and asserts its sequential Poseidon2 hash equals that commitment — all in one step.

```masm
# Good: data is checked against COMMITMENT as part of the pipe
# stack: [num_words, write_ptr, COMMITMENT]
exec.mem::pipe_preimage_to_memory
# => [write_ptr'] — data in memory is guaranteed to match COMMITMENT
```

### Content-addressed keys and missing entries

An advice-map key is a full word (4 felts) sitting on top of the operand stack. `adv.push_mapval` reads that word as the key and pushes the looked-up value onto the *advice* stack (the operand stack is unchanged), so you typically follow it with `adv_loadw` or a pipe to bring the value into the operand stack or memory.

```masm
# Good: key the advice map entry by the commitment itself
# (NOTE_DATA_COMMITMENT is the trusted word already on the operand-stack top)
adv.push_mapval          # value pushed onto the advice stack, keyed by the commitment word
adv_loadw                # => [NOTE_DATA, ...]  pull the value into the operand stack

# Bad: hard-coded magic key (a key is a Word — 4 felts — and this one is meaningless)
push.0x0001.0x0000.0x0000.0x1234
adv.push_mapval

# Good: a missing required entry is an error.
# adv.has_mapkey pushes the presence flag onto the ADVICE stack, so move it to the
# operand stack with adv_push before asserting.
# stack: [KEY, ...]
adv.has_mapkey           # advice stack: [has_key, ...]; operand stack unchanged
adv_push                 # => [has_key, KEY, ...]
assert.err=ERR_MISSING_REQUIRED_ADVICE

# (adv.push_mapval itself already aborts with MapKeyNotFound on an absent key —
#  never assume it returns zero and continue.)
```

The Rust analog is to return `Err` on bad or missing external input rather than panicking or silently defaulting.
