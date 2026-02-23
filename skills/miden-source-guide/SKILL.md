---
name: miden-source-guide
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

**Build loop**: After every contract edit, run `cargo miden build --manifest-path contracts/<name>/Cargo.toml --release`. The project's build hook does this automatically. If the build fails:
1. Read the error message
2. Search the source repos for a working example of the pattern that failed
3. Adapt the working pattern to your use case
4. Rebuild

**Test loop**: Write tests alongside contracts. Run with `cargo test -p integration --release`. When tests fail:
1. Check the error — is it a build error, a runtime assertion, or a proof failure?
2. For assertion failures: check felt arithmetic (modular wrapping) and storage slot naming
3. For unexpected behavior: compare your code against the closest working example in source repos

Never submit code that doesn't compile and pass tests. The verification loop is your quality guarantee.

### 3. Context Engineering with Source Repos

The basic skills (rust-sdk-patterns, miden-testing-patterns, miden-concepts, miden-pitfalls) cover standard patterns. For anything beyond those patterns, Miden's source repositories are the knowledge base.

**How to use source repos effectively**:
- Don't load entire repos into context. Use sub-agents to explore — they search, read relevant files, and summarize findings without filling the main conversation context.
- Read source files only when you need a specific answer (progressive disclosure)
- Look for working examples first, then adapt. Working code that compiles is more reliable than documentation.
- When you find a useful pattern in source, extract just what you need — the exact API call, the exact data layout, the exact test setup.

**Using sub-agents for exploration**:
- Launch an explore sub-agent with a specific question: "Find how P2ID output notes are created in the miden-bank repository"
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
# Required: contains standard note types and account components
git clone --depth 1 https://github.com/0xMiden/miden-base.git ../miden-base

# Required: contains SDK, compiler, and 12 working examples
git clone --depth 1 https://github.com/0xMiden/compiler.git ../compiler

# Required: contains client API for deployment and chain interaction
git clone --depth 1 https://github.com/0xMiden/miden-client.git ../miden-client

# Recommended: complete working banking app with advanced patterns
git clone https://github.com/keinberger/miden-bank.git ../miden-bank
```

### `compiler/` — The Rust-to-MASM Compiler

Contains the SDK that powers `#[component]`, `#[note]`, and `#[tx_script]` macros.

- **`examples/`** — 12 working examples covering every SDK pattern: account components, note scripts, transaction scripts, authentication components, wallets, faucets, storage. These are the most reliable reference for "how to write X" questions.
- **`sdk/`** — SDK macro implementations and type definitions. Explore when you need to understand exactly what a macro does or what types are available.

**Explore when**: Writing any new contract type, understanding how macros work internally, finding working code examples for patterns not covered by skills.

### `miden-base/` — Protocol Layer and Standard Library

Contains the protocol specification, standard components, and standard note types.

- **`crates/miden-standards/`** — Standard note types (P2ID, P2IDE, SWAP, BURN, MINT), standard account components (BasicWallet, BasicFungibleFaucet, authentication components), and their MASM implementations. This is where you find how standard patterns work.
- **`crates/miden-tx/`** — Transaction kernel and execution logic. Explore when you need to understand how transactions are processed.
- **`crates/miden-testing/`** — MockChain implementation internals. Explore when you need to understand testing infrastructure beyond what the testing-patterns skill covers.

**Explore when**: Using standard components, understanding note flows, finding how P2ID/SWAP/faucet patterns work, understanding transaction execution.

### `miden-client/` — Client Library

Contains the Rust API for deploying contracts and interacting with the Miden network.

- Rust client for building transactions, syncing state, managing accounts and notes
- CLI tool source code for reference on client usage patterns

**Explore when**: Deploying contracts to testnet, submitting transactions, syncing state, managing notes on-chain.

### `miden-bank/` — Working Example Application

A complete banking application built with the Rust SDK. Demonstrates advanced patterns that go beyond the basic skills.

- Multiple contract types working together (account, deposit note, withdraw note, tx script)
- Advanced patterns: StorageMap + Value composition, felt arithmetic safety, cross-component calls, P2ID output note creation from within contracts
- Multi-step integration tests with output note verification

**Explore when**: Building multi-contract applications, understanding how pieces fit together, seeing a complete working app end-to-end.

---

## What to Explore for Each Contract Type

| Building This | Explore These Repos | What to Look For |
|---|---|---|
| Account component with storage | `compiler/` examples, `miden-bank/` contracts | StorageMap/Value patterns, pub method signatures |
| Note script | `compiler/` examples, `miden-bank/` contracts | `#[note_script]` pattern, cross-component calls, note inputs parsing |
| Transaction script | `compiler/` examples, `miden-bank/` contracts | `#[tx_script]` pattern, Account binding import |
| Authentication component | `compiler/` examples, `miden-base/` standards | Auth component patterns (NoAuth, Falcon512, ECDSA) |
| Faucet (token minting) | `compiler/` examples, `miden-base/` standards | BasicFungibleFaucet, mint/burn pattern |
| P2ID output notes | `miden-bank/` contracts, `miden-base/` standards | Recipient computation, script root, output_note creation |
| Swap notes | `miden-base/` standards | SwapNote builder, tag construction, payback flow |
| Multi-step tests | `miden-bank/` integration tests | Init → operate → verify flow, output note verification |
| Client deployment | `miden-client/` | TransactionRequestBuilder, sync, submit patterns |

---

## Common Advanced Patterns

These patterns go beyond what the basic skills cover. For each, the source repos contain working implementations.

### Multi-Component Accounts
Compose multiple components (e.g., BasicWallet + custom logic + authentication) into a single account. Standard components in `miden-base/` show how each is implemented. The `compiler/` examples show how to compose them.

### Output Note Creation from Contracts
Create output notes (like P2ID) from within contract code. Requires computing a Recipient hash from serial number, script root, and inputs. The `miden-bank/` withdraw pattern demonstrates this end-to-end.

### Note Inputs Protocol
Pass structured data to notes via `Vec<Felt>` inputs. Define your input layout, document the field ordering, and parse inputs in the `#[note_script]` function. The `miden-bank/` withdraw-request-note demonstrates parsing 10 Felts into asset, serial number, tag, and note type.

### Atomic Swaps
The standard SwapNote in `miden-base/` creates a payback P2ID note automatically when consumed. Explore the SwapNote builder to understand tag construction, storage layout, and the payback mechanism.

### Account Initialization
Use `#[tx_script]` to initialize accounts before they accept operations. The `miden-bank/` init-tx-script calls `account.initialize()` to set an initialization flag, which is checked before every operation.

### Token Creation (Faucets)
Faucet accounts mint and burn fungible or non-fungible tokens. The `compiler/` fungible-faucet example and `miden-base/` BasicFungibleFaucet standard component show how to create and manage tokens.

### P2ID with Expiration (P2IDE)
Send assets with a deadline — the sender can reclaim after the block height passes. The `compiler/` p2ide-note example and `miden-base/` P2IDE standard show the timelock pattern.
