---
name: masm-file-structure
description: Enforce file structure, module-tree declaration, and section ordering for Miden Assembly (.masm) files. Use when editing, reviewing, or creating .masm files.
---

# MASM File Structure

MASM files follow a consistent top-level section order. Use section headers with the long separator line:

```masm
# SECTION NAME
# =================================================================================================
```

## Declare every module in its parent — an undeclared `.masm` file is silently dropped

Modules are discovered by **declaration**, not by scanning the directory. The assembler parses the root module, reads its `mod` / `pub mod` declarations, and walks only that tree. A `.masm` file that no parent module declares is never parsed, never assembled, and produces **no error and no warning** — the procedures inside it simply do not exist, and every call site fails later with a confusing "undefined procedure".

**Adding a new `.masm` file is two steps, not one.** Create the file, then add its `mod` line to the parent.

```masm
# standards/notes/mod.masm — one line per sibling .masm file
pub mod burn
pub mod mint
pub mod p2id
pub mod swap
```

Use `pub mod` when the submodule is reachable from outside the parent, and plain `mod` when it is internal to the parent module.

Two directory layouts are accepted for a declared submodule `foo`:

- **Sibling file** — `<parent_dir>/foo.masm`
- **Directory with `mod.masm`** — `<parent_dir>/foo/mod.masm`

Pick one per module; the parent's `mod foo` line is identical either way. A submodule may not share the name of its own parent module.

## Annotate every procedure that is part of the account interface

A `pub proc` in a component module is **not** automatically an account procedure. The component's interface is the set of exports carrying `@account_procedure` or `@auth_script`; every other `pub proc` is still exported by the package but is not callable as an account procedure — from a transaction script, a note, FPI, or a sibling component. This compiles cleanly and fails only at the call site, so it is easy to miss.

The attributes that appear in protocol source, with their counts at the pinned release:

| Attribute | Use |
|---|---|
| `@account_procedure` | Marks a `pub proc` as part of the account interface. The common case. |
| `@auth_script` | Marks the single authentication procedure of an authentication component. Mutually exclusive with `@account_procedure` — they belong to different component kinds. |
| `@note_script` | Marks a note script entry point (`pub proc main`). |
| `@transaction_script` | Marks a transaction script entry point. |
| `@locals(N)` | Declares the procedure's local-memory frame size (see masm-locals-over-globals). |

Attributes sit on their own lines immediately above `pub proc`, after the `#!` doc comment:

```masm
#! Returns the maximum supply.
#!
#! Inputs:  [pad(16)]
#! Outputs: [max_supply, pad(15)]
#!
#! Invocation: call
@account_procedure
pub proc get_max_supply
    ...
end
```

A procedure may carry more than one attribute. Source orders `@locals(N)` first and the interface attribute directly above `pub proc`:

```masm
@locals(2)
@account_procedure
pub proc approve
    ...
end
```

## Section Order

1. **Module declarations** – `mod` / `pub mod` lines (in a `mod.masm` or a root module); no section header
2. **Imports** – `use` statements only; no section header
3. **Type aliases** – `pub type` / `type` definitions (use `pub type` when referenced cross-module), and `enum` declarations for closed sets of related literals (see masm-constants)
4. **Constants** – see the masm-constants skill for organization (errors and non-error constants are grouped into separate subsections; source uses both orderings)
5. **Events** – `const ... = event(...)` definitions; appears in kernel/event-emitting modules, after Constants (or after imports when no constants section) and before the procedure sections
6. **Procedures** – the procedure region. Two main forms ship in source (plus a rare variant noted below):
   - A single **`# PROCEDURES`** section holding all procedures when a module does not separate API from internals. This is a common form in source (about as many files use a lone `# PROCEDURES` as use the split below), e.g. `standards/notes/p2id.masm`, `standards/data_structures/double_word_array.masm`, and `standards/data_structures/array.masm`. (Kernel `account.masm` is *not* this form: it uses `# PROCEDURES` followed by a separate `# HELPER PROCEDURES` section.)
   - A **`# PUBLIC INTERFACE`** section (`pub proc` that form the module API) followed by a **`# HELPER PROCEDURES`** section (procedures used internally, mostly non-`pub`)—used only when a module explicitly separates its API from its internals, as the agglayer `bridge_in`, `bridge_config`, and `bridge_out` modules do. Note that `pub proc` may still appear under `# HELPER PROCEDURES` when a helper is re-exported or unit-tested.

## Example Structure

The module, constant, type, and procedure names below are illustrative (not a copy of any one source file); the section layout, type shapes, and import namespaces match source.

```masm
use example::leaf::config
use example::leaf::utils
use miden::core::mem
use miden::core::word
use {DoubleWord} from miden::protocol::types

# TYPE ALIASES
# =================================================================================================

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

A module that does not separate its API from its internals collapses the two procedure sections into a single `# PROCEDURES` header instead—e.g. `standards/notes/p2id.masm`, whose `# PROCEDURES` section holds `pub proc main` (the `@note_script` entry point), `pub proc prepare_note` and `pub proc create_output_note` with no public/helper split.

