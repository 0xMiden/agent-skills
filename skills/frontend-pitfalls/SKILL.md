---
name: frontend-pitfalls
description: Critical pitfalls and safety rules for Miden frontend development. Covers WASM initialization, concurrent access crashes, COOP/COEP headers, BigInt handling, Bech32 network mismatches, IndexedDB state loss, auto-sync side effects, Vite configuration, and React rendering race conditions. Use when reviewing, debugging, or writing Miden frontend code.
---

# Miden Frontend Pitfalls

## FP1: WASM Initialization Race (CRITICAL)

Components that use Miden hooks before MidenProvider finishes WASM initialization will crash.

```tsx
// WRONG — crashes if WASM not ready
function App() {
  const { data } = useAccounts(); // throws before init
  return <div>{data?.wallets.length}</div>;
}

// CORRECT — use loadingComponent or check isReady
<MidenProvider
  config={{ rpcUrl: "devnet" }}
  loadingComponent={<p>Loading WASM...</p>}
>
  <App />
</MidenProvider>

// CORRECT — guard with isReady
function App() {
  const { isReady } = useMiden();
  if (!isReady) return <p>Loading...</p>;
  return <WalletView />;
}
```

## FP2: Recursive WASM Access Crash (CRITICAL)

The WASM client is single-threaded. Concurrent calls crash with "recursive use of an object detected".

```tsx
// WRONG — two operations running simultaneously
const handleClick = async () => {
  sync();                    // fires async
  await send({ ... });       // runs concurrently — CRASH
};

// CORRECT — use runExclusive for sequential execution
const { runExclusive } = useMiden();
await runExclusive(async (client) => {
  await client.syncState();
  // now safe to do next operation
});
```

Built-in hooks (useSend, useConsume, etc.) already use runExclusive internally. This pitfall applies when using `useMidenClient()` directly or mixing manual client calls with hook mutations.

## FP3: COOP/COEP Headers Missing (CRITICAL)

WASM SharedArrayBuffer requires these headers. Without them, WASM init silently fails.

```ts
// REQUIRED in vite.config.ts
server: {
  headers: {
    "Cross-Origin-Opener-Policy": "same-origin",
    "Cross-Origin-Embedder-Policy": "require-corp",
  },
},

// ALSO REQUIRED on production server (nginx, Vercel, Cloudflare)
// See vite-wasm-setup skill for deployment configs
```

**Gotcha**: These headers break third-party iframes, external scripts without CORS, and OAuth popups. Use `credentialless` for COEP if cross-origin resources are needed.

**Note**: `midenVitePlugin()` from `@miden-sdk/vite-plugin` handles COOP/COEP automatically via its `crossOriginIsolation` option (defaults to `false` to avoid breaking OAuth popups). Enable it instead of setting headers manually.

## FP4: BigInt Type Mismatch (HIGH)

All token amounts in the SDK are `bigint`. Passing `number` causes TypeScript errors or runtime failures.

```tsx
// WRONG
await send({ from, to, assetId, amount: 1000 });        // number — fails
await createFaucet({ maxSupply: 1000000, ... });          // number — fails

// CORRECT
await send({ from, to, assetId, amount: 1000n });        // bigint literal
await createFaucet({ maxSupply: BigInt(1000000), ... });  // BigInt constructor

// CORRECT — use parseAssetAmount for user input
import { parseAssetAmount } from "@miden-sdk/react";
const amount = parseAssetAmount(inputValue, 8);           // string → bigint
```

**Gotcha**: `JSON.stringify` cannot serialize `bigint`. Use a custom replacer or convert to string first.

## FP5: Bech32 Network Mismatch (HIGH)

Bech32-encoded account IDs include the network. A devnet address on testnet points to a different or nonexistent account.

```tsx
// WRONG — hardcoding a bech32 address used across networks
const ADMIN = "miden1qy35..."; // this is network-specific!

// CORRECT — use hex format for cross-network compatibility
const ADMIN = "0x1234567890abcdef";

// CORRECT — derive bech32 per network
account.bech32id(); // returns correct bech32 for current network
```

Both hex and bech32 formats work in all hooks. Prefer hex for constants, bech32 for display.

## FP6: Auto-Sync Side Effects (MEDIUM)

Default `autoSyncInterval` is 15000ms (15 seconds). Each sync triggers re-renders in useAccounts, useAccount, useNotes, etc.

```tsx
// PROBLEM — form resets every 15 seconds because parent re-renders
<MidenProvider config={{ rpcUrl: "devnet" }}>
  <SendForm />  {/* re-renders on every sync */}
</MidenProvider>

// SOLUTION 1 — preferred: use stable keys and memoization
const MemoizedForm = React.memo(SendForm);

// SOLUTION 2 — disable auto-sync for manual control
<MidenProvider config={{ rpcUrl: "devnet", autoSyncInterval: 0 }}>
```

## FP7: IndexedDB State Loss (MEDIUM)

The client persists accounts, keys, and notes in IndexedDB. Browser "Clear site data", private browsing, or storage pressure can delete everything.

- Warn users that clearing browser data deletes their wallet
- Consider external signers (Para, Turnkey) for production — keys are server-side
- Implement account export/backup for local keystore users

## FP8: Vite Configuration Requirements (MEDIUM)

The `@miden-sdk/vite-plugin` package handles all Miden-specific Vite config.
The minimal setup is:

```ts
import { midenVitePlugin } from "@miden-sdk/vite-plugin";

export default defineConfig({
  plugins: [react(), midenVitePlugin()],
});
```

`midenVitePlugin()` handles: WASM loading, top-level await, WASM pre-bundling exclusion,
and optionally COOP/COEP headers (via the `crossOriginIsolation` option, defaults to `false`
to avoid breaking OAuth popups).

| Option | Default | Purpose |
|--------|---------|---------|
| `crossOriginIsolation` | `false` | Add COOP/COEP headers for SharedArrayBuffer |

Enable `crossOriginIsolation: true` only if your app doesn't use OAuth or cross-origin iframes.
For production COOP/COEP, set headers at the server level (see vite-wasm-setup skill).

## FP9: React StrictMode Double-Init (LOW)

React 19 StrictMode double-invokes effects in development. MidenProvider handles this via `isInitializedRef`, but direct `WebClient.createClient()` calls will initialize twice.

```tsx
// WRONG — manual client creation in useEffect
useEffect(() => {
  const client = await WebClient.createClient(url); // called twice in dev
}, []);

// CORRECT — always use MidenProvider
<MidenProvider config={{ rpcUrl: "devnet" }}>
```

## Quick Reference

| # | Pitfall | Severity | One-Line Rule |
|---|---------|----------|---------------|
| FP1 | WASM init race | CRITICAL | Use loadingComponent or check isReady |
| FP2 | Recursive WASM | CRITICAL | Use runExclusive() for all direct client access |
| FP3 | COOP/COEP | CRITICAL | Add headers in vite.config.ts AND production server |
| FP4 | BigInt | HIGH | All amounts are bigint (1000n not 1000) |
| FP5 | Bech32 mismatch | HIGH | Match network in rpcUrl and addresses |
| FP6 | Auto-sync | MEDIUM | Set autoSyncInterval: 0 if UI stability matters |
| FP7 | IndexedDB loss | MEDIUM | Warn users; use external signers for production |
| FP8 | Vite config | MEDIUM | Use `midenVitePlugin()` — it handles all Miden Vite config |
| FP9 | StrictMode | LOW | Use MidenProvider, not manual client creation |
