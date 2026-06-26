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
- `useAccounts()` — returns `accounts` (e.g. 3 account headers); `wallets` mirrors `accounts` and `faucets` is always `[]` in v0.15 (protocol 0.15 removed faucet-vs-wallet from the account id, so both fields are deprecated). Detect faucet-ness per-account from its components, not from a `faucets` array. The override example above already models this correctly (`wallets: [], faucets: []`).
- `useAccount()` — account with 10.0 TEST token balance
- `useNotes()` — 1 input note, 1 consumable note
- `useSyncState()` — syncHeight: 12345, not syncing
- `useAssetMetadata()` — TEST token metadata (symbol, decimals: 8)
- `useMiden()` — isReady: true

**Mutation hooks** return idle state by default:
- `useSend()` — `{ send: vi.fn(), stage: "idle", isLoading: false }`. Its `result` type is `SendResult { txId, note }` — distinct from `TransactionResult { transactionId }` used by `useMint`/`useConsume`/`useSwap`/`useMultiSend`/`useTransaction`.
- `useMint()`, `useConsume()`, `useSwap()`, `useTransaction()`, `useMultiSend()` — idle shape with `result: TransactionResult | null`.
- `useCreateWallet()` — `{ createWallet: vi.fn(), isCreating: false }`.

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

// Show completed transaction — useSend returns SendResult { txId, note }
vi.mocked(useSend).mockReturnValue({
  send: vi.fn(),
  result: { txId: "0xabc123", note: null },
  isLoading: false,
  stage: "complete",
  error: null,
  reset: vi.fn(),
});

