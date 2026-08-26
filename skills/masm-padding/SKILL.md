---
name: masm-padding
description: Enforce stack padding conventions for Miden Assembly (.masm) procedures based on invocation type. Use when editing, reviewing, or creating .masm procedures, especially those with Invocation annotations.
---

# MASM Padding Conventions

## Overview

Miden assembly has five invocation instructions, which fall into two stack-discipline classes:

| Class | Instructions | Stack behavior | Padding convention |
|-------|--------------|----------------|--------------------|
| **Same context** (inline) | `exec`, `dynexec` | Share the caller's operand stack; no depth-16 boundary | Follows the caller — no required pad-to-16 |
| **Context switch** | `call`, `dyncall`, `syscall` | New/return context; on entry the visible stack is set to depth 16, and on return the depth must be *exactly* 16 or the VM traps | Inputs/Outputs documented to 16 with `pad(N)` (prevailing convention) |

The difference between `dynexec` and `dyncall` is the same as the difference between `exec` and `call`: the `dyn` variants only differ from the static ones in that the call target is resolved dynamically from the stack at runtime. `dynexec` does NOT context-switch — it inlines into the caller's context just like `exec`.

`syscall` is the same as `call` mechanically (it context-switches and truncates the visible stack to 16), except the target must be a procedure in the program's kernel and execution returns to the root context.

## Stack Depth Floor: 16

Miden VM enforces a minimum operand-stack depth of 16 elements (`MIN_STACK_DEPTH = 16` in the VM core). When an operation would naively shrink the stack below 16, the VM auto-fills the missing positions with zeros via the overflow-table mechanism. The actual depth stays exactly 16; only the visible content shrinks.

This invariant applies at the entry/return boundary of the **context-switching** invocations:

- `call`, `dyncall`, and `syscall` procedures (the visible stack is set to 16 on entry, and must be exactly 16 on return or the VM traps),
- note scripts and transaction scripts (entered via `dyncall` at depth 16).

> Note: for `dyncall`, the stack is shifted left by one element (the consumed MAST-root pointer) before being set to 16.

It does NOT apply to the **same-context** invocations `exec` and `dynexec`, which share the caller's stack and can drop the visible count below 16 by consuming caller elements (see Danger Zone).

### Tracking the floor in inline comments

When the naive math would put the stack below 16, the `# =>` tracker must reflect the actual auto-padded depth, not the naive count.

```masm
# entry at depth 16: [VALUE, pad(12)]
dropw
# => [pad(16)]   # correct: VALUE replaced with zeros, depth still 16
```

Not:

```masm
# => [pad(12)]   # wrong: depth is still 16, only the visible content shrank
```

This shows up most often at the start of note scripts that don't use their input arguments:

```masm
@note_script
pub proc main
    dropw
    # => [pad(16)]
    ...
end
```

Standard note scripts are MASM library procs annotated with `@note_script` (`pub proc main … end`), not bare `begin … end` programs. Bare `begin … end` is still valid MASM, but only at the top-level kernel/tx-script program entry.

## Context-Switching Procedures (`call`, `dyncall`, `syscall`)

The VM hard-requires the visible stack depth to be exactly 16 when a context-switching procedure returns; otherwise the VM traps. To make this explicit, the prevailing convention is to document Inputs/Outputs padded to 16 with `pad(N)`:
1. **Doc comments** (`#!`) for Inputs/Outputs
2. **Inline comments** (`#`) showing stack state

### Doc Comment Format

Use `pad(N)` notation where N + other elements = 16:

```masm
#! Inputs:  [ASSET_ID, ASSET_VALUE, pad(8)]
#! Outputs: [pad(16)]
#!
#! Invocation: call
pub proc receive_asset
```

This is a strong convention, not an absolute documentation rule. Some standard `call` procedures document only their *logical* stack (fewer than 16 elements, no `pad`) — e.g. `standards/auth/multisig.masm`'s `set_procedure_threshold` documents `Inputs: [proc_threshold, PROC_ROOT]` / `Outputs: []`, and `standards/auth/guardian.masm`'s `update_guardian_public_key` documents `Inputs: [new_guardian_scheme_id, NEW_GUARDIAN_PUBLIC_KEY]` / `Outputs: []`. Prefer `pad(N)`-to-16 for new code, but do not flag logical-only documentation on an existing `call`/`dyncall`/`syscall` proc as a bug.

Note that padded-to-16 is the majority even for accessors: the multisig `get_threshold_and_num_approvers` documents `Inputs: [pad(16)]` / `Outputs: [default_threshold, num_approvers, pad(14)]`.

### Inline Comment Format

