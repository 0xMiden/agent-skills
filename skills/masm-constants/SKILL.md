---
name: masm-constants
description: Enforce constant definition and organization conventions for Miden Assembly (.masm) files. Use when editing, reviewing, or creating .masm files that define or use constants.
---

# MASM Constant Conventions

## Placement

Constants must be defined at the **top of the file**, before any procedure definitions, but after imports or type aliases.

## Organization

### Constants Section (first)

Group non-error constants by topic. Use blank lines between sections. Common topics:

- **Slot names** – use `word("path::to::slot")` for slot identifiers
- **Memory pointer offsets** – offsets into memory regions
- **Local memory offsets** – offsets within a procedure's local memory
- **Magic numbers / values** – domain-specific literals
- **Event identifiers** – `event("...")` constants consumed by `emit` (e.g. `pub const AUTH_REQUEST_EVENT = event("miden::protocol::auth::request")` used as `emit.AUTH_REQUEST_EVENT`)

Order sections by dependency or by usage frequency. Put the most widely used first.

There is no `trace` decorator and no numeric trace-id constant: print-style debugging goes through the `miden::core::debug` procedures (`print_stack`, `print_mem`, `print_mem_addr`, `print_mem_all`, `print_adv_stack`, `print_adv_stack_all`, `print_adv_map_all`, `print_adv_map_item`), which are ordinary `emit`-based procedures. They print **whenever invoked** — there is no debug-mode toggle gating them — so remove them before merging.

### Errors Section (after constants)

Errors (panic/assert error codes, e.g. `ERR_*`) go in a **dedicated "errors" section**, placed after the constants section. Define them with string values describing the error. (Some files place the errors section before the constants section; constants-first is the more common order and is preferred for new files.)

## Naming

### Memory Pointers

Memory pointer constants describe offsets into memory regions that are shared or global (e.g. layout of a memory region, input/output structure offsets). They are **not** scoped to a single procedure.

```masm
# Good: descriptive, shared usage
const ASSET_OFF = 0
const AMOUNT_OFF = 1
const NOTE_DATA_LEN = 16

# Or grouped by memory region if applicable
const INPUT_PTR_OFF = 0
const INPUT_LEN_OFF = 1
```

### Local Memory Offsets

Local memory offsets describe offsets within a procedure's local memory. Name them with **UPPERCASE descriptive names ending in `_LOC` (or `_LOCAL`)**:

```masm
# Good: descriptive UPPERCASE name ending in _LOC
const IS_SIGNER_FOUND_LOC = 0
const CURRENT_SIGNER_INDEX_LOC = 1
```

When several procedures in the same file define local offsets, **group each procedure's locals under a section comment** and, where names would otherwise collide or be ambiguous, disambiguate with an UPPERCASE context prefix derived from the procedure name. Not every constant in a group needs the prefix — use it only where it adds clarity:

```masm
# bridge_out memory locals
const BRIDGE_OUT_BURN_ASSET_LOC = 0
const DESTINATION_NETWORK_LOC   = 13
const BRIDGE_OUT_IS_NATIVE_LOC  = 14

# create_burn_note memory locals
const CREATE_BURN_NOTE_BURN_ASSET_LOC = 0
const ATTACHMENT_LOC                  = 8
```

Do **not** use a lowercase procedure-name prefix (e.g. `validate_note_NOTE_IDX_LOC`); that form does not appear in the protocol source. Every constant name in protocol source begins with an uppercase letter.

## Related literals belong in an `enum`, not in loose constants

A closed set of related values (a discriminator, a mode, a kind) is declared as an `enum` in the type-aliases section rather than as a run of `const` definitions. The variants are then referenced by name like any other constant:

```masm
# Authority kind stored in the first element of `AUTHORITY_SLOT`, dispatched on by
# `assert_authorized`. Mirrors the Rust `Authority` enum.
enum Authority : u8 {
    AUTH_CONTROLLED = 0,
    OWNER_CONTROLLED = 1,
    RBAC_CONTROLLED = 2,
}
```

