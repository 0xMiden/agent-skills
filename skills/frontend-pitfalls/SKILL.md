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

## FP3: COOP/COEP Headers — Only for the Multi-Threaded (MT) Build (HIGH)

COOP/COEP cross-origin-isolation is **not** a universal requirement. The web SDK ships four entry points along two axes (eager/lazy × ST/MT), and the isolation requirement depends entirely on the threading model:

- The **default** `@miden-sdk/react` (and `@miden-sdk/react/lazy`) and the **default** `@miden-sdk/miden-sdk` (and `/lazy`) are **single-threaded (ST)**. They ship single-threaded WASM that "loads in any browser context" with **no COOP/COEP requirement**. This is why the SDK's shipped example wallet runs full Miden client code (`MidenProvider`) importing the default `@miden-sdk/react` while using the bare `midenVitePlugin()` with no cross-origin isolation — ST simply does not need it.
- Only the **multi-threaded (MT)** variants — `@miden-sdk/react/mt`, `@miden-sdk/react/mt/lazy`, `@miden-sdk/miden-sdk/mt`, `@miden-sdk/miden-sdk/mt/lazy` (wasm-bindgen-rayon, ~3–5× faster local proving) — **require** the page to be cross-origin-isolated (`self.crossOriginIsolated === true`). Without `Cross-Origin-Opener-Policy: same-origin` + `Cross-Origin-Embedder-Policy: require-corp`, the browser refuses to construct `WebAssembly.Memory({ shared: true })` and the MT WASM fails to instantiate at module load.

So: pick ST (the default) and you need no headers at all; opt into MT only if you do local proving on a host whose headers you control.

If you DO opt into the MT build, enable isolation via the Vite plugin explicitly on any route that runs the MT client:

```ts
// in your app's vite.config.ts — only needed for the MT build
import { midenVitePlugin } from "@miden-sdk/vite-plugin";

export default defineConfig({
  plugins: [react(), midenVitePlugin({ crossOriginIsolation: true })],
});
```

Do not rely on the plugin's own default — `@miden-sdk/vite-plugin` defaults `crossOriginIsolation` to `false` (verified false in the executable source across the released tags; note the plugin README incorrectly says the default is `true`). For MT you must pass `true` explicitly. For ST (the default build) leaving it `false` is correct — the example wallet uses bare `midenVitePlugin()` precisely because it is ST, and because `same-origin` COOP would nullify `window.opener` in the Para OAuth popups it pairs with via `paraVitePlugin()`.

For MT, COOP/COEP must also be set on the production server — the plugin covers only the Vite dev and preview servers, not your real production host. See `vite-wasm-setup` for per-host configs (Nginx, Vercel, Cloudflare).

**Gotcha (when isolation is on)**: Cross-origin-isolation breaks third-party iframes, external scripts without CORS, and OAuth popups. If a route must host those and cannot satisfy isolation, stay on the default ST subpaths (they need no isolation) or, if you genuinely need MT elsewhere, use `Cross-Origin-Embedder-Policy: credentialless` for weaker isolation that still allows most cross-origin resources, or scope the headers to only the MT routes. Do not enable isolation globally as a convenience.

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

The `@miden-sdk/vite-plugin` package handles all Miden-specific Vite config. The recommended pattern for any new Miden app is:

```ts
import { midenVitePlugin } from "@miden-sdk/vite-plugin";

export default defineConfig({
  // ST (default build): bare plugin is enough — no isolation needed
  plugins: [react(), midenVitePlugin()],

  // MT build only: opt into cross-origin isolation
  // plugins: [react(), midenVitePlugin({ crossOriginIsolation: true })],
});
```

`midenVitePlugin()` handles WASM loading (esnext build target, top-level await), pre-bundling exclusion (`optimizeDeps.exclude`), package deduplication, a gRPC-web RPC proxy, and — when `crossOriginIsolation: true` is passed — emits the COOP `same-origin` + COEP `require-corp` headers the **MT** build requires for `SharedArrayBuffer` on both the dev `server` and the `preview` server.

| Option | Plugin source default | When to set `true` | Purpose |
|--------|-----------------------|--------------------|---------|
| `crossOriginIsolation` | `false` | Only when importing the MT variants (`/mt`, `/mt/lazy`) | Emit COOP/COEP headers for SharedArrayBuffer |

For the **default single-threaded build**, leave `crossOriginIsolation` at its `false` default — the ST WASM loads in any browser context and needs no headers. Pass `crossOriginIsolation: true` **only** when you opt into the multi-threaded variants for local proving; without the headers the MT WASM can't construct shared memory and fails to instantiate. (The plugin README at v0.15.0 incorrectly documents the default as `true`; the executable source default is `false`, unchanged across the released tags. Do not trust the README.) The shipped example wallet uses bare `midenVitePlugin()` because it is ST (and because isolation would break the Para OAuth popups it pairs with via `paraVitePlugin()`) — see FP3. For an MT production deployment, set the same COOP/COEP headers at your real production host — the plugin only injects them into the Vite dev and preview servers. See `vite-wasm-setup` for host-specific configs.

## FP9: React StrictMode Double-Init (LOW)

React StrictMode double-invokes effects in development (since React 18; the React SDK's peer dep is `react >= 18.0.0`). MidenProvider guards against this, but direct low-level `createClient()` calls will initialize twice.

Naming: `@miden-sdk/miden-sdk` exposes a high-level `MidenClient` wrapper class (the recommended entry point) and a low-level client re-exported as `WasmWebClient` — an `@internal` export used mainly by integration tests, whose type declaration explicitly says "Use MidenClient instead." (The class is named `WebClient` in source and re-exported under the alias `WasmWebClient`.) The React SDK does its own low-level init by importing that internal client locally as `WebClient` (`import { WasmWebClient as WebClient } from "@miden-sdk/miden-sdk"`). For manual low-level setup you would call `WasmWebClient.createClient(...)`, but prefer `MidenProvider` (or the high-level `MidenClient`) so init is guarded.

```tsx
// WRONG — manual low-level client creation in useEffect
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
| FP3 | COOP/COEP | HIGH | Default ST build needs no headers; required ONLY for the `/mt` build |
| FP4 | BigInt | HIGH | Hooks accept `bigint \| number` and coerce; prefer bigint, required at the raw WASM boundary |
| FP5 | Bech32 mismatch | HIGH | Match network in rpcUrl and addresses |
| FP6 | Auto-sync | MEDIUM | Set autoSyncInterval: 0 if UI stability matters |
| FP7 | IndexedDB loss | MEDIUM | Warn users; use external signers for production |
| FP8 | Vite config | MEDIUM | Bare `midenVitePlugin()` for ST; pass `crossOriginIsolation: true` only for the `/mt` build |
| FP9 | StrictMode | LOW | Use MidenProvider, not manual client creation |
