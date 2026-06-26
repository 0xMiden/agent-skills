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
2. **Type aliases** – `pub type` / `type` definitions (use `pub type` when referenced cross-module)
3. **Constants** – see the masm-constants skill for organization (errors and non-error constants are grouped into separate subsections; source uses both orderings)
4. **Events** – `const ... = event(...)` definitions; appears in kernel/event-emitting modules, after Constants (or after imports when no constants section) and before the procedure sections
5. **Procedures** – the procedure region. Two forms ship in v0.15 source:
   - A single **`# PROCEDURES`** section holding all procedures when a module does not separate API from internals. This is the most common form in source (more files use `# PROCEDURES` than the split below), e.g. `standards/notes/p2id.masm`, `standards/data_structures/double_word_array.masm`, and kernel `account.masm`.
   - A **`# PUBLIC INTERFACE`** section (`pub proc` that form the module API) followed by a **`# HELPER PROCEDURES`** section (procedures used internally, mostly non-`pub`)—used only when a module explicitly separates its API from its internals. Note that `pub proc` may still appear under `# HELPER PROCEDURES` when a helper is re-exported or unit-tested.

## Example Structure

The module, constant, type, and procedure names below are illustrative (not a copy of any one source file); the section layout, type shapes, and import namespaces match v0.15 source.

```masm
use example::leaf::config
use example::leaf::utils
use miden::core::mem
use miden::core::word

# TYPE ALIASES
# =================================================================================================

pub type DoubleWord = struct { word_lo: word, word_hi: word }
pub type MemoryAddress = u32

# CONSTANTS
# =================================================================================================

const PROOF_DATA_PTR = 0
const CLAIM_PROOF_DATA_WORD_LEN = 134

# ERRORS
# =================================================================================================

const ERR_BRIDGE_NOT_MAINNET = "mainnet flag must be 1 for a mainnet deposit"
const ERR_LEADING_BITS_NON_ZERO = "leading bits of global index must be zero"

# PUBLIC INTERFACE
# =================================================================================================

#! Main entry point. Computes the leaf value and verifies it.
#!
#! Inputs:  [LEAF_DATA_KEY, PROOF_DATA_KEY, pad(8)]
#! Outputs: [pad(16)]
#!
#! Invocation: call
pub proc verify_leaf_root
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
proc get_leaf_value(leaf_data_key: word) -> DoubleWord
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

A module that does not separate its API from its internals collapses the two procedure sections into a single `# PROCEDURES` header instead—e.g. `standards/notes/p2id.masm`, whose `# PROCEDURES` section holds both `pub proc main` and `pub proc new` with no public/helper split.

> The order of the `# CONSTANTS` and `# ERRORS` subsections is **not fixed** in the v0.15 source: some files place errors first (e.g. the agglayer `bridge_in`, `bridge_config`, and `eth_address` modules), others place non-error constants first (e.g. many `miden-standards` components such as `signature`, `guardian`, `multisig`). Both orders ship even within the same directory (`standards/notes/mint.masm` is constants-first while `standards/notes/p2id.masm` is errors-first). Either order is acceptable—just keep the two grouped separately rather than interleaved.

> Type aliases are conventionally defined before constants, and v0.15 source is overwhelmingly type-first (e.g. `eth_address.masm`, `types.masm`, kernel `api.masm`/`memory.masm`). This is a convention, not an absolute: a component may place local-memory constants before a type alias—e.g. `standards/data_structures/double_word_array.masm` defines its `SLOT_ID_PREFIX_LOC` / `SLOT_ID_SUFFIX_LOC` / `INDEX_LOC` constants ahead of its in-module `type DoubleWord`. Prefer type-first; don't treat const-before-type as an error.

