---
name: frontend-pitfalls
description: Critical pitfalls and safety rules for Miden frontend development. Covers WASM initialization, concurrent access crashes, COOP/COEP headers, BigInt handling, Bech32 network mismatches, IndexedDB state loss, auto-sync side effects, Vite configuration, and React rendering race conditions. Use when reviewing, debugging, or writing Miden frontend code.
---

# Miden Frontend Pitfalls

## FP1: WASM Initialization Race (CRITICAL)

Components that use Miden hooks before MidenProvider finishes WASM initialization will crash.

```tsx
// WRONG — renders empty before WASM is ready
function App() {
  const { accounts } = useAccounts(); // returns empty arrays before WASM is ready
  return <div>{accounts.length}</div>;
}

// CORRECT — use loadingComponent or check isReady
<MidenProvider
  config={{ rpcUrl: "testnet" }}
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
const client = useMidenClient();
const { runExclusive } = useMiden();
await runExclusive(async () => {
  await client.syncState();
  // now safe to do next operation
});
```

Built-in hooks (useSend, useConsume, etc.) already use runExclusive internally. This pitfall applies when using `useMidenClient()` directly or mixing manual client calls with hook mutations.

## FP3: COOP/COEP Headers Missing (CRITICAL)

WASM SharedArrayBuffer requires these headers. Without them, WASM init silently fails.

Opt into COOP/COEP via the Vite plugin explicitly on any route that runs Miden client code:

```ts
// in your app's vite.config.ts
import { midenVitePlugin } from "@miden-sdk/vite-plugin";

export default defineConfig({
  plugins: [react(), midenVitePlugin({ crossOriginIsolation: true })],
});
```

Do not rely on the plugin's own default — `@miden-sdk/vite-plugin` defaults `crossOriginIsolation` to `false` (verified false in every v0.14–v0.15 tag of the executable source; note the plugin README incorrectly says the default is `true`). Pass `true` explicitly. The SDK's shipped example wallet deliberately omits this argument (`midenVitePlugin()`) because it pairs with `paraVitePlugin()` for Para OAuth, and `same-origin` COOP nullifies `window.opener` in OAuth popups — so isolation must stay OFF there. Your own app, if it runs Miden client code on a route, must opt in.

COOP/COEP must also be set on the production server — the plugin covers the Vite dev and preview servers, not your real production host. See `vite-wasm-setup` for per-host configs (Nginx, Vercel, Cloudflare).

**Gotcha (opt-out path)**: Cross-origin-isolation breaks third-party iframes, external scripts without CORS, and OAuth popups (this is exactly why the example wallet leaves it off). If a route must host those and cannot use Miden client code, either (a) use `Cross-Origin-Embedder-Policy: credentialless` for weaker isolation that still allows most cross-origin resources, or (b) scope `crossOriginIsolation: false` to that specific route and accept that Miden operations won't work there. Do not disable isolation globally as a convenience.

## FP4: BigInt at the Raw WASM Boundary (HIGH)

The React SDK hooks (`useSend`, `useCreateFaucet`, `useMultiSend`, …) accept `bigint | number` for amounts and coerce to `bigint` internally — `SendOptions.amount` and `CreateFaucetOptions.maxSupply` are both typed `bigint | number`, and `useCreateFaucet` calls `BigInt(options.maxSupply)` before forwarding. So `number` does NOT fail at the hook layer. `bigint` is required only at the raw WASM client (`@miden-sdk/miden-sdk`) boundary, where amounts are `bigint` with no coercion.

```tsx
// FINE at the React-SDK hook layer — number is coerced
await send({ from, to, assetId, amount: 1000 });
await createFaucet({ maxSupply: 1000000, ... });

// ALSO FINE — pass bigint directly (preferred; avoids precision loss above 2^53)
await send({ from, to, assetId, amount: 1000n });
await createFaucet({ maxSupply: BigInt(1000000), ... });

// REQUIRED at the raw WASM client boundary — must be bigint
// (the low-level @miden-sdk/miden-sdk client does not coerce number)

// CORRECT — use parseAssetAmount for user input (decimal string → bigint)
import { parseAssetAmount } from "@miden-sdk/react";
const amount = parseAssetAmount(inputValue, 8);           // string → bigint
```

Prefer `bigint` everywhere anyway: a `number` above `2^53` loses precision before it ever reaches the coercion, so large supplies/amounts must be `bigint` or a decimal string parsed via `parseAssetAmount`.

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
<MidenProvider config={{ rpcUrl: "testnet" }}>
  <SendForm />  {/* re-renders on every sync */}