// Other mutation hooks return TransactionResult { transactionId }
vi.mocked(useMint).mockReturnValue({
  mint: vi.fn(),
  result: { transactionId: "0xdef456" },
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
  WALLET_ID_1,           // "0x0a00000000000001"
  WALLET_ID_2,           // "0x0a00000000000002"
  FAUCET_ID,             // "0x0a00000000000003"
  COUNTER_ID,            // "0x0a00000000000004"
  MOCK_WALLET_HEADER,    // { id, nonce, storageCommitment }
  MOCK_FAUCET_HEADER,    // { id, nonce, storageCommitment }
  MOCK_ASSET_BALANCE,    // { assetId, amount: 1000000000n, symbol: "TEST", decimals: 8 }
  MOCK_ACCOUNT,          // { id, nonce, bech32id() }
  MOCK_TRANSACTION_RESULT, // { transactionId: "0x..." } — useMint / useConsume / useSwap / useMultiSend / useTransaction
  MOCK_SEND_RESULT,        // { txId: "0x...", note: null }  — useSend
  MOCK_NOTE_SUMMARY,       // { id, assets, sender }
} from "@/__tests__/fixtures";
```

Key characteristics:
- Account IDs use hex format (`0x...`) — network-agnostic test fixtures
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

## Wallet connection state in tests

The [frontend template](https://github.com/0xMiden/frontend-template)'s wallet button (in `src/components/AppContent.tsx`) drives off **`useMidenFiWallet()`** from `@miden-sdk/miden-wallet-adapter-react`, not the generic `useSigner()`. The button gates on `wallet.readyState` (from `@miden-sdk/miden-wallet-adapter-base`) so the UI can render an "Install MidenFi Wallet" state before the extension is detected, rather than falling through to the adapter's Chrome-Web-Store fallback. When testing wallet-connect UI, mock both modules and override per test.

Setup at the top of the test file:

```tsx
vi.mock("@miden-sdk/react", () => import("@/__tests__/mocks/miden-sdk-react"));
vi.mock("@miden-sdk/miden-wallet-adapter-react", () => ({
  useMidenFiWallet: vi.fn(() => ({
    wallet: null,
    connected: false,
    connecting: false,
    connect: vi.fn(),
    disconnect: vi.fn(),
  })),
}));
vi.mock("@miden-sdk/miden-wallet-adapter-base", () => ({
  WalletReadyState: {
    Installed: "Installed",
    NotDetected: "NotDetected",
    Loadable: "Loadable",
    Unsupported: "Unsupported",
  },
}));

import { useMidenFiWallet } from "@miden-sdk/miden-wallet-adapter-react";
```

Per-test overrides match the states the template renders:

```tsx
// extension not detected — shows disabled "Install MidenFi Wallet"
vi.mocked(useMidenFiWallet).mockReturnValue({
  wallet: { adapter: {} as never, readyState: "NotDetected" } as never,
  connected: false,
  connecting: false,
  connect: vi.fn(),
  disconnect: vi.fn(),
} as never);

// installed + disconnected — shows "Connect Wallet"
vi.mocked(useMidenFiWallet).mockReturnValue({
  wallet: { adapter: {} as never, readyState: "Installed" } as never,
  connected: false,
  connecting: false,
  connect: vi.fn(),
  disconnect: vi.fn(),
} as never);

// connected — shows "Disconnect Wallet"
vi.mocked(useMidenFiWallet).mockReturnValue({
  wallet: { adapter: {} as never, readyState: "Installed" } as never,
  connected: true,
  connecting: false,
  connect: vi.fn(),
  disconnect: vi.fn(),
} as never);
```

See `src/components/__tests__/AppContent.test.tsx` in the [frontend template](https://github.com/0xMiden/frontend-template) for the full pattern (including a `walletState()` helper that cuts per-test boilerplate).

For app code that needs the selected signer account for client-side flows (transaction-building hooks, etc.), `useMiden()` exposes `signerAccountId` / `signerConnected` as lower-level provider state — mock those via the `@miden-sdk/react` mock factory.

Vitest config externalizes `@miden-sdk/miden-wallet-adapter-react` to prevent broken transitive resolution.

## Automated Verification Pipeline

The [frontend template](https://github.com/0xMiden/frontend-template) ships a `.claude/settings.json` that wires Claude Code hooks to enforce quality automatically. All three checks live under a single `PostToolUse` matcher (`Edit|Write`) and fire on every `.ts`/`.tsx` edit in `src/` (the typecheck and affected-tests hooks early-exit otherwise); the template ships no `Stop` hook:

1. **PostToolUse: typecheck** — `npx tsc -b --noEmit` on every `.ts`/`.tsx` edit in `src/`
2. **PostToolUse: affected tests** — `npx vitest --changed --run` on every `.ts`/`.tsx` edit in `src/`
3. **PostToolUse: full verification** — `npx vitest --run && npx tsc -b --noEmit && npx vite build` (same `Edit|Write` matcher), so the full suite + build run on each src edit rather than at task completion

If any hook fails (exit code 2), the agent is blocked from proceeding until the issue is fixed. Copy the same hook layout into your own `.claude/settings.json` to get the same enforcement locally.

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
7. Task complete       → full suite + build runs on each src edit (PostToolUse)
```

## Common Mistakes

**Forgetting vi.clearAllMocks()**: Always call in `beforeEach` to prevent mock state leaking between tests.

**Not mocking the SDK**: Components importing from `@miden-sdk/react` will fail without `vi.mock()` because the real SDK requires WASM initialization.

**Using number instead of bigint for result/fixture amounts**: Result and fixture amounts are typed strictly as `bigint` (`AssetBalance.amount`, `NoteAsset.amount`, and `useAccount().getBalance()`), so mock them with bigint literals (`1000n`, not `1000`). Hook input options (`SendOptions.amount`, `MintOptions.amount`, `MultiSendRecipient.amount`, `CreateFaucetOptions.maxSupply`) accept `bigint | number`, but prefer bigint to avoid precision loss.

**Testing implementation details**: Test what the user sees (text, buttons, states), not internal hook calls. Use `screen.getByRole`, `screen.getByText`, not internal component state.