> Types referenced cross-module must be marked `pub type` (v0.15 requires `pub` for any constant, procedure, or type referenced from another module). Every cross-module type alias in v0.15 source is `pub type`; only the in-module `type DoubleWord` in `double_word_array.masm` is non-`pub`. The `# EVENTS` subsection holds `const NAME = event("...")` definitions and appears in kernel/event-emitting modules. Where a constants section is present it sits after constants and before the procedure sections (e.g. the kernel `account`, `output_note`, `link_map`, and `prologue` modules); in modules with no constants section it follows the imports directly (e.g. the kernel `main` module and `protocol/auth.masm`, where `# EVENTS` is the first content section).

## Guidelines

- **Imports**: One `use` per line; group by module/namespace. A blank line MAY separate import groups by namespace (as the kernel `main`/`api`/`note`/`prologue`/`epilogue` modules do), but keeping the whole block unseparated is equally common and dominant in source (e.g. the agglayer modules); the Example Structure above follows the unseparated form. Do not put a blank line between `use` statements within the same group. Use the bare top-level namespace, e.g. `use agglayer::bridge::bridge_config` and `use miden::core::word`—there is no `miden::agglayer::` prefix. Imports normally form a single block at the top; a few source files (e.g. `standards/notes/mint.masm`, `agglayer/bridge/bridge_out.masm`) interleave a stray `use` near the related constants—prefer the single top block, but this is not treated as an error.
- **Type aliases**: Define shared types (e.g. `DoubleWord`, `MemoryAddress`) conventionally before constants. v0.15 source is overwhelmingly type-first, though a component may place local-memory constants ahead of a type alias (e.g. `double_word_array.masm`)—so prefer type-first but don't treat const-before-type as a violation. Mark a type `pub type` when it is referenced from another module (v0.15 requires `pub` for cross-module references); a non-`pub` `type` used only within its own module is valid.
- **Constants**: Defer to the masm-constants skill for *organization*—group non-error constants by topic and keep `ERR_*` in a dedicated errors subsection. The *order* of the errors subsection relative to the non-error constants is not fixed in source; it may come before or after, so don't treat either order as a hard rule. Just don't interleave them.
- **Events**: Kernel/event-emitting modules collect their `const NAME = event("...")` definitions in an `# EVENTS` subsection. It sits after the constants section when one is present, otherwise directly after the imports, and always before the procedure sections.
- **Procedures**: When a module does not separate its API from its internals, put every procedure under a single `# PROCEDURES` section (the most common form in v0.15 source, e.g. `standards/notes/p2id.masm`, `standards/data_structures/double_word_array.masm`, kernel `account.masm`). Split into `# PUBLIC INTERFACE` and `# HELPER PROCEDURES` only when the module deliberately separates API from internals.
- **Public interface**: Only `pub proc`; these are the module’s API. Order by importance or call flow.
- **Helper procedures**: Procedures that support the public interface, mostly non-`pub`. May include `pub proc` helpers (e.g. a `get_leaf_value` that is used internally, re-exported, or exercised in unit tests).

## When Sections Are Omitted

- No imports → start with type aliases or constants
- No type aliases → constants follow imports
- No events → procedure sections follow constants directly (most non-kernel modules have no `# EVENTS` section)
- No public/helper split → use a single `# PROCEDURES` section for all procedures
- No helpers → public interface is the last section

## Validation Checklist

- [ ] Imports at top (if any), using the bare namespace (no `miden::agglayer::` prefix)
- [ ] Type aliases before procedures; conventionally before constants (type-first is the v0.15 norm, but const-before-type is acceptable in some component files)
- [ ] Cross-module types marked `pub type`
- [ ] Constants before procedures; errors grouped in their own subsection, separate from non-error constants (either order)
- [ ] Events (if any) grouped in an `# EVENTS` subsection after constants (or after imports if no constants section) and before procedures
- [ ] Procedures under a single `# PROCEDURES` section, OR split into `# PUBLIC INTERFACE` (`pub proc`) before `# HELPER PROCEDURES` when the module separates API from internals
- [ ] Section headers use `# SECTION NAME` and `# ===...` separator
