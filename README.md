# Miden Agent Skills

Skills for AI agents when working with Miden projects.

## Skills

### Rust SDK (Smart Contract Development)

- **rust-sdk-patterns** – Complete guide to `#[component]`, `#[note]`, `#[tx_script]` macros, storage patterns, native functions, asset handling, and cross-component calls
- **rust-sdk-testing-patterns** – MockChain testing workflow, account/note creation, storage verification, and multi-step test patterns
- **miden-concepts** – Miden architecture from a developer perspective: actor model, accounts, notes, transactions, assets, privacy
- **rust-sdk-pitfalls** – Critical safety rules: felt arithmetic, comparison operators, stack limits, argument limits, storage naming, no-std
- **rust-sdk-source-guide** – Advanced development guide: AI practices (Plan Mode, verification-driven development, sub-agents, context engineering) and Miden source repository map for discovering patterns beyond basic skills

### React Frontend Development

- **react-sdk-patterns** – Complete `@miden-sdk/react` hook API reference: MidenProvider, query hooks, mutation hooks, transaction stages, signer integration, utilities
- **frontend-pitfalls** – Critical frontend pitfalls: WASM init race, recursive access crash, COOP/COEP headers, BigInt handling, Bech32 mismatch, IndexedDB state loss
- **vite-wasm-setup** – Vite + WASM configuration: required plugins, deployment headers (Nginx, Vercel, Cloudflare), TypeScript config, troubleshooting
- **frontend-source-guide** – Advanced frontend development guide: AI practices and miden-client source repository map for discovering patterns beyond basic skills

### Miden Assembly

- **masm-inline-comments** – Inline commenting conventions for .masm files (lowercase, avoid over-commenting)
- **masm-doc-comments** – Procedure documentation format (`#!` doc blocks with Inputs, Outputs, Where, Panics, Invocation)
- **masm-padding** – Stack padding conventions for `call` vs `exec` procedures

## Installation

### Cursor

```bash
git clone https://github.com/0xMiden/agent-skills.git
ln -sf "$(pwd)/agent-skills/skills" ~/.cursor/skills
```

### Claude Code

```bash
git clone https://github.com/0xMiden/agent-skills.git
ln -sf "$(pwd)/agent-skills/skills" ~/.claude/skills
```

### Codex

```bash
git clone https://github.com/0xMiden/agent-skills.git
ln -sf "$(pwd)/agent-skills/skills" ~/.codex/skills
```

**Note:** If `~/.cursor/skills` already exists with other skills, or you want to selectively import skills, symlink individual skills instead:

```bash
cd ~/{.cursor/.claude/.codex}/skills
ln -sf /path/to/agent-skills/skills/masm-padding .
```

## Usage

Skills are applied automatically when the agent detects relevant tasks. You can also invoke them explicitly via most agent interfaces.
