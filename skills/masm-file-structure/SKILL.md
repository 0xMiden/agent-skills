---
name: masm-file-structure
description: Enforce file structure and section ordering for Miden Assembly (.masm) files. Use when editing, reviewing, or creating .masm files.
---

# MASM File Structure

MASM files follow a consistent top-level section order. Use section headers with the long separator line:

```masm
# SECTION NAME
# =================================================================================================
```

## Section Order

1. **Imports** – `use` statements only; no section header
2. **Type aliases** – `type` definitions
3. **Constants** – see the masm-constants skill for organization (errors and non-error constants are grouped into separate subsections; source uses both orderings)
4. **Public interface** – `pub proc` procedures that form the module API
5. **Helper procedures** – `proc` (non-pub) procedures used internally

## Example Structure

```masm
use miden::agglayer::bridge::bridge_config
use miden::agglayer::bridge::leaf_utils
use miden::core::mem
use miden::core::word

# TYPE ALIASES
# =================================================================================================

type BeWord = struct @bigendian { a: felt, b: felt, c: felt, d: felt }
type DoubleWord = struct { word_lo: BeWord, word_hi: BeWord }
type MemoryAddress = u32

# CONSTANTS
# =================================================================================================

const PROOF_DATA_PTR = 0
const PROOF_DATA_WORD_LEN = 134

# ERRORS
# =================================================================================================

const ERR_BRIDGE_NOT_MAINNET = "bridge not mainnet"
const ERR_LEADING_BITS_NON_ZERO = "leading bits of global index must be zero"

# PUBLIC INTERFACE
# =================================================================================================

#! Main entry point. Computes the leaf value and verifies it.
#!
#! Inputs:  [LEAF_DATA_KEY, PROOF_DATA_KEY, pad(8)]
#! Outputs: [pad(16)]
#!
#! Invocation: call
pub proc verify_leaf_bridge
    exec.get_leaf_value
    exec.verify_leaf
end

# HELPER PROCEDURES
# =================================================================================================

#! Loads leaf data and computes the leaf value.
#!
#! Inputs:  [LEAF_DATA_KEY]
#! Outputs: [LEAF_VALUE[8]]
#!
#! Invocation: exec
proc get_leaf_value(leaf_data_key: BeWord) -> DoubleWord
    ...
end

#! Verifies leaf against Merkle proof.
#!
#! Inputs:  [LEAF_VALUE[8], PROOF_DATA_KEY]
#! Outputs: []
#!
#! Invocation: exec
proc verify_leaf
    ...
end
```

> The order of the `# CONSTANTS` and `# ERRORS` subsections is **not fixed** in the v0.15 source: some files place errors first (e.g. the agglayer `bridge_config` and `eth_address` modules), others place non-error constants first (e.g. many `miden-standards` components such as `signature`, `guardian`, `multisig`). Both orders ship even within the same directory (`standards/notes/mint.masm` is constants-first while `standards/notes/p2id.masm` is errors-first). Either order is acceptable—just keep the two grouped separately rather than interleaved.

## Guidelines

- **Imports**: One `use` per line; group by module. No blank lines between imports.
- **Type aliases**: Define shared types (e.g. `DoubleWord`, `MemoryAddress`) before constants or procedures.
- **Constants**: Defer to the masm-constants skill for *organization*—group non-error constants by topic and keep `ERR_*` in a dedicated errors subsection. The *order* of the errors subsection relative to the non-error constants is not fixed in source; it may come before or after, so don't treat either order as a hard rule. Just don't interleave them.
- **Public interface**: Only `pub proc`; these are the module’s API. Order by importance or call flow.
- **Helper procedures**: Non-pub procedures that support the public interface. May include `pub proc` helpers (e.g. `get_leaf_value`) if they are used internally, re-exported, or used for unit tests.

## When Sections Are Omitted

- No imports → start with type aliases or constants
- No type aliases → constants follow imports
- No helpers → public interface is the last section

## Validation Checklist

- [ ] Imports at top (if any)
- [ ] Type aliases before constants and procedures
- [ ] Constants before procedures; errors grouped in their own subsection, separate from non-error constants (either order)
- [ ] Public interface (`pub proc`) before helper procedures
- [ ] Section headers use `# SECTION NAME` and `# ===...` separator
