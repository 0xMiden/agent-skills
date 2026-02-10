---
name: react-sdk-patterns
description: Enforce conventions for the Miden React SDK (packages/react-sdk). Use when editing hooks, Zustand store, provider, types, or tests in the React SDK — especially for hook return types, store mutations, strict TypeScript patterns, and testing.
---

# Miden React SDK Patterns

## TypeScript Strict Mode

The SDK uses `strict: true` with additional strictness flags. All code must pass `tsc --noEmit`.

### Nullable Types

Always explicitly type nullable values with `| null`, never rely on `undefined` as absence:

```typescript
// State
client: WebClient | null;
initError: Error | null;

// Setter must accept the same nullable type
setClient: (client: WebClient | null) => void;
setInitError: (error: Error | null) => void;
```

### Error Assertion

Always convert unknown caught errors to `Error`:

```typescript
catch (err) {
  setError(err instanceof Error ? err : new Error(String(err)));
}
```

### Input Normalization

Accept multiple input types and normalize with `useMemo`:

```typescript
const accountIdStr = useMemo(() => {
  if (!accountId) return undefined;
  if (typeof accountId === "string") return accountId;
  if (typeof (accountId as AccountId).toString === "function") {
    return (accountId as AccountId).toString();
  }
  return String(accountId);
}, [accountId]);
```

### Exhaustive Switch

Always include a default case in switch statements:

```typescript
function getStorageMode(mode: "private" | "public" | "network") {
  switch (mode) {
    case "private":
      return AccountStorageMode.private();
    case "public":
      return AccountStorageMode.public();
    case "network":
      return AccountStorageMode.network();
    default:
      return AccountStorageMode.private();
  }
}
```

## Zustand Store

### Store Definition

Single interface for all state and actions. Create with `create<T>()()`:

```typescript
interface MidenStoreState {
  // State
  client: WebClient | null;
  isReady: boolean;
  accounts: AccountHeader[];
  accountDetails: Map<string, Account>;
  sync: { syncHeight: number; isSyncing: boolean; lastSyncTime: number | null; error: Error | null };

  // Actions
  setClient: (client: WebClient | null) => void;
  setAccounts: (accounts: AccountHeader[]) => void;
  setAccountDetails: (accountId: string, account: Account) => void;
  setSyncState: (sync: Partial<MidenStoreState["sync"]>) => void;
  reset: () => void;
}

export const useMidenStore = create<MidenStoreState>()((set) => ({
  ...initialState,
  setClient: (client) => set({ client, isReady: true }),
  setSyncState: (sync) => set((state) => ({
    sync: { ...state.sync, ...sync },
  })),
  setAccountDetails: (accountId, account) =>
    set((state) => {
      const newMap = new Map(state.accountDetails);
      newMap.set(accountId, account);
      return { accountDetails: newMap };
    }),
  reset: () => set(initialState),
}));
```

### Store Conventions

- **Partial merge** for nested objects: `{ ...state.sync, ...sync }`
- **Immutable Map updates**: create new Map, set on it, return it
- **Always provide `reset()`** that restores `initialState`
- **Separate loading flags** per async operation: `isLoadingAccounts`, `isLoadingNotes`
- **Selector hooks** for optimized re-renders:

```typescript
export const useClient = () => useMidenStore((state) => state.client);
export const useIsReady = () => useMidenStore((state) => state.isReady);
```

## Hook Patterns

### Query Hook Structure

Query hooks follow a 10-step pattern:

