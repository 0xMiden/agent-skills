---
name: rust-sdk-patterns
description: Complete guide to writing Miden smart contracts with the Rust SDK. Covers #[component], #[note], #[tx_script] macros, storage patterns, native functions, asset handling, cross-component calls, and P2ID note creation. Use when writing, editing, or reviewing Miden Rust contract code.
---

# Miden Rust SDK Patterns

## Three Contract Types

### Account Component (`#[component]`)
Defines reusable logic and storage for accounts. Accounts are composed of one or more components.

```rust
#![no_std]
#![feature(alloc_error_handler)]
use miden::{component, felt, Felt, StorageMap, StorageMapAccess, Value, Word};

#[component]
struct MyComponent {
    #[storage(description = "a simple flag")]
    flag: Value,

    #[storage(description = "balance mapping")]
    balances: StorageMap,
}

#[component]
impl MyComponent {
    // Read-only method
    pub fn get_flag(&self) -> Word {
        self.flag.read()
    }

    // Mutating method
    pub fn set_flag(&mut self, val: Word) {
        self.flag.write(val);
    }

    // StorageMap access
    pub fn get_balance(&self, key: Word) -> Felt {
        self.balances.get(&key)
    }

    pub fn set_balance(&mut self, key: Word, val: Felt) {
        self.balances.set(key, val);
    }
}
```

**Cargo.toml for accounts:**
```toml
[lib]
crate-type = ["cdylib"]

[dependencies]
miden = { version = "0.9" }

[package.metadata.component]
package = "miden:my-component"

[package.metadata.miden]
project-kind = "account"
supported-types = ["RegularAccountImmutableCode"]
```

### Note Script (`#[note]`)
Executes when a note is consumed by an account. Can call component methods on the consuming account.

```rust
#![no_std]
#![feature(alloc_error_handler)]
use miden::*;
use crate::bindings::miden::my_component::my_component;

#[note]
struct MyNote;

#[note]
impl MyNote {
    #[note_script]
    fn run(self, _arg: Word) {
        let sender = active_note::get_sender();
        let assets = active_note::get_assets();
        for asset in assets {
            my_component::deposit(sender, asset);
        }
    }
}
```

**Cargo.toml for notes:**
```toml
[lib]
crate-type = ["cdylib"]

[dependencies]
miden = { version = "0.9" }

[package.metadata.component]
package = "miden:my-note"

[package.metadata.miden.dependencies]
"miden:my-component" = { path = "../my-component" }

[package.metadata.component.target.dependencies]
"miden:my-component" = { path = "../my-component/target/generated-wit/" }

[package.metadata.miden]
project-kind = "note-script"
```

### Transaction Script (`#[tx_script]`)
One-off logic executed in the context of an account. Used for initialization, admin operations, etc.

```rust
#![no_std]
#![feature(alloc_error_handler)]
use miden::*;
use crate::bindings::Account;

#[tx_script]
fn run(_arg: Word, account: &mut Account) {
    account.initialize();
}
```

**Cargo.toml:** Same as account but with `project-kind = "tx-script"`.

## Storage Types

| Type | Usage | Read | Write |
|------|-------|------|-------|
| `Value` | Single Word slot (flags, simple state) | `.read() -> Word` | `.write(Word)` |
| `StorageMap` | Key-value mapping (balances, records) | `.get(&Word) -> Felt` | `.set(Word, Felt)` |

**Storage keys** are always `Word` (4 Felts). Use `Word::from_u64_unchecked(a, b, c, d)` or `Word::from([f0, f1, f2, f3])`.

## Native Function Modules