An `enum` names its backing integer type after a colon and can be marked `pub` when other modules reference its variants. Reach for it whenever you would otherwise write three or four constants that are only meaningful relative to one another; keep plain `const` for standalone values.

## Formatting

Put **spaces around the equals sign**. This is the dominant style throughout protocol source: `miden-protocol` uses spaces exclusively, and `miden-standards` and `miden-agglayer` use them for the large majority of their definitions. A handful of older files still use the no-space form (`const IS_SIGNER_FOUND_LOC=0`) — do **not** treat those as errors when editing them, and keep a file internally consistent with its surrounding style.

**Errors** – string value describing the error:
```masm
const ERR_BRIDGE_NOT_MAINNET = "mainnet flag must be 1 for a mainnet deposit"
const ERR_UNAUTHORIZED = "unauthorized"
```

**Slots** – use `word()` with the slot path; mark them `pub const` when the slot is part of a component's public storage layout:
```masm
pub const THRESHOLD_CONFIG_SLOT = word("miden::standards::auth::multisig::threshold_config")
pub const AUTHORITY_SLOT = word("miden::standards::access::authority::authority_config")
```

**Offsets / numeric values** – plain numbers:
```masm
const IS_SIGNER_FOUND_LOC = 0
const CURRENT_SIGNER_INDEX_LOC = 1
```

## Visibility (`pub const`)

A constant must be declared `pub const` to be referenced from another module. This is **not** limited to storage slots: any constant imported elsewhere via `use {CONST} from path::to::module` (or referenced as `module::CONST`) must be `pub`, including magic numbers, memory-pointer offsets, and event identifiers. Keep purely file-local constants as plain `const`.

```masm
# pub because they are imported by other modules
pub const MAX_ASSETS_PER_NOTE = 16
pub const AUTH_REQUEST_EVENT = event("miden::protocol::auth::request")

# file-local only — no pub needed
const IS_SIGNER_FOUND_LOC = 0
```

## Example Layout

```masm
# TYPE ALIASES
# =================================================================================================

enum Authority : u8 {
    AUTH_CONTROLLED = 0,
    OWNER_CONTROLLED = 1,
    RBAC_CONTROLLED = 2,
}

# CONSTANTS
# =================================================================================================

# Storage slots
pub const AUTHORITY_SLOT = word("miden::standards::access::authority::authority_config")

# Memory pointers
const ASSET_OFF = 0
const AMOUNT_OFF = 1

# bridge_out memory locals
const BRIDGE_OUT_BURN_ASSET_LOC = 0
const DESTINATION_NETWORK_LOC   = 13
const BRIDGE_OUT_IS_NATIVE_LOC  = 14

# create_burn_note memory locals
const CREATE_BURN_NOTE_BURN_ASSET_LOC = 0
const ATTACHMENT_LOC                  = 8

# ERRORS
# =================================================================================================

const ERR_BRIDGE_NOT_MAINNET = "mainnet flag must be 1 for a mainnet deposit"
const ERR_UNAUTHORIZED = "unauthorized"
const ERR_NOTE_NOT_FOUND = "note not found"

# PROCEDURES
# =================================================================================================

pub proc validate_note
    ...
end
```

## Validation Checklist

- [ ] All constants defined at top of file (after imports / type aliases)
- [ ] Errors in dedicated section, placed after the constants section (constants-first preferred)
- [ ] Non-error constants grouped by topic with blank lines between sections
- [ ] A closed set of related literals is declared as an `enum` in the type-aliases section, not as loose constants
- [ ] Memory pointers (shared/global) use descriptive names
- [ ] Local memory offsets use UPPERCASE names ending in `_LOC`/`_LOCAL`, grouped under a per-procedure section comment when multiple procedures define locals
- [ ] No lowercase procedure-name prefix on local offset constants
- [ ] Constants referenced from another module are declared `pub const`; file-local constants stay plain `const`
- [ ] Spaces around `=` for new/edited constants (a few older files are no-space; keep file consistent)
- [ ] No `trace` decorator or numeric trace id; print-style debugging uses `miden::core::debug` and is removed before merge
- [ ] Section comments used to label topic groups
