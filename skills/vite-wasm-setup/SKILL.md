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

`@miden-sdk/vite-plugin` abstracts Miden-specific Vite configuration:

- **WASM loading** — Configures Vite to correctly import `.wasm` modules
- **Top-level await** — Enables top-level `await` required by the WASM SDK initialization
- **optimizeDeps** — Excludes `@miden-sdk/miden-sdk` from pre-bundling (pre-bundling corrupts the WASM binary)
- **COOP/COEP headers** — Emits `Cross-Origin-Opener-Policy: same-origin` + `Cross-Origin-Embedder-Policy: require-corp` on the dev server when `crossOriginIsolation: true`

You don't need to install or configure `vite-plugin-wasm`, `vite-plugin-top-level-await`, or dexie aliases manually.

## Required Dependencies

Keep all `@miden-sdk/*` runtime packages aligned. The [frontend template](https://github.com/0xMiden/frontend-template)'s `package.json` pins them as an exact-version set; upgrade all four together and re-run the full verification suite (including the wallet-confirmed increment E2E) whenever you bump.

```json
{
  "dependencies": {
    "@miden-sdk/react": "<matches miden-sdk>",
    "@miden-sdk/miden-sdk": "<authoritative version>",
    "@miden-sdk/miden-wallet-adapter-base": "<may lag by a patch>",
    "@miden-sdk/miden-wallet-adapter-react": "<may lag by a patch>"
  },
  "devDependencies": {
    "@miden-sdk/vite-plugin": "<matches miden-sdk>"
  }
}
```

Notes:
- **Always check your app's `package.json` (or the [frontend template](https://github.com/0xMiden/frontend-template)'s) for the authoritative versions** — this skill intentionally doesn't inline them because they shift across SDK releases.
- The wallet adapter packages are versioned separately from the core SDK. Their `peerDependencies` typically allow `^<major>.<minor>.x`, so a patch-level gap between the adapter and the core SDK is expected and fine.
- When you bump, clean-install: `rm -rf node_modules yarn.lock && yarn install`. Vite's dep optimizer caches resolved SDK paths, and stale caches can surface as `ERR_BLOCKED_BY_RESPONSE` or spurious `Failed to fetch` errors on module workers.

## Production Deployment Headers

COOP/COEP headers must be set on the production server. `midenVitePlugin({ crossOriginIsolation: true })` only affects the Vite dev server.

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
