---
name: vite-wasm-setup
description: Guide to configuring Vite for Miden WASM applications. Covers the midenVitePlugin() setup, COOP/COEP headers, production deployment headers, TypeScript compatibility, and troubleshooting common Vite + WASM issues. Use when setting up a new Miden frontend, debugging build or runtime errors related to WASM or Vite configuration, or deploying to production.
---

# Vite + WASM Configuration for Miden

## Required vite.config.ts

```typescript
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";
import { midenVitePlugin } from "@miden-sdk/vite-plugin";

export default defineConfig({
  plugins: [react(), midenVitePlugin()],
});
```

`midenVitePlugin()` accepts an options object:

```typescript
midenVitePlugin({ crossOriginIsolation: true })
// Enables COOP/COEP headers in dev server. Defaults to false to avoid breaking OAuth popups.
```

## What midenVitePlugin() Handles

`@miden-sdk/vite-plugin` abstracts all Miden-specific Vite configuration:

- **WASM loading** — Configures Vite to correctly import `.wasm` modules
- **Top-level await** — Enables top-level `await` required by the WASM SDK initialization
- **optimizeDeps** — Excludes `@miden-sdk/miden-sdk` from pre-bundling (pre-bundling corrupts the WASM binary)
- **COOP/COEP headers** — Optionally adds `Cross-Origin-Opener-Policy` and `Cross-Origin-Embedder-Policy` headers via `crossOriginIsolation` option

You do not need to install or configure `vite-plugin-wasm`, `vite-plugin-top-level-await`, or dexie aliases manually.

## Required Dependencies

```json
{
  "dependencies": {
    "@miden-sdk/react": "^0.13.0",
    "@miden-sdk/miden-sdk": "^0.13.0"
  },
  "devDependencies": {
    "@miden-sdk/vite-plugin": "^0.13.0"
  }
}
```

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
| "SharedArrayBuffer is not defined" | Missing COOP/COEP headers | Use `midenVitePlugin({ crossOriginIsolation: true })` in dev; set headers on production server |
| WASM module not found | SDK not configured correctly | Ensure `midenVitePlugin()` is in plugins array |
| "Top-level await not supported" | Missing plugin setup | Ensure `midenVitePlugin()` is in plugins array |
| WASM init hangs | COEP blocking WASM fetch | Check network tab for blocked requests; enable `crossOriginIsolation` |
| Build succeeds but WASM fails at runtime | Wrong MIME type | Serve .wasm as application/wasm |
| "recursive use of an object" | Concurrent WASM access | Use runExclusive() from useMiden() |
| Double initialization in dev | React StrictMode | Use MidenProvider (handles this internally) |
