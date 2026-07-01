---
name: signer-integration
description: Guide to integrating external signers (Para, Turnkey, MidenFi wallet adapter) and building custom signers for Miden React frontends. Covers provider setup, passkey authentication, unified signer interface, custom SignerContext implementation, and custom account components. Use when adding wallet connection, authentication, or external key management to a Miden frontend.
---

# Miden Signer Integration

## Overview

By default, MidenProvider uses a **local keystore** (keys in IndexedDB, no wallet connection needed). For production apps, wrap MidenProvider with a signer provider to use external key management.

Signer providers must wrap MidenProvider (outer → inner):
```
<SignerProvider>      ← manages keys + auth
  <MidenProvider>     ← manages Miden client
    <App />
  </MidenProvider>
</SignerProvider>
```

## Pre-Built Signer Providers

### Para (EVM Wallets)
```tsx
import { ParaSignerProvider, useParaSigner } from "@miden-sdk/use-miden-para-react";

<ParaSignerProvider apiKey="your-api-key" environment="PRODUCTION">
  <MidenProvider config={{ rpcUrl: "testnet" }}>
    <App />
  </MidenProvider>
</ParaSignerProvider>

const { para, wallet, isConnected } = useParaSigner();
```

### Turnkey (Passkey Authentication)
```tsx
import { TurnkeySignerProvider } from "@miden-sdk/miden-turnkey-react";

// `config` is REQUIRED, and `defaultOrganizationId` is required within it.
// Type: Pick<TurnkeySDKBrowserConfig, "defaultOrganizationId">
//       & Partial<Omit<TurnkeySDKBrowserConfig, "defaultOrganizationId">>
// — only the other fields (e.g. `apiBaseUrl`) are optional; `apiBaseUrl`
// defaults to https://api.turnkey.com. There is NO env-var fallback for the
// org id (the provider does not read `VITE_TURNKEY_ORG_ID`).
<TurnkeySignerProvider config={{ defaultOrganizationId: "your-org-id" }}>
  <MidenProvider config={{ rpcUrl: "testnet" }}>
    <App />
  </MidenProvider>
</TurnkeySignerProvider>

// Or override the apiBaseUrl default:
<TurnkeySignerProvider config={{
  apiBaseUrl: "https://api.turnkey.com",
  defaultOrganizationId: "your-org-id",
}}>
  ...
</TurnkeySignerProvider>
```

`TurnkeySignerProvider` also accepts optional `customComponents` and `importAccountId` props, which it forwards into `accountConfig` (see "Custom Account Components").

Connect via passkey:
```tsx
import { useSigner } from "@miden-sdk/react";
import { useTurnkeySigner } from "@miden-sdk/miden-turnkey-react";

// useSigner() returns null in local-keystore mode (no signer provider mounted),
// so guard before destructuring.
const signer = useSigner();
if (!signer) return null;
const { isConnected, connect, disconnect } = signer;
await connect();  // triggers passkey flow, auto-selects account

// Turnkey-specific extras
const { client, account, setAccount } = useTurnkeySigner();
```

### MidenFi Wallet Adapter (Browser Extension)
```tsx
import { MidenFiSignerProvider } from "@miden-sdk/miden-wallet-adapter-react";
import { WalletAdapterNetwork } from "@miden-sdk/miden-wallet-adapter-base";

<MidenFiSignerProvider
  appName="My App"                                        // optional: passed to MidenWalletAdapter
  network={WalletAdapterNetwork.Testnet}                  // WalletAdapterNetwork enum: Devnet | Testnet | Localnet
  autoConnect                                             // reconnect on mount. Default: false
  storageMode="public"                                    // "private" | "public". Default: "public"
  customComponents={[myComponent]}                        // optional: custom AccountComponents
  privateDataPermission={permission}                      // optional: private data access level
  allowedPrivateData={allowedData}                        // optional: allowed private data types
>
  <MidenProvider config={{ rpcUrl: "testnet" }}>
    <App />
  </MidenProvider>
</MidenFiSignerProvider>
```