| Module | Key Functions | Purpose |
|--------|--------------|---------|
| `native_account::` | `add_asset(Asset)`, `remove_asset(Asset)`, `incr_nonce()` | Modify account vault/nonce |
| `active_account::` | `get_id() -> AccountId`, `get_balance(AccountId) -> Felt` | Query current account |
| `active_note::` | `get_inputs() -> Vec<Felt>`, `get_assets() -> Vec<Asset>`, `get_sender() -> AccountId` | Query note being consumed |
| `output_note::` | `create(Tag, NoteType, Recipient) -> NoteIdx`, `add_asset(Asset, NoteIdx)` | Create output notes |
| `faucet::` | `create_fungible_asset(Felt) -> Asset`, `mint(Asset)`, `burn(Asset)` | Asset minting |
| `tx::` | `get_block_number() -> Felt`, `get_block_timestamp() -> Felt` | Transaction context |
| Intrinsics | `assert(bool)`, `assertz(Felt)`, `assert_eq(Felt, Felt)` | Validation |

## Asset Handling

Fungible asset Word layout: `[amount, 0, faucet_suffix, faucet_prefix]`

```rust
// Access asset amount
let amount = asset.inner[0];

// Add asset to account vault
native_account::add_asset(asset);

// Remove asset from account vault
native_account::remove_asset(asset.clone());
```

## P2ID Output Note Creation

To send assets to another account, create a P2ID (Pay-to-ID) output note:

```rust
fn create_p2id_note(&mut self, serial_num: Word, asset: &Asset,
                     recipient_id: AccountId, tag: Felt, note_type: Felt) {
    let tag = Tag::from(tag);
    let note_type = NoteType::from(note_type);
    let script_root = Self::p2id_note_root(); // Hardcoded P2ID script digest

    // P2ID inputs: [suffix, prefix] of recipient
    let recipient = Recipient::compute(serial_num, script_root,
        vec![recipient_id.suffix, recipient_id.prefix]);

    let note_idx = output_note::create(tag, note_type, recipient);
    native_account::remove_asset(asset.clone());
    output_note::add_asset(asset.clone(), note_idx);
}
```

## Note Inputs

Notes receive data via inputs (Vec<Felt>), accessed with `active_note::get_inputs()`:

```rust
let inputs = active_note::get_inputs();
// Parse: Asset = inputs[0..4], serial_num = inputs[4..8], tag = inputs[8], type = inputs[9]
let asset = Asset::new(Word::from([inputs[0], inputs[1], inputs[2], inputs[3]]));
let serial_num = Word::from([inputs[4], inputs[5], inputs[6], inputs[7]]);
```

## Cross-Component Dependencies

To call another component's methods from a note or tx script:

1. Add to note's Cargo.toml:
```toml
[package.metadata.miden.dependencies]
"miden:target-component" = { path = "../target-component" }

[package.metadata.component.target.dependencies]
"miden:target-component" = { path = "../target-component/target/generated-wit/" }
```

2. Import the bindings:
```rust
use crate::bindings::miden::target_component::target_component;
// Call: target_component::method_name(args);
```

## Common Type Conversions

```rust
// Felt from integer
let f = felt!(42);
let f = Felt::new(42);
let f = Felt::from_u32(42);
let f = Felt::from_u64_unchecked(42);

// Word from Felts
let w = Word::from([f0, f1, f2, f3]);
let w = Word::from_u64_unchecked(0, 0, 0, 1);
let w = Word::new([f0, f1, f2, f3]);

// Felt to u64 (for comparisons and arithmetic safety)
let n: u64 = f.as_u64();
```

## No-std Requirements

Every contract file must include:
```rust
#![no_std]
#![feature(alloc_error_handler)]
```

If you need heap allocation (Vec, String, etc.):
```rust
extern crate alloc;
use alloc::vec::Vec;
```

## Validation Checklist

- [ ] `#![no_std]` and `#![feature(alloc_error_handler)]` at top of every contract
- [ ] `crate-type = ["cdylib"]` in Cargo.toml
- [ ] Correct `project-kind` in `[package.metadata.miden]`
- [ ] Cross-component deps in both `[package.metadata.miden.dependencies]` and `[package.metadata.component.target.dependencies]`
- [ ] Felt arithmetic validated before subtraction (see miden-pitfalls skill)
- [ ] Felt comparisons use `.as_u64()` (see miden-pitfalls skill)
