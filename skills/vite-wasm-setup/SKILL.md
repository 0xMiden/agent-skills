---
name: vite-wasm-setup
description: Guide to configuring Vite for Miden WASM applications. Covers the midenVitePlugin() setup, COOP/COEP headers, production deployment headers, TypeScript compatibility, and troubleshooting common Vite + WASM issues. Use when setting up a new Miden frontend, debugging build or runtime errors related to WASM or Vite configuration, or deploying to production.
---

# Vite + WASM Configuration for Miden

## Required `vite.config.ts`

```typescript
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";
import { midenVitePlugin } from "@miden-sdk/vite-plugin";

export default defineConfig({
  plugins: [react(), midenVitePlugin()],
});
```

`midenVitePlugin()` works with no options for the common case — the default `@miden-sdk/miden-sdk` / `@miden-sdk/react` imports ship **single-threaded (ST)** WASM that loads in any browser context, so the default client runs with no cross-origin isolation. The plugin's `crossOriginIsolation` option defaults to `false` for the same reason, and the v0.15.0 example wallet app calls `midenVitePlugin()` bare. Don't reach for `crossOriginIsolation: true` unless you have actually opted into the multi-threaded build (see below).

Pass `crossOriginIsolation: true` **only** if you import the **multi-threaded (MT)** WASM variant — `@miden-sdk/miden-sdk/mt` (or `/mt/lazy`) and `@miden-sdk/react/mt` (or `/mt/lazy`). The MT build uses `wasm-bindgen-rayon` and `SharedArrayBuffer` / `WebAssembly.Memory({ shared: true })` for ~3–5x faster local proving, which the browser only constructs when the page is cross-origin-isolated (COOP `same-origin` + COEP `require-corp`). On the default ST imports those headers are unnecessary. The [frontend template](https://github.com/0xMiden/frontend-template)'s `vite.config.ts` is the source-of-truth reference for the current setup.

If your app must host third-party iframes, OAuth popups, or other cross-origin resources that don't emit `require-corp`, stay on the default ST imports and leave `crossOriginIsolation: false` (the default) — you keep a fully working Miden client and only forgo MT-accelerated local proving on that route. Enabling `crossOriginIsolation: true` also breaks OAuth-popup flows (e.g. Para), because `same-origin` COOP nullifies `window.opener` in popups. If you genuinely need both MT proving and cross-origin resources, embed the latter via `credentialless` COEP as a workaround (see the Gotchas section below).

## What midenVitePlugin() Handles

`@miden-sdk/vite-plugin` abstracts Miden-specific Vite configuration. It does **not** register a `.wasm` module loader — Vite's built-in handling does the actual `.wasm` import. What the plugin sets up:

- **WASM dedup / single copy** — `resolve.alias` (exact-match regex on the WASM package), `resolve.dedupe`, and `resolve.preserveSymlinks` force a single resolved copy of `@miden-sdk/miden-sdk` (avoids WASM class-identity issues across symlinked/monorepo setups)
- **optimizeDeps.exclude** — Excludes `@miden-sdk/miden-sdk` from pre-bundling (pre-bundling corrupts the WASM binary)
- **Top-level await** — Sets `build.target: "esnext"`, which enables the top-level `await` the WASM SDK initialization requires
- **ES-module workers** — Sets `worker.format: "es"`, required for the WASM SDK's module workers
- **COOP/COEP headers (opt-in, MT only)** — `crossOriginIsolation` defaults to `false`. When set to `true`, emits `Cross-Origin-Opener-Policy: same-origin` + `Cross-Origin-Embedder-Policy: require-corp` on **both** the Vite dev server and the Vite preview server (see Production Deployment Headers). Only needed to satisfy the cross-origin-isolation requirement of the MT WASM variant; the default ST build doesn't need these headers
- **gRPC-web dev proxy** — Proxies `/rpc.Api` to `rpcProxyTarget` (default `https://rpc.testnet.miden.io`) during `vite` (serve) to bypass CORS in dev; set `rpcProxyTarget: false` to disable
- **React context dedup** — Externalizes `@miden-sdk/react` during esbuild pre-bundling so signer-provider React contexts share one identity

You don't need to install or configure `vite-plugin-wasm`, `vite-plugin-top-level-await`, or dexie aliases manually.

## Required Dependencies

Two packages move together as the core SDK pair: `@miden-sdk/miden-sdk` (the WASM client) and `@miden-sdk/react` (the React hooks). At v0.15.0 both are `0.15.0` and share a WASM ABI, so they must match. The **vite-plugin and the wallet adapters are versioned independently** and can trail the core SDK by a minor/patch — don't assume they're in lockstep. The [frontend template](https://github.com/0xMiden/frontend-template)'s `package.json` is the reference for the current pin set; re-run your app's full build + end-to-end suite whenever you bump.

```json
{
  "dependencies": {
    "@miden-sdk/react": "<matches @miden-sdk/miden-sdk>",
    "@miden-sdk/miden-sdk": "<authoritative core version>",
    "@miden-sdk/miden-wallet-adapter-react": "<independent — see 0xMiden/wallet-adapter repo>"
  },
  "devDependencies": {
    "@miden-sdk/vite-plugin": "<may lag the core SDK by a minor/patch — see your package.json>"
  }
}
```

Notes:
- **`@miden-sdk/react` and `@miden-sdk/miden-sdk` must match** — they link against the same WASM ABI, so a mixed pair (e.g. one built against an older WASM ABI, one against the current) won't link. Upgrade them together.
- **The `@miden-sdk/vite-plugin` does NOT track the core SDK version.** At v0.15.0 the plugin trails the core SDK by a minor and is NOT on the same version as `@miden-sdk/miden-sdk@0.15.0`; they only realign later in the 0.15 line. Always defer to your app's `package.json` (or the frontend template's) for the authoritative plugin pin — never assume `vite-plugin === miden-sdk`.
- **The wallet adapters live in a separate repo.** `@miden-sdk/miden-wallet-adapter-react` (and its companion `@miden-sdk/miden-wallet-adapter-base`) are published from [`0xMiden/wallet-adapter`](https://github.com/0xMiden/wallet-adapter), not the web-sdk repo, and are versioned independently. Confirm the exact package names and versions against that repo (or your app's `package.json`); the `-react` adapter's `peerDependencies` pin `@miden-sdk/react` at `^<major>.<minor>.x`, so a patch-level gap from the core SDK is expected and fine.
- **Always check your app's `package.json` (or the [frontend template](https://github.com/0xMiden/frontend-template)'s) for the authoritative versions** — this skill intentionally doesn't inline them because they shift across SDK releases.
- When you bump, do a clean install with your project's package manager: delete `node_modules` and the lockfile it actually uses, then reinstall. The web-sdk uses pnpm (`rm -rf node_modules pnpm-lock.yaml && pnpm install`). For an app repo, use whatever package manager its lockfile implies — e.g. the frontend template's v0.15 branch ships a `yarn.lock` (`rm -rf node_modules && yarn install`), while another app may use `npm ci` or `pnpm install`. Vite's dep optimizer caches resolved SDK paths, and stale caches can surface as `ERR_BLOCKED_BY_RESPONSE` or spurious `Failed to fetch` errors on module workers.

## Production Deployment Headers

These headers apply **only if you ship the MT WASM variant** (`/mt` or `/mt/lazy`). The default ST build needs none of this — skip the whole section if you're on the default imports. If you are on MT, the COOP/COEP headers must be set on the production server: `midenVitePlugin({ crossOriginIsolation: true })` only emits them on the Vite dev server (`vite`) and the Vite preview server (`vite preview`) — it does not touch your real production host. Configure the headers separately on nginx/Vercel/Cloudflare/etc.

### Nginx
```nginx
add_header Cross-Origin-Opener-Policy same-origin;
add_header Cross-Origin-Embedder-Policy require-corp;
```

### Vercel (vercel.json)
```json
{
  "headers": [
    {
      "source": "/(.*)",
      "headers": [
        { "key": "Cross-Origin-Opener-Policy", "value": "same-origin" },
        { "key": "Cross-Origin-Embedder-Policy", "value": "require-corp" }
      ]
    }
  ]
}
```

### Cloudflare Pages (_headers)
```
/*
  Cross-Origin-Opener-Policy: same-origin
  Cross-Origin-Embedder-Policy: require-corp
```

### WASM MIME Type
Ensure your server serves `.wasm` files with `application/wasm` MIME type.

## COOP/COEP Gotchas

These gotchas only apply once you've enabled cross-origin isolation for the MT build — the default ST build sets no such headers and is unaffected. When COOP `same-origin` + COEP `require-corp` are in force, they break:
- **Third-party iframes** (YouTube embeds, Twitter embeds, analytics)
- **External scripts** without CORS headers
- **OAuth popups** from different origins

Workaround: Use `credentialless` for COEP if you need cross-origin resources:
```
Cross-Origin-Embedder-Policy: credentialless
```

Note: `credentialless` provides weaker isolation but allows most cross-origin resources.

## TypeScript Compatibility

Standard Vite-compatible tsconfig settings work with Miden. The only actual constraint is ES2020+ for `bigint` support:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "moduleResolution": "bundler"
  }
}
```

`module: "ESNext"` and `moduleResolution: "bundler"` are standard Vite defaults, not Miden-specific requirements. If you're using the Vite-generated tsconfig, no changes are needed beyond ensuring `target` is ES2020+.

## Troubleshooting

| Issue | Cause | Fix |
|-------|-------|-----|
| "SharedArrayBuffer is not defined" (MT build only) | Importing `/mt` or `/mt/lazy` on a page that isn't cross-origin-isolated | Set `midenVitePlugin({ crossOriginIsolation: true })` and add the COOP/COEP headers on your production host; or switch back to the default ST imports, which don't need them |
| WASM module not found | SDK not configured correctly | Ensure `midenVitePlugin()` is in plugins array |
| "Top-level await not supported" | Missing plugin setup | Ensure `midenVitePlugin()` is in plugins array |
| WASM init hangs | COEP blocking WASM fetch | Check network tab for blocked requests; verify COOP/COEP headers are present |
| Build succeeds but WASM fails at runtime | Wrong MIME type | Serve .wasm as application/wasm |
| "recursive use of an object" | Concurrent WASM access | Use runExclusive() from useMiden() |
| Double initialization in dev | React StrictMode | Use MidenProvider (handles this internally) |
