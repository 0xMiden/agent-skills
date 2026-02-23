---
name: vite-wasm-setup
description: Guide to configuring Vite for Miden WASM applications. Covers required plugins (wasm, top-level-await), COOP/COEP headers, dexie alias resolution, optimizeDeps configuration, production deployment headers, and troubleshooting common Vite + WASM issues. Use when setting up a new Miden frontend, debugging build or runtime errors related to WASM or Vite configuration, or deploying to production.
---

# Vite + WASM Configuration for Miden

## Required vite.config.ts

```typescript
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";
import wasm from "vite-plugin-wasm";
import topLevelAwait from "vite-plugin-top-level-await";
import path from "path";

export default defineConfig({
  plugins: [react(), wasm(), topLevelAwait()],
  resolve: {
    alias: {
      dexie: path.resolve(__dirname, "node_modules/dexie"),
    },
  },
  optimizeDeps: {
    exclude: ["@miden-sdk/miden-sdk"],
    include: ["dexie"],
  },
  server: {
    headers: {
      "Cross-Origin-Opener-Policy": "same-origin",
      "Cross-Origin-Embedder-Policy": "require-corp",
    },
  },
});
```

## Why Each Config Is Required

### vite-plugin-wasm
Enables `.wasm` file imports in Vite. Without it, WASM modules fail to load at runtime.

### vite-plugin-top-level-await
Allows top-level `await` in modules. The Miden SDK uses top-level await for WASM initialization. Without this plugin, builds fail with "Top-level await is not available".

### dexie alias
The Miden SDK uses dexie (IndexedDB wrapper) internally. Without the alias, linked or monorepo packages may resolve to a different dexie instance, causing "dexie is not a constructor" errors.

### optimizeDeps.exclude @miden-sdk/miden-sdk
The WASM SDK must NOT be pre-bundled by Vite's dependency optimizer. Pre-bundling corrupts the WASM binary, causing silent initialization failures.

### optimizeDeps.include dexie
Conversely, dexie SHOULD be pre-bundled to avoid module resolution issues in the browser.

### COOP/COEP Headers
SharedArrayBuffer (used by WASM threads) requires Cross-Origin-Opener-Policy and Cross-Origin-Embedder-Policy headers. Without them, the browser blocks SharedArrayBuffer and WASM fails silently.

## Required Dependencies

```json
{
  "dependencies": {
    "@miden-sdk/react": "^0.13.0",
    "@miden-sdk/miden-sdk": "^0.13.0",
    "dexie": "^4.2.1"
  },
  "devDependencies": {
    "vite-plugin-wasm": "^3.5.0",
    "vite-plugin-top-level-await": "^1.6.0"
  }
}
```

## Production Deployment Headers

COOP/COEP headers must also be set on the production server, not just Vite dev server.

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

## TypeScript Configuration

Required tsconfig settings for Miden WASM:
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

ES2022+ is required for BigInt support and top-level await.

## Troubleshooting

| Issue | Cause | Fix |
|-------|-------|-----|
| "SharedArrayBuffer is not defined" | Missing COOP/COEP headers | Add headers to vite.config.ts server block |
| "dexie is not a constructor" | Missing dexie alias | Add resolve.alias in vite.config.ts |
| WASM module not found | SDK pre-bundled by Vite | Add `@miden-sdk/miden-sdk` to optimizeDeps.exclude |
| "Top-level await not supported" | Missing plugin | Add vite-plugin-top-level-await to plugins |
| Build succeeds but WASM fails at runtime | Wrong MIME type | Serve .wasm as application/wasm |
| "recursive use of an object" | Concurrent WASM access | Use runExclusive() from useMiden() |
| WASM init hangs | COEP blocking WASM fetch | Check network tab for blocked requests |
| Double initialization in dev | React StrictMode | Use MidenProvider (handles this internally) |
