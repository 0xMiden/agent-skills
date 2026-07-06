---
name: decouple-component-from-storage
description: Use when writing a GENERIC MASM storage utility (one that operates over a caller-chosen slot or map) inside a reusable account component — receive the slot id as a parameter so the utility works against any slot, not one hard-coded one.
---

# Decouple Generic Storage Utilities from a Hard-Coded Slot

## Rule

A *generic* storage utility — a procedure meant to operate over **any** caller-chosen storage slot or map — must not bake one specific slot id into its body. Take the slot as a parameter — the slot id split into its `slot_id_suffix` / `slot_id_prefix` felts — and pass it into the storage-access procedure (`active_account::get_item` / `get_map_item`, `native_account::set_item` / `set_map_item`). The caller that knows which slot to operate on supplies the slot id.

This is the pattern the standard `array` / `double_word_array` data-structure utilities use: `get(slot_id_suffix, slot_id_prefix, index)` takes the slot id as input so one utility can drive many different maps.

This rule does **not** apply to a component referencing its **own dedicated named slot**. That is already correct and portable — see "When NOT to parameterize" below.

## Why

A generic helper that hard-codes one slot id can only ever serve that one slot — it cannot be reused over a different map even though its logic is identical. Worse, if the caller wants it to act on a slot that is not the hard-coded one, there is no silent misread to detect: `active_account::get_item` / `get_map_item` **panic** if the supplied slot id does not exist in account storage, so a mismatch faults loudly rather than returning a wrong value. Taking the slot id as a parameter makes the utility reusable and lets the caller point it at whichever slot it owns.

## When NOT to parameterize

A storage slot id in v0.15 is **name-derived, not positional**: the id is the first two felts of the hash of the slot's name (`word("project::component::slot")`), so the **same slot name produces the same slot id in every account that installs the component**. A component that reads/writes its **own** dedicated named slot is therefore portable by construction — there is nothing to parameterize, because the name fixes the id everywhere.

The canonical, correct idiom for a component's own slot is a named-`word` constant plus a `[0..2]` slice — do not "fix" this to take a slot parameter:

```masm
# Correct and portable: the component owns this named slot; the name hashes
# to the same slot id in every account, so no parameter is needed.
pub const AUTHORITY_SLOT = word("miden::standards::access::authority")

pub proc assert_authorized
    push.AUTHORITY_SLOT[0..2] exec.active_account::get_item
    # => [authority, role_symbol, 0, 0]
    # ...
end
```

This is exactly what the standard `authority`, `ownable2step`, `rbac`, `multisig`, `pausable`, and faucet components do for their own slots. `assert_authorized` has a fixed `[] -> []` signature and is invoked with no slot argument by many consumers — parameterizing it would break every call site.

## Examples

```masm
# Good: a GENERIC array utility takes the slot id and uses it for the storage
# access, so the same code drives any map the caller owns (cf. standard array.masm).
pub proc get(slot_id_suffix: felt, slot_id_prefix: felt, index: felt) -> word
    movup.2 push.0.0.0        # build KEY = [0, 0, 0, index]
    movup.5 movup.5           # => [slot_id_suffix, slot_id_prefix, KEY]
    exec.active_account::get_map_item
    # => [VALUE]
end

# The caller, which knows which map to operate on, passes the slot id in:
push.index push.MY_MAP_SLOT_ID_PREFIX push.MY_MAP_SLOT_ID_SUFFIX
exec.get

# Bad: a generic array getter hard-codes ONE specific map's slot id, so it can
# only ever serve that single map even though the logic is reusable.
pub proc get_at(index: felt) -> word
    push.0.0.0                       # build KEY = [0, 0, 0, index]
    push.SOME_FIXED_MAP_SLOT[0..2]   # baked-in slot id — not a parameter
    # => [slot_id_suffix, slot_id_prefix, KEY]
    exec.active_account::get_map_item
    # => [VALUE]
end
```