</MidenProvider>

// SOLUTION 1 — preferred: use stable keys and memoization
const MemoizedForm = React.memo(SendForm);

// SOLUTION 2 — disable auto-sync for manual control
<MidenProvider config={{ rpcUrl: "testnet", autoSyncInterval: 0 }}>
```

## FP7: IndexedDB State Loss (MEDIUM)

The client persists accounts, keys, and notes in IndexedDB. Browser "Clear site data", private browsing, or storage pressure can delete everything.

- Warn users that clearing browser data deletes their wallet
- Consider external signers (Para, Turnkey) for production — keys are server-side
- Implement account export/backup for local keystore users

## FP8: Vite Configuration Requirements (MEDIUM)

The `@miden-sdk/vite-plugin` package handles all Miden-specific Vite config. The recommended pattern for any new Miden app that runs client code is:

```ts
import { midenVitePlugin } from "@miden-sdk/vite-plugin";

export default defineConfig({
  plugins: [react(), midenVitePlugin({ crossOriginIsolation: true })],
});
```

`midenVitePlugin()` handles WASM loading (esnext build target, top-level await), pre-bundling exclusion (`optimizeDeps.exclude`), package deduplication, a gRPC-web RPC proxy, and — when `crossOriginIsolation: true` is passed — emits the COOP `same-origin` + COEP `require-corp` headers the SDK requires for `SharedArrayBuffer` on both the dev `server` and the `preview` server.

| Option | Plugin source default | Recommended for a Miden client route | Purpose |
|--------|-----------------------|--------------------------------------|---------|
| `crossOriginIsolation` | `false` | **`true`** | Emit COOP/COEP headers for SharedArrayBuffer |

Always pass `crossOriginIsolation: true` explicitly on routes that run Miden client code. The plugin's `false` default is wrong for such a route, and relying on it risks silent WASM-init failures. (The plugin README at v0.15.0 incorrectly documents the default as `true`; the executable source default is `false`, unchanged across v0.14–v0.15. Do not trust the README.) The shipped example wallet uses the bare `midenVitePlugin()` precisely because it pairs with `paraVitePlugin()` and must keep isolation OFF for Para OAuth — see FP3 for the opt-out path. For production, set the same headers at your real production host — the plugin only injects them into the Vite dev and preview servers. See `vite-wasm-setup` for host-specific configs.

## FP9: React StrictMode Double-Init (LOW)

React StrictMode double-invokes effects in development (since React 18; the React SDK's peer dep is `react >= 18.0.0`). MidenProvider guards against this, but direct `WasmWebClient.createClient()` calls will initialize twice.

Naming: `@miden-sdk/miden-sdk` exports the raw wasm-bindgen client as `WasmWebClient` (an internal export used by integration tests) and a separate higher-level `WebClient` wrapper class. They are two distinct classes. The React SDK does its low-level init via the raw client, importing it locally as `WebClient` (`import { WasmWebClient as WebClient } from "@miden-sdk/miden-sdk"`). For manual low-level setup, create the raw client via `WasmWebClient.createClient(...)`.

```tsx
// WRONG — manual client creation in useEffect
useEffect(() => {
  const client = await WasmWebClient.createClient(url); // called twice in dev
}, []);

// CORRECT — always use MidenProvider
<MidenProvider config={{ rpcUrl: "testnet" }}>
```

## Quick Reference

| # | Pitfall | Severity | One-Line Rule |
|---|---------|----------|---------------|
| FP1 | WASM init race | CRITICAL | Use loadingComponent or check isReady |
| FP2 | Recursive WASM | CRITICAL | Use runExclusive() for all direct client access |
| FP3 | COOP/COEP | CRITICAL | Add headers in your `vite.config.ts` AND production server |
| FP4 | BigInt | HIGH | Hooks accept `bigint \| number` and coerce; prefer bigint, required at the raw WASM boundary |
| FP5 | Bech32 mismatch | HIGH | Match network in rpcUrl and addresses |
| FP6 | Auto-sync | MEDIUM | Set autoSyncInterval: 0 if UI stability matters |
| FP7 | IndexedDB loss | MEDIUM | Warn users; use external signers for production |
| FP8 | Vite config | MEDIUM | Always pass `midenVitePlugin({ crossOriginIsolation: true })` — don't rely on the plugin default |
| FP9 | StrictMode | LOW | Use MidenProvider, not manual client creation |
