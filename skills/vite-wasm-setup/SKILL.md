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
  plugins: [react(), midenVitePlugin({ crossOriginIsolation: true })],
});
```

Pass `crossOriginIsolation: true` explicitly. The Miden WASM client uses `SharedArrayBuffer` via Rust atomics, which is only available when the page is cross-origin-isolated (COOP `same-origin` + COEP `require-corp`). Don't rely on the plugin's own default — it has shifted across releases, so the [frontend template](https://github.com/0xMiden/frontend-template)'s `vite.config.ts` is the source-of-truth reference.

If your app must host third-party iframes, OAuth popups, or other cross-origin resources that don't emit `require-corp`, either (a) embed them via `credentialless` COEP as a workaround (see the Gotchas section below), or (b) set `crossOriginIsolation: false` and accept that Miden client operations won't work on that route.

## What midenVitePlugin() Handles

`@miden-sdk/vite-plugin` abstracts Miden-specific Vite configuration. It does **not** register a `.wasm` module loader — Vite's built-in handling does the actual `.wasm` import. What the plugin sets up:

- **WASM dedup / single copy** — `resolve.alias` (exact-match regex on the WASM package), `resolve.dedupe`, and `resolve.preserveSymlinks` force a single resolved copy of `@miden-sdk/miden-sdk` (avoids WASM class-identity issues across symlinked/monorepo setups)
- **optimizeDeps.exclude** — Excludes `@miden-sdk/miden-sdk` from pre-bundling (pre-bundling corrupts the WASM binary)
- **Top-level await** — Sets `build.target: "esnext"`, which enables the top-level `await` the WASM SDK initialization requires
- **ES-module workers** — Sets `worker.format: "es"`, required for the WASM SDK's module workers
- **COOP/COEP headers** — When `crossOriginIsolation: true`, emits `Cross-Origin-Opener-Policy: same-origin` + `Cross-Origin-Embedder-Policy: require-corp` on **both** the Vite dev server and the Vite preview server (see Production Deployment Headers)
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
- **`@miden-sdk/react` and `@miden-sdk/miden-sdk` must match** — they link against the same WASM ABI, so a mixed pair (e.g. one on 0.14, one on 0.15) won't link. Upgrade them together.
- **The `@miden-sdk/vite-plugin` does NOT track the core SDK version.** At v0.15.0 the plugin ships as `0.14.11` (and the example app pins `^0.14.5`) against `@miden-sdk/miden-sdk@0.15.0`; they only realign later in the 0.15 line. Always defer to your app's `package.json` (or the frontend template's) for the authoritative plugin pin — never assume `vite-plugin === miden-sdk`.
- **The wallet adapters live in a separate repo.** `@miden-sdk/miden-wallet-adapter-react` (and its companion `@miden-sdk/miden-wallet-adapter-base`) are published from [`0xMiden/wallet-adapter`](https://github.com/0xMiden/wallet-adapter), not the web-sdk repo, and are versioned independently. Confirm the exact package names and versions against that repo (or your app's `package.json`); the `-react` adapter's `peerDependencies` pin `@miden-sdk/react` at `^<major>.<minor>.x`, so a patch-level gap from the core SDK is expected and fine.
- **Always check your app's `package.json` (or the [frontend template](https://github.com/0xMiden/frontend-template)'s) for the authoritative versions** — this skill intentionally doesn't inline them because they shift across SDK releases.
- When you bump, clean-install: `rm -rf node_modules yarn.lock && yarn install`. Vite's dep optimizer caches resolved SDK paths, and stale caches can surface as `ERR_BLOCKED_BY_RESPONSE` or spurious `Failed to fetch` errors on module workers.

## Production Deployment Headers

COOP/COEP headers must be set on the production server. `midenVitePlugin({ crossOriginIsolation: true })` only emits them on the Vite dev server (`vite`) and the Vite preview server (`vite preview`) — it does not touch your real production host. Configure the headers separately on nginx/Vercel/Cloudflare/etc.

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

These headers break:
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
| "SharedArrayBuffer is not defined" | COOP/COEP headers not reaching the browser | Verify `midenVitePlugin({ crossOriginIsolation: true })` is in plugins; check production server headers separately |
| WASM module not found | SDK not configured correctly | Ensure `midenVitePlugin()` is in plugins array |
| "Top-level await not supported" | Missing plugin setup | Ensure `midenVitePlugin()` is in plugins array |
| WASM init hangs | COEP blocking WASM fetch | Check network tab for blocked requests; verify COOP/COEP headers are present |
| Build succeeds but WASM fails at runtime | Wrong MIME type | Serve .wasm as application/wasm |
| "recursive use of an object" | Concurrent WASM access | Use runExclusive() from useMiden() |
| Double initialization in dev | React StrictMode | Use MidenProvider (handles this internally) |