With `MidenFiSignerProvider` in place, use `useSigner()` from the React SDK to manage connection state. The regular React SDK hooks (`useSend`, `useConsume`, etc.) automatically sign via the connected wallet — no additional wiring needed.

> The provider accepts an `accountType` prop, but it is a no-op: account visibility is determined solely by `storageMode` (`private`/`public`), and the provider always imports the account by ID (`importAccountId`), bypassing the builder path entirely. Omit it.

### Frontend-template-specific MidenFi pattern

The [frontend template](https://github.com/0xMiden/frontend-template) (on web-sdk 0.15 — `@miden-sdk/miden-sdk@0.15.3`, `@miden-sdk/react@0.15.3`, wallet adapters `0.15.1`) deviates from the generic patterns above in three places worth knowing when the wallet extension is the primary signer:

- **Provider order is INVERTED: `MidenProvider` runs OUTSIDE `MidenFiSignerProvider`** — see `src/providers.tsx`. This is the opposite of the canonical signer-outer / Miden-inner nesting at the top of this skill, and it is deliberate. In v0.15, when a signer provider is an *ancestor* of `MidenProvider`, `MidenProvider` treats it as its external keystore and does NOT create the `WebClient` until the signer connects (the init effect sees `signerIsConnected === false` and returns early before building the client). With a wallet that hasn't connected — or any environment without the extension — the app would hang on "Initializing…" and even public reads couldn't run. The template never signs *through* `MidenProvider` (its only write goes via the wallet's `requestTransaction`), so it runs `MidenProvider` in local-keystore mode (no signer ancestor → it initializes immediately, reads work pre-connect) and keeps `MidenFiSignerProvider` *inside*, purely for the connect button and the wallet's `requestTransaction`. `MidenFiSignerProvider` works standalone (it provides its own `WalletContext` + `SignerContext`; no `MultiSignerProvider` needed). Use this inversion only when you do not sign through `MidenProvider`; if external-keystore signing IS the goal, keep the canonical signer-outer order so `MidenProvider` picks up the signer's `signCb`/`accountConfig`.
- **Wallet button uses `useMidenFiWallet()` + `WalletReadyState`** — see `src/components/AppContent.tsx`. The button gates on `wallet?.readyState` (rendering a disabled "Install MidenFi Wallet" state unless `readyState` is `Installed` or `Loadable`) so it can show install state before the extension is detected. `useSigner().connect()` would silently fall through to the adapter's `window.open(adapter.url, ...)` install fallback; gating on `readyState` avoids that path.
- **Custom transaction flow calls `requestTransaction(...)` directly** — see `src/hooks/useIncrementCounter.ts`. It reads `address` / `connected` / `requestTransaction` off `useMidenFiWallet()`, builds a bespoke request (`TransactionRequestBuilder` with a custom `Note`), wraps it in `Transaction.createCustomTransaction(...)`, and hands it to the wallet for signing + submission. The React SDK mutation hooks (`useSend`, `useConsume`, ...) don't cover this custom note construction, and the tx is submitted by the wallet rather than the local client — so `useWaitForCommit` doesn't apply (the template polls the counter's storage map instead).
  - **The note APIs in that hook (use as the reference):** the JS `NoteMetadata` constructor is attachment-less — `new NoteMetadata(sender, noteType, tag)`. Build a note with `new Note(assets, metadata, recipient)`, and build attachments with `NoteAttachment.fromWord(scheme, word)` / `NoteAttachment.fromWords(scheme, words)` (read back via `.toWords()`), or `createNoteAttachment(...)` from `@miden-sdk/react`.
  - **Heads up: the template ships this write path DISABLED on v0.15** (`INCREMENT_ONCHAIN_BLOCKED = true` in `src/config.ts`). A network-executed custom-script note needs a `NetworkAccountTarget` attachment, and while v0.15 can *build* that attachment, the web SDK exposes no way to *attach* one to a custom-script note — only `Note.createP2IDNote`/`createP2IDENote` accept an attachment (and those force the P2ID script). So the template surfaces the blocker instead of submitting a doomed, fee-bearing tx. Treat the increment as the correct v0.15 *construction* reference, but note the network-targeting step is pending an upstream web-SDK entry point.

## Unified Signer Interface

Works with any signer provider above. `useSigner()` returns `null` in local-keystore mode (no signer provider mounted), so guard before destructuring:
```tsx
import { useSigner } from "@miden-sdk/react";

const signer = useSigner();
if (!signer) return null; // local keystore mode — no external signer

const { isConnected, connect, disconnect, name } = signer;

if (!isConnected) {
  return <button onClick={connect}>Connect {name}</button>;
}
```

## Building a Custom Signer

Implement `SignerContextValue` via `SignerContext.Provider`:

```tsx
import { SignerContext } from "@miden-sdk/react";
import { AccountStorageMode } from "@miden-sdk/miden-sdk";

<SignerContext.Provider value={{
  name: "MyWallet",
  storeName: `mywallet_${userAddress}`,  // unique per user for DB isolation
  isConnected: true,
  accountConfig: {
    publicKeyCommitment: userPublicKeyCommitment,  // Uint8Array
    storageMode: AccountStorageMode.private(),      // AccountStorageMode instance, not a string
  },
  signCb: async (pubKey, signingInputs) => {
    // Route to your signing service
    return signature;  // Uint8Array
  },
  connect: async () => { /* trigger wallet connection */ },
  disconnect: async () => { /* clear session */ },
}}>
  <MidenProvider config={{ rpcUrl: "testnet" }}>
    <App />
  </MidenProvider>
</SignerContext.Provider>
```

**Required fields:**
- `name` — Display name for the signer
- `storeName` — Unique string per user (isolates IndexedDB data between users)
- `accountConfig` — `{ publicKeyCommitment: Uint8Array; storageMode: AccountStorageMode; ... }` (storage mode is an `AccountStorageMode` instance, e.g. `AccountStorageMode.private()`, not a string)
- `signCb` — Callback that signs transaction data with your key management service
- `connect` / `disconnect` — Session lifecycle handlers

## Custom Account Components

Attach application-specific `AccountComponent` instances (e.g., DEX logic from `.masp` packages) to accounts created by the signer:

```tsx
import { type SignerAccountConfig } from "@miden-sdk/react";
import { AccountComponent } from "@miden-sdk/miden-sdk";

const myDexComponent: AccountComponent = await loadCompiledComponent();

const accountConfig: SignerAccountConfig = {
  publicKeyCommitment: userPublicKeyCommitment,
  storageMode: myStorageMode,            // an AccountStorageMode instance (e.g. AccountStorageMode.public())
  customComponents: [myDexComponent],
};
```

`SignerAccountConfig` has an `accountType` field, but it is ignored — account kind and code mutability are not encoded in the account, so visibility comes solely from `storageMode`. Omit it.

Components are appended to the `AccountBuilder` after the default basic wallet component. The field is optional — omitting it preserves default behavior.

## Which Signer to Choose

| Signer | Auth Method | Keys Stored | Best For |
|--------|-------------|-------------|----------|
| Local keystore (default) | None | Browser IndexedDB | Development, demos |
| Para | EVM wallet | Para servers | Apps with existing EVM users |
| Turnkey | Passkey (biometric) | Turnkey servers | Consumer apps, no seed phrases |
| MidenFi Wallet | Browser extension | Extension | Power users with MidenFi wallet |
| Custom | Your choice | Your infrastructure | Enterprise, custom auth flows |

**Key trade-off**: Local keystore requires no setup but keys are lost if the user clears browser data. External signers persist keys server-side but add a dependency.
