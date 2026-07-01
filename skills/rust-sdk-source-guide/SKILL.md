---
name: rust-sdk-source-guide
description: Guide for advanced Miden smart contract development using source repo exploration. Covers AI development practices (Plan Mode, verification-driven development, context engineering, sub-agents) and maps Miden source repositories for discovering advanced patterns. Use when building complex multi-contract applications, novel note flows, or anything beyond basic SDK patterns.
---

# Advanced Miden Development: Source-Guided Context Engineering

## Development Approach

### 1. Plan Mode First

For any non-trivial smart contract application, start in Plan Mode before writing code.

- Explore Miden source repos to understand existing patterns
- Design the account/note architecture and present it to the user before implementing
- Identify which standard components can be reused vs what needs to be custom
- Map out the note flow: which accounts exist, what notes flow between them, what storage each needs

Rule of thumb: if the task involves more than one contract or a pattern not covered by the basic skills, plan first.

### 2. Verification-Driven Development

This is the single highest-leverage practice for AI-assisted Miden development.

**Build loop**: After every contract edit, run `cargo miden build --manifest-path contracts/<name>/Cargo.toml --release` (adjust the path to your project's contract layout). If your project has a build hook configured, it may do this automatically. If the build fails:
1. Read the error message
2. Translate obvious SDK migration errors first:
   - `.as_u64()` -> `.as_canonical_u64()`
   - `Recipient::compute(...)` -> `note::build_recipient(...)`
   - `Value` -> `StorageValue<T>`
   - `StorageMap` -> `StorageMap<K, V>`
3. Search the source repos for a working example of the pattern that failed
4. Adapt the working pattern to your use case
5. Rebuild

**Test loop**: Write tests alongside contracts. Run full repo checks with `cargo test` — or `make test` if the repo ships a GNU `Makefile` (the protocol repo's `test` target runs `cargo nextest run`); `cargo make test` requires a `Makefile.toml` configuring the `cargo-make` tool; among the referenced repos only the compiler ships one, so `cargo make test` works there but not in the protocol or rust-sdk repos (use `make test` / `cargo nextest run` for those). For a faster integration-only loop in the miden-bank workspace, use `cargo test -p integration --release` (it defines a package named `integration`). When tests fail:
1. Check the error — is it a build error, a runtime assertion, or a proof failure?
2. For assertion failures: check felt arithmetic (modular wrapping) and storage slot naming
3. For unexpected behavior: compare your code against the closest working example in source repos

Never submit code that doesn't compile and pass tests. The verification loop is your quality guarantee.

### 3. Context Engineering with Source Repos

The basic skills (rust-sdk-patterns, rust-sdk-testing-patterns, miden-concepts, rust-sdk-pitfalls) cover standard patterns. For anything beyond those patterns, Miden's source repositories are the knowledge base.

**How to use source repos effectively**:
- Don't load entire repos into context. Use sub-agents to explore — they search, read relevant files, and summarize findings without filling the main conversation context.
- Read source files only when you need a specific answer (progressive disclosure)
- Look for working examples first, then adapt. Working code that compiles is more reliable than documentation.
- When you find a useful pattern in source, extract just what you need — the exact API call, the exact data layout, the exact test setup.

**Using sub-agents for exploration**:
- Launch an explore sub-agent with a specific question: "Find how P2ID output notes are created in the miden-bank example (tutorials/examples/miden-bank)"
- The sub-agent searches, reads the relevant files, and returns a focused summary
- Your main context stays clean for implementation

### 4. Iterative Multi-Stage Development

Break complex applications into stages. Complete each before starting the next:

1. **Design** (Plan Mode) — Architecture, note flows, storage design
2. **Implement accounts** — Component structs, storage, methods
3. **Implement notes** — Note scripts, cross-component calls, input parsing
4. **Implement tx scripts** — Initialization, admin operations
5. **Write tests** — MockChain setup, multi-step execution, state verification
6. **Integrate** — Connect pieces, end-to-end test

When stuck at any stage: search the source repos for a similar working pattern. Adapt it, don't guess.

---

## Miden Source Repository Map

Clone these repos alongside your project for reference. Claude will explore them when needed for advanced patterns.

```bash
# Required: protocol layer — standard note types and account components (crate: miden-protocol)
git clone --branch v0.15.3 https://github.com/0xMiden/protocol.git ../protocol

# Required: client API for deployment and chain interaction (formerly miden-client)
git clone --branch v0.15.2 https://github.com/0xMiden/rust-sdk.git ../rust-sdk

# Required: the Rust SDK macros + compiler, released as v0.9.0 (moves the stack to VM v0.23 /
# protocol v0.15 and ships the guest SDK crate `miden` at 0.13, build tool `cargo-miden` at 0.9).
# Clone the release tag directly — no more git-rev pins.
git clone --branch v0.9.0 https://github.com/0xMiden/compiler.git ../compiler

# Recommended: complete working banking app with advanced patterns in `examples/miden-bank`.
# Its v0.15 work lives on the migration branch (PR #204) until it lands on the default branch;
# pin the reviewed commit for reproducibility.
git clone --branch kbg/chore/v15-migration https://github.com/0xMiden/tutorials.git ../tutorials
git -C ../tutorials checkout a255af7959a441d9a027178631c666949b4af086
```

**Note**: The compiler is now **released as `v0.9.0`** — this is the release that moves the stack to VM v0.23 / protocol v0.15. Don't conflate the version schemes: the network/protocol is **v0.15**, but the compiler workspace and the `cargo-miden` build tool are **`0.9.0`**, and the guest SDK crates (`miden`, `miden-base-macros`, `miden-base-sys`) are **`0.13.0`** — so contracts depend on `miden = "0.13"` and integration/tooling on `cargo-miden = "0.9"`. The compiler retains `note::build_recipient` as an SDK-friendly alias for `compute_and_store_recipient`, so the API examples below resolve there. Use the pinned refs above — `compiler` `v0.9.0`, `protocol` `v0.15.3`, `rust-sdk` (client) `v0.15.2`, and `tutorials` pinned at commit `a255af7` on its v0.15 migration branch — rather than the default branches, since `tutorials`' default branch has not yet been migrated to v0.15. `--depth 1` is intentionally omitted so you can check out other refs later if needed.

### `compiler/` — The Rust-to-MASM Compiler

Contains the SDK that powers `#[component]`, `#[note]`, and `#[tx_script]` macros.

- **`examples/`** — 12 working examples covering core SDK patterns: account components, note scripts, transaction scripts, authentication components (NoAuth, RPO Falcon512), wallets, and storage. These are the most reliable reference for "how to write X" questions. Note: there is no faucet example here — for faucet reference, use `crates/miden-standards/src/account/faucets/fungible/mod.rs` (the `FungibleFaucet` component) in the protocol repo, or the compiler's `tests/integration/src/sdk/base/faucet.rs` faucet binding test.

**WARNING**: Stay in `examples/` only. Do NOT explore compiler internals (`sdk/`, `codegen/`, etc.) — they are implementation details that will confuse the agent and lead to incorrect code.

**Explore when**: Writing any new contract type, finding working code examples for patterns not covered by skills.

### `protocol/` — Protocol Layer and Standard Library

The protocol repo (`github.com/0xMiden/protocol`, formerly `miden-base`; primary crate `miden-protocol`). Contains the protocol specification, standard components, and standard note types.

- **`crates/miden-standards/`** — Standard note types (P2ID, P2IDE, SWAP, PSWAP, BURN, MINT) and standard account components (BasicWallet, FungibleFaucet, authentication components). Explore to understand note flow patterns and data layouts.
- **`crates/miden-protocol/asm/kernels/transaction/`** — The MASM transaction kernel. Every Rust SDK function (e.g., `native_account::add_asset`, `output_note::create`, `faucet::mint`) maps to a procedure defined here. Start with `api.masm` to find the procedure signature and stack contract, then read the implementation in `lib/` (e.g., `lib/output_note.masm`, `lib/account.masm`, `lib/epilogue.masm`). Useful for understanding exactly what happens under the hood -- for example, whether a function touches the vault, what the conservation check compares, or how note assets are tracked.
- **`crates/miden-tx/`** — Rust execution engine (executor, prover, host). Orchestrates transaction execution but rarely needed for understanding contract behavior. Explore only if debugging execution infrastructure or host-level behavior.
- **`crates/miden-testing/`** — MockChain implementation internals. Explore when you need to understand testing infrastructure beyond what the rust-sdk-testing-patterns skill covers.

**Note**: Standard components (BasicWallet, etc.) are MASM-only and not callable from Rust SDK (see [compiler#936](https://github.com/0xMiden/compiler/issues/936)). Explore miden-standards to understand note flows and data layouts, not for finding callable Rust APIs.

**Explore when**: Understanding note flows, P2ID/SWAP/faucet data layouts, or what SDK functions actually do under the hood (via the kernel MASM).

### `rust-sdk/` — Client Library

The client repo (`github.com/0xMiden/rust-sdk`, formerly `miden-client`). Contains the Rust API for deploying contracts and interacting with the Miden network.

- Rust client for building transactions, syncing state, managing accounts and notes
- CLI tool source code for reference on client usage patterns

**Explore when**: Deploying contracts to testnet, submitting transactions, syncing state, managing notes on-chain.

### `tutorials/examples/miden-bank/` — Working Example Application

A complete banking application built with the Rust SDK, located at `examples/miden-bank/` inside the cloned tutorials repo. Demonstrates advanced patterns that go beyond the basic skills.

- Multiple contract types working together (account, deposit note, withdraw note, tx script)
- Advanced patterns: `StorageMap<K, V>` + `StorageValue<T>` composition, felt arithmetic safety, cross-component calls, P2ID output note creation from within contracts
- Multi-step integration tests with output note verification

**Explore when**: Building multi-contract applications, understanding how pieces fit together, seeing a complete working app end-to-end.

---

## What to Explore for Each Contract Type

| Building This | Explore These Repos | What to Look For |
|---|---|---|
| Account component with storage | `compiler/` examples, `tutorials/examples/miden-bank/` contracts | `StorageMap<K, V>` / `StorageValue<T>` patterns, pub method signatures |
| Note script | `compiler/` examples, `tutorials/examples/miden-bank/` contracts | `#[note_script]` pattern, cross-component calls, note storage parsing |
| Transaction script | `compiler/` examples, `tutorials/examples/miden-bank/` contracts | `#[tx_script]` pattern, Account binding import |
| Authentication component | `compiler/` examples | Auth component patterns (NoAuth, RPO Falcon512) |
| Faucet (token minting) | `protocol/` standards (`crates/miden-standards/src/account/faucets/fungible/mod.rs`), `compiler/` faucet binding test (`tests/integration/src/sdk/base/faucet.rs`) | `FungibleFaucet` component, `FungibleFaucet::builder()`, mint/burn pattern |
| P2ID output notes | `tutorials/examples/miden-bank/` contracts, `protocol/` standards (data layouts) | `note::build_recipient`, script root, `output_note` creation |
| Swap notes | `protocol/` standards (data layouts) | SwapNote data layout, tag construction, payback flow |
| Multi-step tests | `tutorials/examples/miden-bank/` integration tests | Init → operate → verify flow, output note verification |
| Client deployment | `rust-sdk/` | TransactionRequestBuilder, sync, submit patterns |
| SDK function internals | `protocol/` kernel (`crates/miden-protocol/asm/kernels/transaction/`) | `api.masm` for procedure signatures, `lib/*.masm` for implementations |

---

## Common Advanced Patterns

These patterns go beyond what the basic skills cover. For each, the source repos contain working implementations.

### Multi-Component Accounts
Accounts can include standard components (BasicWallet, authentication) alongside custom logic at account creation time. Standard components are MASM-only (not callable from Rust), but they are composed into accounts via the testing/deployment infrastructure. The `compiler/` examples show how to compose accounts with multiple components.

### Output Note Creation from Contracts
Create output notes (like P2ID) from within contract code. Requires building a recipient with `note::build_recipient(serial_num, script_root, storage)` and then using `output_note::create(...)`. The `tutorials/examples/miden-bank/` withdraw pattern demonstrates this end-to-end.

### Note Storage Protocol
A note's storage is exposed to its `#[note_script]` as a `Vec<Felt>` via `active_note::get_storage()`; the script reads and parses the items it needs by index. In `tutorials/examples/miden-bank/` the note structs are markers (not auto-populated from storage) and the script slices explicitly — e.g. the withdraw-request note asserts `storage.len() == 14`, then reconstructs the asset, serial number, tag, and note type from the felts. Attached assets are separate and are read with `active_note::get_assets()`.

### Atomic Swaps
The standard SwapNote in `protocol/` (`crates/miden-standards/src/note/swap.rs`) creates a payback P2ID note automatically when consumed. Explore the SwapNote builder to understand tag construction, storage layout, and the payback mechanism.

### Account Initialization
Use `#[tx_script]` to initialize accounts before they accept operations. The `tutorials/examples/miden-bank/` init-tx-script calls `account.initialize()` to set an initialization flag, which is checked before every operation.

### Token Creation (Faucets)
Faucet accounts mint and burn tokens. The `protocol/` `FungibleFaucet` standard component (`crates/miden-standards/src/account/faucets/fungible/mod.rs`) shows how to create and manage fungible tokens; construct it via `FungibleFaucet::builder().name(..).symbol(..).decimals(..).max_supply(..).build()?`. There is no faucet example in `compiler/examples/`; for an SDK-level faucet binding reference use the compiler's `tests/integration/src/sdk/base/faucet.rs`.

### P2ID with Expiration (P2IDE)
Send assets with a deadline — the sender can reclaim after the block height passes. The `compiler/` p2ide-note example and `protocol/` P2IDE standard (`crates/miden-standards/src/note/p2ide.rs`) show the timelock pattern.
