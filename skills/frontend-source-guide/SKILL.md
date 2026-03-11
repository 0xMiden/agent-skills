---
name: frontend-source-guide
description: Guide for advanced Miden frontend development using source repo exploration. Covers AI development practices (Plan Mode, verification-driven development, context engineering, sub-agents) and maps the miden-client source repository for discovering advanced patterns. Use when building complex applications beyond basic hook usage, implementing custom signers, working with raw WebClient, or troubleshooting SDK internals.
---

# Advanced Miden Frontend Development: Source-Guided Context Engineering

## Development Approach

### 1. Plan Mode First

For any non-trivial frontend application, start in Plan Mode before writing code.

- Explore React SDK source and examples to understand available patterns
- Design the component hierarchy, data flow, and which hooks to use
- Identify which built-in hooks cover your needs vs what requires raw WebClient
- Map out the user flow: account creation, token operations, note handling

Rule of thumb: if the task involves custom transactions, external signers, or patterns not covered by the basic skills, plan first.

### 2. Verification-Driven Development

This is the single highest-leverage practice for AI-assisted frontend development.

**Type check loop**: After every file edit, run `npx tsc -b --noEmit`. The project's type check hook does this automatically. If types fail:
1. Read the error message
2. Search the React SDK source for the correct type signature or hook usage
3. Adapt the working pattern to your use case
4. Recheck

**Dev server loop**: Run `npm run dev` and check the browser. When something fails:
1. Check the browser console for WASM errors, network errors, or React errors
2. For WASM errors: check COOP/COEP headers and Vite config (see frontend-pitfalls skill)
3. For unexpected behavior: compare your code against the example wallet in the React SDK

Never submit code that doesn't type-check. The verification loop is your quality guarantee.

### 3. Context Engineering with Source Repos

The basic skills (react-sdk-patterns, frontend-pitfalls, vite-wasm-setup) cover standard patterns. For anything beyond those patterns, the miden-client source repository is the knowledge base.

**How to use source repos effectively**:
- Don't load entire repos into context. Use sub-agents to explore — they search, read relevant files, and summarize findings without filling the main conversation context.
- Read source files only when you need a specific answer (progressive disclosure)
- Look for working examples first, then adapt. The example wallet app is the most reliable reference.
- When you find a useful pattern in source, extract just what you need — the exact hook call, the exact type, the exact provider setup.

**Using sub-agents for exploration**:
- Launch an explore sub-agent with a specific question: "Find how useSwap handles the payback note type in the React SDK"
- The sub-agent searches, reads the relevant files, and returns a focused summary
- Your main context stays clean for implementation

### 4. Iterative Frontend Development

Break complex applications into stages. Complete each before starting the next:

1. **Design** (Plan Mode) — Component hierarchy, data flow, hook selection
2. **Provider setup** — MidenProvider config, signer integration if needed
3. **Query components** — Account display, balance rendering, note lists
4. **Mutation components** — Send forms, mint buttons, consume flows
5. **Transaction UX** — Stage progress, error handling, loading states
6. **Polish** — Auto-sync tuning, memoization, edge cases

When stuck at any stage: search the React SDK source for a similar working pattern. Adapt it, don't guess.

---

## Miden Source Repository Map

Clone this repo alongside your project for reference. Claude will explore it when needed for advanced patterns.

```bash
# Contains React SDK source, WebClient WASM bindings, and working examples
git clone --depth 1 https://github.com/0xMiden/miden-client.git ../miden-client
```

### `packages/react-sdk/` — React SDK Source

The primary reference for all frontend development.

- **`src/hooks/`** — All 18+ hook implementations. Each file is self-contained. Read these to understand exact parameters, error handling, and stage progression.
- **`src/context/MidenProvider.tsx`** — Client initialization, sync loop, signer detection, runExclusive lock. Read this to understand initialization order.
- **`src/context/SignerContext.ts`** — External signer interface. Read this when implementing custom signers.
- **`src/store/MidenStore.ts`** — Zustand store structure. Read this to understand cached state and what triggers re-renders.
- **`src/utils/`** — Utility implementations (amounts, notes, bech32, runExclusive, accountParsing).
- **`src/types/index.ts`** — All TypeScript interfaces. The single source of truth for option types, result types, and configuration.
- **`examples/wallet/`** — Complete working wallet app. The most reliable reference for how to set up MidenProvider, create accounts, display balances, claim notes, and send tokens.

**Explore when**: Writing any new component, understanding exact hook behavior, finding how a specific feature works, debugging unexpected behavior.

### `crates/web-client/` — WASM Client Bindings

The Rust-to-WASM bridge that the React SDK wraps.

- Contains the `WebClient` struct and all methods available via `useMidenClient()`
- JavaScript bindings in `js/` directory

**Explore when**: A hook doesn't exist for your operation, understanding what WebClient methods are available, debugging WASM-level errors.

### `crates/idxdb-store/` — IndexedDB Persistence

The browser storage layer for accounts, keys, notes, and transaction history.

**Explore when**: Debugging data persistence issues, understanding what's stored in IndexedDB, investigating storage isolation for external signers.

---

## What to Explore for Each Pattern

| Building This | Explore These Paths | What to Look For |
|---|---|---|
| Basic wallet UI | `examples/wallet/` | MidenProvider setup, useAccounts, useSend |
| Custom transaction | `src/hooks/useTransaction.ts` | Request factory pattern, client methods |
| External signer | `src/context/SignerContext.ts` | SignerContextValue interface, signCb |
| Note consumption flow | `src/hooks/useConsume.ts` | NoteId parsing, filter construction |
| Swap UI | `src/hooks/useSwap.ts` | Swap options, dual note types |
| Token display | `src/utils/amounts.ts` | formatAssetAmount, parseAssetAmount |
| Account ID formatting | `src/utils/accountBech32.ts` | toBech32AccountId |
| State management | `src/store/MidenStore.ts` | Zustand selectors, cached state |
| Direct WebClient usage | `src/context/MidenProvider.tsx` | useMidenClient(), runExclusive |
| Multi-step workflow | `src/hooks/useWaitForCommit.ts`, `useWaitForNotes.ts` | Polling, timeout patterns |

---

## Common Advanced Patterns

### Custom Hooks Wrapping WebClient
For operations not covered by built-in hooks, create custom hooks that use useMidenClient() and runExclusive:
```tsx
function useBlockHeader(blockNumber: number) {
  const client = useMidenClient();
  const { runExclusive } = useMiden();
  const [data, setData] = useState(null);
  useEffect(() => {
    // Note: runExclusive() may be simplified in a future SDK version.
    // Check SDK changelog when upgrading.
    runExclusive(async () => {
      const header = await client.getBlockHeaderByNumber(blockNumber);
      setData(header);
    });
  }, [blockNumber]);
  return data;
}
```

### Multi-Step Workflows
Compose hooks for complex flows (mint → wait for commit → sync → consume):
```tsx
const { mutate: mint } = useMint();
const { mutate: waitForCommit } = useWaitForCommit();
const { mutate: waitForNotes } = useWaitForNotes();
const { mutate: consume } = useConsume();

const mintAndConsume = async () => {
  const { transactionId } = await mint({ targetAccountId, faucetId, amount });
  await waitForCommit({ transactionId });
  await waitForNotes({ accountId: targetAccountId });
  await consume({ accountId: targetAccountId, noteIds: [...] });
};
```

### Custom Signer Implementation
Implement the SignerContextValue interface, wrap MidenProvider in your provider. Reference `src/context/SignerContext.ts` for the exact interface contract. The `storeName` field must be unique per user to ensure IndexedDB isolation.