Track padding through the procedure:

```masm
exec.native_account::add_asset
# => [ASSET_VALUE', pad(12)]

# drop the final asset
dropw
# => [pad(16)]
```

## Same-Context Procedures (`exec`, `dynexec`)

Procedures invoked with `exec` or `dynexec` share the caller's stack directly, so their padding follows whatever the caller establishes — there is no depth-16 boundary to satisfy.

Most `exec`/`dynexec` procedures document only their logical inputs/outputs, without explicit padding:

```masm
#! Inputs:  [PK_COMM, scheme_id]
#! Outputs: []
#!
#! Invocation: exec
pub proc authenticate_transaction
```

The same holds for `dynexec`, whose `Invocation:` value dominates the kernel API module (`kernels/transaction/lib/api.masm`).

The exception is when the caller itself works in a padded-to-16 context: then the `exec`/`dynexec` proc documents `pad(N)`-to-16 to match the caller. The kernel transaction API does this — its `dynexec` procedures pad to 16 (e.g. `account_get_initial_commitment` documents `Inputs: [pad(16)]` / `Outputs: [INIT_COMMITMENT, pad(12)]`) because the surrounding caller convention is padded. Padding for same-context procedures is dictated by the caller, not by the invocation instruction.

### Why Padding Follows the Caller

`exec` / `dynexec` procedures share the caller's stack directly. Imposing a fixed pad would be misleading because:
- The actual stack may have additional elements from the caller
- The procedure may consume caller's stack elements

### Danger Zone

If an `exec` or `dynexec` procedure's stack falls below the elements it expects, it will consume stack items from its caller, potentially leading to unexpected behavior. This is a bug and should be fixed by ensuring the procedure maintains sufficient stack depth and avoiding dropping more stack elements than available.

### Example of Dangerous Behavior

```masm
# => [num_approvers, threshold]
dropw # dropw drops 4 elements, which will result in "negative" stack consumption (consuming 2 elements from the caller's stack)
```


## Intermediate States

Inside a procedure, the stack may temporarily exceed 16 elements:

```masm
# => [num_approvers, threshold, MULTISIG_CONFIG, pad(12)]
#     ^--- 18 elements total, must be reduced before return
```

For context-switching procedures these extra elements must be explicitly dropped before the procedure returns (directly or via called procedures), so the visible depth is exactly 16 on return.

## Debugging Stack Depth

When unsure whether the stack matches the depth you expect, inspect it at runtime with the `miden::core::debug` procedures. There are no `debug.*` decorators — print-style debugging is done with ordinary procedure calls that `emit` a well-known event the host handles by printing VM state:

```masm
use miden::core::debug

exec.debug::print_stack        # print the entire operand stack
```

The module exports `print_stack`, `print_mem`, `print_mem_addr`, `print_mem_all`, `print_adv_stack`, `print_adv_stack_all`, `print_adv_map_all`, and `print_adv_map_item`. `print_mem` takes `[start, end]` and consumes both; a range wider than 1024 addresses aborts execution.

- `sdepth` – push the current stack depth onto the stack as a felt; useful when you need depth as a runtime value, e.g. to assert it:

  ```masm
  sdepth push.16 eq assert.err="depth must be 16 here"
  ```

**These procedures print unconditionally.** They are `emit`-based, carry no MAST decorator cost, and there is no debug-mode toggle that suppresses them — `miden-vm run` has no `--debug` or `--release` flag. Anything you leave in ships and prints on every execution, so remove every `exec.debug::print_*` call before committing production MASM.

## Validation Checklist

For all invocation types:
- [ ] Inline `# =>` trackers reflect the post-auto-pad depth (never below 16) at the context-switch boundaries that enforce the floor (`call`, `dyncall`, `syscall`, note scripts, tx scripts)
- [ ] No `exec.debug::print_*` call is left in production MASM (they print unconditionally)

For context-switching procedures (`call`, `dyncall`, `syscall`):
- [ ] On return, the visible stack depth is exactly 16 (the VM traps otherwise)
- [ ] Inputs/Outputs doc comments prefer `pad(N)`-to-16; logical-only documentation on existing procs is acceptable, not a bug
- [ ] Inline comments use `# =>` format with `pad(N)` notation
- [ ] All intermediate states track the full stack including padding

For same-context procedures (`exec`, `dynexec`):
- [ ] Padding follows the caller — logical (un-padded) I/O when the caller is logical; `pad(N)`-to-16 when the caller works in a padded-to-16 context (e.g. kernel API `dynexec` procs)
- [ ] Verify the stack never drops below the elements the procedure expects, so it does not consume the caller's stack