```typescript
export function useAccount(accountId: string | AccountId | undefined): AccountResult {
  // 1. Get dependencies from context
  const { client, isReady, runExclusive } = useMiden();

  // 2. Get store accessors
  const accountDetails = useMidenStore((state) => state.accountDetails);
  const setAccountDetails = useMidenStore((state) => state.setAccountDetails);

  // 3. Local state for async tracking
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState<Error | null>(null);

  // 4. Normalize inputs with useMemo
  const accountIdStr = useMemo(() => { /* ... */ }, [accountId]);

  // 5. Get cached value from store
  const account = accountIdStr ? (accountDetails.get(accountIdStr) ?? null) : null;

  // 6. Define refetch with useCallback
  const refetch = useCallback(async () => {
    if (!client || !isReady || !accountIdStr) return;
    setIsLoading(true);
    setError(null);
    try {
      const result = await runExclusive(() => client.getAccount(accountIdObj));
      setAccountDetails(accountIdStr, result);
    } catch (err) {
      setError(err instanceof Error ? err : new Error(String(err)));
    } finally {
      setIsLoading(false);
    }
  }, [client, isReady, accountIdStr, setAccountDetails]);

  // 7. Initial fetch effect
  useEffect(() => {
    if (isReady && accountIdStr && !account) {
      refetch();
    }
  }, [isReady, accountIdStr, account, refetch]);

  // 8. Sync-triggered refresh
  useEffect(() => {
    if (!isReady || !accountIdStr || !lastSyncTime) return;
    refetch();
  }, [isReady, accountIdStr, lastSyncTime, refetch]);

  // 9. Derived state with useMemo
  const assets = useMemo(() => /* derive */, [account]);

  // 10. Return typed result
  return { account, assets, isLoading, error, refetch };
}
```

### Mutation Hook Structure

Mutation hooks track stage progression:

```typescript
type TransactionStage = "idle" | "executing" | "proving" | "submitting" | "complete";

export function useSend(): UseSendResult {
  const { client, isReady, sync, runExclusive, prover } = useMiden();
  const [result, setResult] = useState<TransactionResult | null>(null);
  const [isLoading, setIsLoading] = useState(false);
  const [stage, setStage] = useState<TransactionStage>("idle");
  const [error, setError] = useState<Error | null>(null);

  const send = useCallback(async (options: SendOptions): Promise<TransactionResult> => {
    if (!client || !isReady) throw new Error("Miden client is not ready");

    setIsLoading(true);
    setStage("executing");
    setError(null);

    try {
      // Execute
      setStage("proving");
      // Prove
      setStage("submitting");
      // Submit
      setStage("complete");
      setResult(txSummary);
      await sync();
      return txSummary;
    } catch (err) {
      setError(err instanceof Error ? err : new Error(String(err)));
      setStage("idle");
      throw err;
    } finally {
      setIsLoading(false);
    }
  }, [client, isReady, prover, runExclusive, sync]);

  const reset = useCallback(() => {
    setResult(null);
    setIsLoading(false);
    setStage("idle");
    setError(null);
  }, []);

  return { send, result, isLoading, stage, error, reset };
}
```

### Hook Naming

| Pattern | Naming | Example |
|---------|--------|---------|
| Query (plural) | `use<Entities>` | `useAccounts`, `useNotes` |
| Query (single) | `use<Entity>` | `useAccount` |
| Mutation | `use<Action>` | `useSend`, `useCreateWallet`, `useSwap` |
| State | `use<StateName>` | `useSyncState` |
| Result type | `Use<HookName>Result` | `UseSendResult`, `UseAccountResult` |

### Hook Return Types

Always define an explicit interface for the return type:

```typescript
export interface UseSendResult {
  send: (options: SendOptions) => Promise<TransactionResult>;
  result: TransactionResult | null;
  isLoading: boolean;
  stage: TransactionStage;
  error: Error | null;
  reset: () => void;
}
```

## Provider Pattern

### MidenProvider Structure

```typescript
export function MidenProvider({
  children,
  config = {},
  loadingComponent,
  errorComponent,
}: MidenProviderProps) {
  // Zustand store access
  const { client, isReady, isInitializing, initError, setClient } = useMidenStore();

  // Refs for non-reactive state
  const syncIntervalRef = useRef<ReturnType<typeof setInterval> | null>(null);
  const isInitializedRef = useRef(false);
  const clientLockRef = useRef(new AsyncLock());

  // Memoized config
  const resolvedConfig = useMemo(() => ({ ...config }), [config]);

  // runExclusive for serialized async ops
  const runExclusive = useCallback(
    async <T,>(fn: () => Promise<T>): Promise<T> =>
      clientLockRef.current.runExclusive(fn),
    []
  );

  // Sync function shared via context
  const sync = useCallback(async () => {
    if (!client || !isReady) return;
    // Prevent concurrent syncs, use runExclusive
  }, [client, isReady, runExclusive]);

  // One-time initialization effect
  useEffect(() => {
    if (isInitializedRef.current) return;
    isInitializedRef.current = true;
    // ... WebClient.createClient() setup
  }, []);

  // Auto-sync interval
  useEffect(() => {
    if (!isReady || !client) return;
    const interval = config.autoSyncInterval ?? DEFAULTS.AUTO_SYNC_INTERVAL;
    if (interval <= 0) return;
    syncIntervalRef.current = setInterval(() => sync(), interval);
    return () => { if (syncIntervalRef.current) clearInterval(syncIntervalRef.current); };
  }, [isReady, client, config.autoSyncInterval, sync]);

  // Render loading/error states
  if (isInitializing && loadingComponent) return <>{loadingComponent}</>;
  if (initError && errorComponent) {
    return <>{typeof errorComponent === "function" ? errorComponent(initError) : errorComponent}</>;
  }

  return <MidenContext.Provider value={contextValue}>{children}</MidenContext.Provider>;
}
```

