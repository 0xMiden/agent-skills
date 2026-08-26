---
name: masm-locals-over-globals
description: Use when a MASM procedure needs temporary scratch storage — keep it in procedure-local memory so it cannot collide in a shared global region.
---

# Prefer Procedure Locals for MASM Scratch Storage

## Rule

When a MASM procedure needs scratch storage that lives only for the duration of one invocation, use procedure-local memory rather than allocating in a shared global memory region.

Declare how many locals the procedure needs with `@locals(N)` directly above the procedure, then read and write them:

- `loc_store` / `loc_load` for a single element.
- `loc_storew_le` / `loc_loadw_le` (or the `_be` variants) for a full word.

There are no bare `loc_storew` / `loc_loadw` instructions: writing one is a parse error (`deprecated instruction: 'loc_storew' has been removed`). Always name the byte order with the `_le` / `_be` variant.

Global memory regions are reserved for state that crosses procedure boundaries (kernel inputs, account data, advice-keyed state). Stashing per-call scratch there leaks an implementation detail into a shared namespace and ties the procedure to a fixed address.

This ranks locals against globals only. Scratch that fits on the operand stack — loop counters, pointers, indices — belongs there rather than in a local; see `cheap-masm-equivalents`.

## Why

Procedure locals are addressed relative to the procedure's frame pointer (FMP) and bounded by its declared `@locals(N)` count: the assembler emits a prologue that advances the FMP to allocate the frame on entry and an epilogue that restores it on exit, so two callers of the same procedure can't collide. A hard-coded scratch slot in global memory writes to a fixed absolute address with no per-procedure isolation, so it risks colliding with another procedure using the same address, forces every caller to avoid clobbering it, and locks the layout.

A procedure with no `@locals` declaration allocates zero locals, so any `loc_store` / `loc_load` in it fails to assemble — always declare the locals you use.

## Examples

```masm
# Good
@locals(2)
proc compute_hash
    # write the two scratch values into local slots 0 and 1
    loc_store.0
    loc_store.1
    # ...
    loc_load.0
    loc_load.1
end

# Bad: scratch in a shared global region
const SCRATCH_PTR = 0x4000
proc compute_hash
    mem_store.SCRATCH_PTR        # collides with anyone else using SCRATCH_PTR
end
```
