# Miden Agent Skills

Skills for AI agents when working with Miden Assembly (.masm) files.

## Skills

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

Skills are applied automatically when the agent detects relevant tasks (editing, reviewing, or creating .masm files). You can also invoke them explicitly via most agent interfaces.    