> The order of the `# CONSTANTS` and `# ERRORS` subsections is **not fixed** in source: some files place errors first (e.g. the agglayer `bridge_in`, `bridge_config`, and `bridge_out` modules, and `standards/notes/p2id.masm` and `standards/notes/mint.masm`), others place non-error constants first (e.g. `standards/access/authority.masm`, `standards/access/rbac.masm`, `standards/auth/guardian.masm`, `standards/auth/signature.masm`). Either order is acceptable—just keep the two grouped separately rather than interleaved. Some kernel modules also spell the header `# CONSTS` (e.g. kernel `prologue.masm`).

> Where a module declares type aliases, they come before its constants — `protocol/src/types.masm` is the canonical type-alias module and opens with `# TYPE ALIASES`. Most modules declare none at all and open with errors or constants instead (kernel `api.masm` opens with `# NOTE` then `# ERRORS`; kernel `memory.masm` opens with `# ERRORS`), so "type-first" describes the ordering *within* a module that has both, not a rule that every module leads with types.

> Types referenced cross-module must be marked `pub type` (`pub` is required for any constant, procedure, or type referenced from another module). Every type alias in protocol source at the pinned release is `pub type`; a bare non-`pub` `type` does not appear anywhere, so treat `pub type` as the default and reach for a private `type` only for a genuinely module-local alias. The `# EVENTS` subsection holds `const NAME = event("...")` definitions and appears in kernel/event-emitting modules. Where a constants section is present it sits after constants and before the procedure sections (e.g. the kernel `account`, `output_note`, and `link_map` modules); in modules with no constants section it follows the imports directly (e.g. the kernel `bin/main.masm` module and `protocol/src/auth.masm`, where `# EVENTS` is the first content section).

## Guidelines

- **Module declarations**: Every `.masm` file must be declared by its parent with `mod` / `pub mod`, or it is silently excluded from the build. One declaration per line, no section header.
- **Imports**: One `use` per line; group by module/namespace. A blank line MAY separate import groups by namespace (as the kernel `bin/main.masm` module does), but keeping the whole block unseparated is equally common and dominant in source (e.g. the agglayer modules); the Example Structure above follows the unseparated form. Do not put a blank line between `use` statements within the same group. Use the bare top-level namespace, e.g. `use agglayer::bridge::bridge_config` and `use miden::core::word`—there is no `miden::agglayer::` prefix. A selective import names the items in braces: `use {DoubleWord} from miden::protocol::types`. Imports normally form a single block at the top; a few source files (e.g. `agglayer/bridge/bridge_out.masm`) interleave a stray `use` near the related constants—prefer the single top block, but this is not treated as an error.
- **Type aliases**: Define shared types (e.g. `DoubleWord`, `MemoryAddress`) before the module's constants. Most modules declare no type aliases at all and open with errors or constants — that is normal, not a violation. Mark a type `pub type` when it is referenced from another module; every type alias in protocol source is `pub type`, so make that the default.
- **Constants**: Defer to the masm-constants skill for *organization*—group non-error constants by topic and keep `ERR_*` in a dedicated errors subsection. The *order* of the errors subsection relative to the non-error constants is not fixed in source; it may come before or after, so don't treat either order as a hard rule. Just don't interleave them.
- **Events**: Kernel/event-emitting modules collect their `const NAME = event("...")` definitions in an `# EVENTS` subsection. It sits after the constants section when one is present, otherwise directly after the imports, and always before the procedure sections.
- **Procedures**: When a module does not separate its API from its internals, put every procedure under a single `# PROCEDURES` section (a common form in source, e.g. `standards/notes/p2id.masm`, `standards/data_structures/double_word_array.masm`, `standards/data_structures/array.masm`). Split into `# PUBLIC INTERFACE` and `# HELPER PROCEDURES` only when the module deliberately separates API from internals. (A module may also keep a `# PROCEDURES` header for its main procedures and add a separate `# HELPER PROCEDURES` section, as kernel `account.masm` does.)
- **Public interface**: Only `pub proc`; these are the module’s API. Order by importance or call flow.
- **Helper procedures**: Procedures that support the public interface, mostly non-`pub`. May include `pub proc` helpers (e.g. a `get_leaf_value` that is used internally, re-exported, or exercised in unit tests).

## When Sections Are Omitted

- No imports → start with type aliases or constants
- No type aliases → constants follow imports
- No events → procedure sections follow constants directly (most non-kernel modules have no `# EVENTS` section)
- No public/helper split → use a single `# PROCEDURES` section for all procedures
- No helpers → public interface is the last section

## Validation Checklist

- [ ] The file is declared in its parent module with `mod` / `pub mod` (`pub mod` if reachable from outside)
- [ ] Imports at top (if any), using the bare namespace (no `miden::agglayer::` prefix)
- [ ] Type aliases (if any) before constants and procedures; a module with no type aliases opens with errors or constants
- [ ] Cross-module types marked `pub type`
- [ ] Constants before procedures; errors grouped in their own subsection, separate from non-error constants (either order)
- [ ] Events (if any) grouped in an `# EVENTS` subsection after constants (or after imports if no constants section) and before procedures
- [ ] Procedures under a single `# PROCEDURES` section, OR split into `# PUBLIC INTERFACE` (`pub proc`) before `# HELPER PROCEDURES` when the module separates API from internals
- [ ] Section headers use `# SECTION NAME` and `# ===...` separator