### Context Consumer Hooks

```typescript
// Main hook — throws if not in Provider
export function useMiden(): MidenContextValue {
  const context = useContext(MidenContext);
  if (!context) throw new Error("useMiden must be used within a MidenProvider");
  return context;
}

// Convenience hook — throws if client not ready
export function useMidenClient(): WebClient {
  const { client, isReady } = useMiden();
  if (!client || !isReady) throw new Error("Miden client is not ready");
  return client;
}
```

### AsyncLock

Use `AsyncLock` to serialize async operations that mutate client state:

```typescript
export class AsyncLock {
  private pending: Promise<void> = Promise.resolve();

  runExclusive<T>(fn: () => Promise<T>): Promise<T> {
    const run = this.pending.then(fn, fn);
    this.pending = run.then(() => undefined, () => undefined);
    return run;
  }
}
```

All client method calls from hooks must go through `runExclusive` to prevent race conditions.

## Export Organization

In `src/index.ts`, export in this order:

1. Context/Provider (core setup)
2. Query Hooks (alphabetical)
3. Mutation Hooks (alphabetical)
4. Types (comprehensive)
5. Re-exported SDK types (convenience)
6. Utilities (select functions)
7. Defaults/Constants

Export result types alongside their hooks:
```typescript
export { useSend } from "./hooks/useSend";
export type { UseSendResult } from "./hooks/useSend";
```

## Testing

### Unit Tests (Vitest)

Reset store before each test:

```typescript
import { describe, it, expect, beforeEach } from "vitest";
import { useMidenStore } from "../../store/MidenStore";

beforeEach(() => {
  useMidenStore.getState().reset();
});
```

Test state directly via `getState()`:

```typescript
it("should set client and mark as ready", () => {
  const mockClient = createMockWebClient();
  useMidenStore.getState().setClient(mockClient as any);
  const state = useMidenStore.getState();
  expect(state.client).toBe(mockClient);
  expect(state.isReady).toBe(true);
});
```

### Mock Factories

Use factory functions with override merge pattern:

```typescript
export const createMockWebClient = (
  overrides: Partial<MockWebClientType> = {}
) => {
  const defaultClient: MockWebClientType = {
    getAccounts: vi.fn().mockResolvedValue([]),
    getAccount: vi.fn().mockResolvedValue(null),
    newWallet: vi.fn().mockResolvedValue(createMockAccount()),
  };
  return { ...defaultClient, ...overrides };
};
```

Rules:
- Use `vi.fn()` for all method mocks
- Use `mockResolvedValue()` for async methods
- Provide `createMock*` factories for each SDK type
- Support overrides via spread: `{ ...defaults, ...overrides }`

### Integration Tests (Playwright)

Use `page.evaluate()` to execute in browser context:

```typescript
test("hook executes transaction", async ({ page }) => {
  await page.goto("http://localhost:8081/react-hooks.html");

  const result = await page.evaluate(async () => {
    const api = (window as any).__reactSdk;
    return await api.runTransaction();
  });

  expect(result.transactionId).toBeTruthy();
});
```

### Verification Checklist

After any react-sdk change, always run:
```bash
cd packages/react-sdk
yarn typecheck    # tsc --noEmit
yarn lint         # eslint
yarn test         # vitest
yarn build        # tsup + DTS generation
```
