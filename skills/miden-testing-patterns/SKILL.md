---
name: miden-testing-patterns
description: Guide to testing Miden smart contracts with MockChain. Covers test setup, contract building, account/note creation, transaction execution, storage verification, faucet setup, and output note verification. Use when writing, editing, or debugging Miden integration tests.
---

# Miden Testing Patterns (MockChain)

## Test File Setup

Tests go in `integration/tests/`. All tests are async and use MockChain for local execution without a network.

```rust
use integration::helpers::{
    build_project_in_dir, create_testing_account_from_package, create_testing_note_from_package,
    AccountCreationConfig, NoteCreationConfig,
};
use miden_client::{
    account::{StorageMap, StorageSlot, StorageSlotName},
    transaction::OutputNote,
    Felt, Word,
};
use miden_testing::{Auth, MockChain};
use std::{path::Path, sync::Arc};

#[tokio::test]
async fn my_test() -> anyhow::Result<()> {
    // ... test body ...
    Ok(())
}
```

## Step-by-Step Test Pattern

### 1. Initialize MockChain Builder
```rust
let mut builder = MockChain::builder();
```

### 2. Create Sender/Wallet Accounts
```rust
// Simple wallet (no assets)
let sender = builder.add_existing_wallet(Auth::BasicAuth)?;

// Wallet with assets
let sender = builder.add_existing_wallet_with_assets(
    Auth::BasicAuth,
    [FungibleAsset::new(faucet.id(), 100)?.into()],
)?;
```

### 3. Set Up Faucets (for fungible assets)
```rust
let faucet = builder.add_existing_basic_faucet(
    Auth::BasicAuth,
    "TOKEN",     // token symbol
    1000,        // max supply
    Some(10),    // decimals (None for 0)
)?;
```

### 4. Build Contracts
```rust
let contract_package = Arc::new(build_project_in_dir(
    Path::new("../contracts/my-account"),
    true, // release mode
)?);
let note_package = Arc::new(build_project_in_dir(
    Path::new("../contracts/my-note"),
    true,
)?);
```

### 5. Create Account with Storage

**Storage slot naming convention** (CRITICAL):
```
miden::component::[snake_case(package.metadata.component.package)]::[field_name]
```

Examples:
- Package `miden:counter-account`, field `count_map` → `miden::component::miden_counter_account::count_map`
- Package `miden:bank-account`, field `balances` → `miden::component::miden_bank_account::balances`

Rule: Replace colons and hyphens with underscores in the package name.

```rust
// Define storage slots
let slot_name = StorageSlotName::new("miden::component::miden_counter_account::count_map").unwrap();

// StorageMap slot (key-value mapping)
let key = Word::from([Felt::new(0), Felt::new(0), Felt::new(0), Felt::new(1)]);
let initial_value = Word::from([Felt::new(0), Felt::new(0), Felt::new(0), Felt::new(0)]);
let storage_slots = vec![StorageSlot::with_map(
    slot_name.clone(),
    StorageMap::with_entries([(key, initial_value)]).unwrap(),
)];

// Value slot (single Word)
let value_slot_name = StorageSlotName::new("miden::component::miden_bank_account::initialized").unwrap();
let storage_slots = vec![StorageSlot::with_value(
    value_slot_name.clone(),
    Word::default(),
)];

// Create account with storage
let cfg = AccountCreationConfig {
    storage_slots,
    ..Default::default()
};
let mut account = create_testing_account_from_package(contract_package.clone(), cfg).await?;
```

### 6. Create Notes
```rust
// Simple note (no assets, no inputs)
let note = create_testing_note_from_package(
    note_package.clone(),
    sender.id(),
    NoteCreationConfig::default(),
)?;

// Note with assets and inputs
use miden_client::note::NoteAssets;
use miden_standards::notes::FungibleAsset;

let note_assets = NoteAssets::new(vec![FungibleAsset::new(faucet.id(), 50)?.into()])?;
let note = create_testing_note_from_package(
    note_package.clone(),
    sender.id(),
    NoteCreationConfig {
        assets: note_assets,
        inputs: vec![Felt::new(42), Felt::new(0)],  // custom note inputs
        ..Default::default()
    },
)?;
```

