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

// `config` is REQUIRED. `defaultOrganizationId` is required;
// `apiBaseUrl` is optional (defaults to https://api.turnkey.com).
// There is no environment-variable fallback — pass the org id explicitly.
<TurnkeySignerProvider config={{ defaultOrganizationId: "your-org-id" }}>
  <MidenProvider config={{ rpcUrl: "testnet" }}>
    <App />
  </MidenProvider>
</TurnkeySignerProvider>

// Or override the API base URL:
<TurnkeySignerProvider config={{
  apiBaseUrl: "https://api.turnkey.com",
  defaultOrganizationId: "your-org-id",
}}>
  ...
</TurnkeySignerProvider>
```

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

> The provider still accepts an `accountType` prop, but it is a no-op as of protocol 0.15: account visibility is determined solely by `storageMode` (`private`/`public`), and the provider always imports the account by ID (`importAccountId`), bypassing the builder path entirely. Omit it.

### Frontend-template-specific MidenFi pattern

The [frontend template](https://github.com/0xMiden/frontend-template) deviates from the generic `useSigner()` approach in two places — worth knowing because it's a pattern you'll likely want when the wallet extension is the primary signer:

- **Wallet button uses `useMidenFiWallet()` + `WalletReadyState`** — see `src/components/AppContent.tsx` in the frontend template. The button gates on `wallet.readyState` so it can render a disabled "Install MidenFi Wallet" state before the extension is detected. `useSigner().connect()` would silently fall through to the adapter's `window.open(adapter.url, ...)` install fallback (Chrome Web Store → Play Store redirect on some platforms); gating on `readyState` avoids that path entirely.
- **Custom transaction flow calls `wallet.requestTransaction(...)` directly** — see `src/hooks/useIncrementCounter.ts` in the frontend template. The counter increment builds a bespoke transaction request (via `TransactionRequestBuilder` with a custom `Note`) and hands it to the wallet (wrapped in `Transaction.createCustomTransaction(...)`) for signing + submission. The React SDK mutation hooks (`useSend`, `useConsume`, ...) don't cover this kind of custom note construction, and the tx is submitted by the wallet rather than the local client — so `useWaitForCommit` doesn't apply either.
  - **Heads up: the published frontend template is still on web-sdk 0.14** (`@miden-sdk/miden-sdk@0.14.x`) and its note-construction code uses removed-in-0.15 APIs — notably `NoteAttachment.newNetworkAccountTarget(...)` and `new NoteMetadata(...).withAttachment(attachment)`. On the 0.15 surface neither exists: `NoteAttachment.newNetworkAccountTarget` (and the old `newWord`/`newArray`/`asWord`/`asArray` accessors) were removed, and the JS `NoteMetadata` constructor is now attachment-less (`new NoteMetadata(sender, noteType, tag)`) with `withAttachment()`/`attachment()`/`withTag()` removed. Build attachments with `NoteAttachment.fromWord(scheme, word)` / `NoteAttachment.fromWords(scheme, words)` (read back via `.toWords()`). The JS `Note` constructor signature is unchanged — `new Note(assets, metadata, recipient)` still takes a `NoteMetadata` (it reconstructs a `PartialNoteMetadata` internally), so the breakage is the attachment APIs, not the `Note` constructor. Treat the template's note-building snippet as a 0.14 pattern to port, not a 0.15 reference.

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

`SignerAccountConfig` still carries an `accountType` field, but it is `@deprecated` and ignored as of protocol 0.15: code mutability and faucet/wallet account kinds are no longer encoded in the account, so visibility is determined solely by `storageMode`. Omit it.

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
