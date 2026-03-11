---
name: testing-patterns
description: Testing conventions, mock factory, fixtures, and TDD workflow for Miden frontend development. Covers Vitest + testing-library setup, @miden-sdk/react module mocking, realistic fixture data, test patterns for query and mutation hooks, and the automated verification pipeline. Use when writing, running, or debugging tests for Miden React components.
---

# Miden Frontend Testing Patterns

## Test Stack

- **Vitest** — Test runner (extends Vite config for consistent behavior)
- **@testing-library/react** — Component rendering and queries
- **@testing-library/user-event** — User interaction simulation
- **@testing-library/jest-dom** — DOM assertion matchers (toBeInTheDocument, toBeDisabled, etc.)
- **jsdom** — Browser environment for tests

## Mock Factory: `@miden-sdk/react`

All Miden SDK hooks are mocked via `src/__tests__/mocks/miden-sdk-react.ts`. This module exports mock implementations of every hook with realistic default return values.

### Usage in test files

```tsx
// 1. Mock the entire module (hoisted to top by vitest)
vi.mock("@miden-sdk/react", () => import("@/__tests__/mocks/miden-sdk-react"));

// 2. Import hooks you want to override
import { useAccounts, useSend } from "@miden-sdk/react";

// 3. Override per-test
it("shows empty state", () => {
  vi.mocked(useAccounts).mockReturnValue({
    accounts: [],
    wallets: [],
    faucets: [],
    isLoading: false,
    error: null,
    refetch: vi.fn(),
  });
  render(<MyComponent />);
});
```

### Default mock return values

**Query hooks** return populated data by default:
- `useAccounts()` — 2 wallets, 1 faucet
- `useAccount()` — account with 10.0 TEST token balance
- `useNotes()` — 1 input note, 1 consumable note
- `useSyncState()` — syncHeight: 12345, not syncing
- `useAssetMetadata()` — TEST token metadata (symbol, decimals: 8)
- `useMiden()` — isReady: true

**Mutation hooks** return idle state by default:
- `useSend()` — `{ send: vi.fn(), stage: "idle", isLoading: false }`
- `useMint()`, `useConsume()`, `useSwap()`, `useTransaction()` — similar pattern
- `useCreateWallet()` — `{ createWallet: vi.fn(), isCreating: false }`

### Simulating transaction stages

```tsx
// Show "proving" stage
vi.mocked(useSend).mockReturnValue({
  send: vi.fn(),
  result: null,
  isLoading: true,
  stage: "proving",
  error: null,
  reset: vi.fn(),
});

// Show completed transaction
vi.mocked(useSend).mockReturnValue({
  send: vi.fn(),
  result: { transactionId: "0xabc123" },
  isLoading: false,
  stage: "complete",
  error: null,
  reset: vi.fn(),
});
```

## Fixtures

Realistic test data in `src/__tests__/fixtures/`:

```tsx
import {
  WALLET_ID_1,           // "mtst1qy35qfqdvpjx2e5zf9hkp4vr"
  WALLET_ID_2,           // "mtst1qa7k9qjf8dp4x2e5zf9hkp5vr"
  FAUCET_ID,             // "mtst1qx9y8zjf2dp4x2e5zf9hkp3vr"
  COUNTER_ID,            // "mtst1aru8adnrqspgcsr3drk2n990lyc070ll"
  MOCK_WALLET_HEADER,    // { id, nonce, storageCommitment }
  MOCK_FAUCET_HEADER,    // { id, nonce, storageCommitment }
  MOCK_ASSET_BALANCE,    // { assetId, amount: 1000000000n, symbol: "TEST", decimals: 8 }
  MOCK_ACCOUNT,          // { id, nonce, bech32id() }
  MOCK_TRANSACTION_RESULT, // { transactionId: "0x..." }
  MOCK_NOTE_SUMMARY,    // { id, assets, sender }
} from "@/__tests__/fixtures";
```

Key characteristics:
- Account IDs use bech32 format (`mtst1...`)
- Amounts are `bigint` (e.g., `1000000000n` = 10.0 with 8 decimals)
- Asset metadata uses TEST token with 8 decimals

## Test Patterns (copy-adaptable)

Reference tests in `src/__tests__/patterns/`:

| Pattern | File | Tests |
|---------|------|-------|
| Provider/context setup | `provider-setup.test.tsx` | ready, loading, error states |
| Query hook component | `query-hook.test.tsx` | data, loading, error, empty states |
| Mutation hook component | `mutation-hook.test.tsx` | idle, stages, success, error, argument verification |

### Minimum test coverage per component

Every component test should cover:
1. **Success state** — renders correctly with data
2. **Loading state** — shows loading indicator
3. **Error state** — shows error message, recovery action
4. **User interactions** — buttons, forms trigger correct handler calls

## Mocking the wallet adapter

The app uses `@miden-sdk/miden-wallet-adapter`. Mock it at the module level:

```tsx
vi.mock("@miden-sdk/miden-wallet-adapter", () => ({
  WalletMultiButton: () => <button>Connect Wallet</button>,
  useWallet: vi.fn(() => ({
    address: "mtst1...",
    connected: true,
    requestTransaction: vi.fn(),
  })),
}));
```

Vitest config externalizes `@miden-sdk/miden-wallet-adapter*` sub-packages to prevent broken transitive resolution from the reactui sub-package.

## Automated Verification Pipeline

Hooks in `.claude/settings.json` enforce quality automatically:

1. **PostToolUse: typecheck** — `npx tsc -b --noEmit` on every `.ts`/`.tsx` edit in `src/`
2. **PostToolUse: affected tests** — `npx vitest --changed --run` on every `.ts`/`.tsx` edit in `src/`
3. **Stop hook** — Full `vitest --run && tsc -b --noEmit && vite build` before task completion

If any hook fails (exit code 2), the agent is blocked from proceeding until the issue is fixed.

## TDD Flow

```
1. Write test (describe expected behavior)
   ↓
2. yarn test           → RED (test fails)
   ↓
3. Implement code
   ↓
4. Auto hooks fire     → typecheck + affected tests
   ↓
5. yarn test           → GREEN (all pass)
   ↓
6. Refactor if needed
   ↓
7. Task complete       → Stop hook: full suite + build
```

## Common Mistakes

**Forgetting vi.clearAllMocks()**: Always call in `beforeEach` to prevent mock state leaking between tests.

**Not mocking the SDK**: Components importing from `@miden-sdk/react` will fail without `vi.mock()` because the real SDK requires WASM initialization.

**Using number instead of bigint**: Mock amounts must use `bigint` (`1000n`, not `1000`). The SDK enforces this at the type level.

**Testing implementation details**: Test what the user sees (text, buttons, states), not internal hook calls. Use `screen.getByRole`, `screen.getByText`, not internal component state.