### 7. Add to MockChain and Build
```rust
builder.add_account(account.clone())?;
builder.add_output_note(OutputNote::Full(note.clone()));
let mut mock_chain = builder.build()?;
```

### 8. Execute Transaction
```rust
let tx_context = mock_chain
    .build_tx_context(account.id(), &[note.id()], &[])?
    .build()?;

let executed_transaction = tx_context.execute().await?;

// Apply state changes
account.apply_delta(executed_transaction.account_delta())?;

// Add to chain and prove
mock_chain.add_pending_executed_transaction(&executed_transaction)?;
mock_chain.prove_next_block()?;
```

### 9. Execute with Transaction Script
```rust
use miden_client::transaction::TransactionScript;

let tx_script_package = Arc::new(build_project_in_dir(
    Path::new("../contracts/my-tx-script"),
    true,
)?);
let program = tx_script_package.unwrap_program();
let tx_script = TransactionScript::new((*program).clone());

let tx_context = mock_chain
    .build_tx_context(account.id(), &[], &[])?
    .tx_script(tx_script)
    .build()?;

let executed = tx_context.execute().await?;
account.apply_delta(&executed.account_delta())?;
mock_chain.add_pending_executed_transaction(&executed)?;
mock_chain.prove_next_block()?;
```

### 10. Verify Storage State
```rust
// Read StorageMap value
let value = account.storage()
    .get_map_item(&slot_name, key)?;
assert_eq!(value, Word::from([Felt::new(0), Felt::new(0), Felt::new(0), Felt::new(1)]));
```

### 11. Verify Output Notes
```rust
use miden_client::note::{Note, NoteAssets, NoteMetadata, NoteRecipient};

let expected_note = Note::new(expected_assets, expected_metadata, expected_recipient);

let tx_context = mock_chain
    .build_tx_context(account.id(), &[note.id()], &[])?
    .extend_expected_output_notes(vec![OutputNote::Full(expected_note)])
    .build()?;

// execute() will verify output notes match
let executed = tx_context.execute().await?;
```

## Multi-Step Test Pattern

For contracts requiring initialization before use:

```rust
#[tokio::test]
async fn multi_step_test() -> anyhow::Result<()> {
    let mut builder = MockChain::builder();
    // ... setup ...
    let mut mock_chain = builder.build()?;

    // Step 1: Initialize (via tx script)
    let init_tx_context = mock_chain
        .build_tx_context(account.id(), &[], &[])?
        .tx_script(init_script)
        .build()?;
    let executed_init = init_tx_context.execute().await?;
    account.apply_delta(&executed_init.account_delta())?;
    mock_chain.add_pending_executed_transaction(&executed_init)?;
    mock_chain.prove_next_block()?;

    // Step 2: Main operation (via note consumption)
    let tx_context = mock_chain
        .build_tx_context(account.id(), &[note.id()], &[])?
        .build()?;
    let executed = tx_context.execute().await?;
    account.apply_delta(&executed.account_delta())?;
    mock_chain.add_pending_executed_transaction(&executed)?;
    mock_chain.prove_next_block()?;

    // Step 3: Verify state
    // ...

    Ok(())
}
```

## Key Dependencies (integration/Cargo.toml)

```toml
[dependencies]
cargo-miden = { git = "https://github.com/0xMiden/compiler", branch = "next" }
miden-client = { version = "0.13", features = ["tonic", "testing"] }
miden-standards = { version = "0.13", default-features = false, features = ["testing"] }
miden-testing = "0.13"
miden-core = { version = "0.20" }
tokio = { version = "1.40", features = ["rt-multi-thread", "net", "macros", "fs"] }
anyhow = "1.0"
```

## Validation Checklist

- [ ] Test function is `async` and uses `#[tokio::test]`
- [ ] Storage slot names follow `miden::component::package_name::field_name` pattern
- [ ] All contracts built before account/note creation
- [ ] `apply_delta()` called after each `execute()`
- [ ] `prove_next_block()` called after `add_pending_executed_transaction()`
- [ ] Notes added to builder via `add_output_note(OutputNote::Full(...))`
- [ ] Faucet set up before creating assets
