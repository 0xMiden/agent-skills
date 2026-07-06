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

- **Provider order is INVERTED: `MidenProvider` runs OUTSIDE `MidenFiSignerProvider`** — see `src/providers.tsx`. This is the opposite of the canonical signer-outer / Miden-inner nesting at the top of this skill, and it is deliberate. In v0.15, when a signer provider is an *ancestor* of `MidenProvider`, `MidenProvider` treats it as its external keystore and does NOT create the `WebClient` until the signer connects (the init effect sees `signerIsConnected === false` and returns early before building the client). With a wallet that hasn't connected — or any environment without the extension — the app would hang on "Initializing…" and even public reads couldn't run. The template never signs *through* `MidenProvider` (it signs its only write, the counter increment, through the local `WebClient` rather than the wallet), so it runs `MidenProvider` in local-keystore mode (no signer ancestor → it initializes immediately, reads work pre-connect) and keeps `MidenFiSignerProvider` *inside*, purely for the connect button and the wallet's `requestTransaction`. `MidenFiSignerProvider` works standalone (it provides its own `WalletContext` + `SignerContext`; no `MultiSignerProvider` needed). Use this inversion only when you do not sign through `MidenProvider`; if external-keystore signing IS the goal, keep the canonical signer-outer order so `MidenProvider` picks up the signer's `signCb`/`accountConfig`.
- **Wallet button uses `useMidenFiWallet()` + `WalletReadyState`** — see `src/components/AppContent.tsx`. The button gates on `wallet?.readyState` (rendering a disabled "Install MidenFi Wallet" state unless `readyState` is `Installed` or `Loadable`) so it can show install state before the extension is detected. `useSigner().connect()` would silently fall through to the adapter's `window.open(adapter.url, ...)` install fallback; gating on `readyState` avoids that path.
- **The counter increment is a local two-transaction flow, not a wallet-signed tx** — see `src/hooks/useIncrementCounter.ts`. It does not use the wallet at all. It creates a throwaway local sender (`client.newWallet(...)`), publishes a plain increment note as that sender's own output note (`TransactionRequestBuilder().withOwnOutputNotes(...)`), then consumes the note *as the counter* (`client.newConsumeTransactionRequest([note])`). Both transactions are submitted by the local `WebClient` via `submitNewTransactionWithProver(accountId, request, prover)` (remote prover), never by the wallet, so `useWaitForCommit` doesn't apply and the template polls the counter's storage map instead. This mirrors the project-template `increment_count` reference.
  - **The note APIs in that hook (use as the reference):** the JS `NoteMetadata` constructor is attachment-less — `new NoteMetadata(sender, noteType, tag)`. Build the note with `new Note(new NoteAssets(), metadata, recipient)`. The increment note carries no attachment and uses tag `0`; the counter is a plain **public `NoAuth`** account, so anyone can consume the note against it with no signature. (Attachments still exist for other uses — `NoteAttachment.fromWord(scheme, word)` / `fromWords(scheme, words)`, read back via `.toWords()`, or `createNoteAttachment(...)` — but the increment does not need one. v0.15 removed the network-account model, so there is no network-execution targeting.)
  - **Two hard requirements (don't regress):** (1) the client runs with `useWorker: false` on `MidenProvider`. The default worker shim keeps a separate in-memory SMT forest per thread; consuming against an *imported* (not locally-created) account applies a delta transaction whose apply step looks the account up in the executing (worker) forest, which never contains the late-imported counter, so it fails with `account data wasn't found` ([web-sdk#222](https://github.com/0xMiden/web-sdk/issues/222)). One thread means one forest, which fixes it. (2) Submits go through the remote prover (`submitNewTransactionWithProver`) so the worker-less single thread only pays local execution, not minutes of local proving. The increment works end-to-end on v0.15 (verified on testnet); there is no `INCREMENT_ONCHAIN_BLOCKED` flag.

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
