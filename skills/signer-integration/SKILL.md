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
import { ParaSignerProvider } from "@miden-sdk/para";

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

// Config is optional — defaults to https://api.turnkey.com
// and reads VITE_TURNKEY_ORG_ID from environment
<TurnkeySignerProvider>
  <MidenProvider config={{ rpcUrl: "testnet" }}>
    <App />
  </MidenProvider>
</TurnkeySignerProvider>

// Or with explicit config:
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

const { isConnected, connect, disconnect } = useSigner();
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
  accountType="RegularAccountImmutableCode"               // Default: "RegularAccountImmutableCode"
  storageMode="public"                                    // "private" | "public" | "network". Default: "public"
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

### Frontend-template-specific MidenFi pattern

The [frontend template](https://github.com/0xMiden/frontend-template) deviates from the generic `useSigner()` approach in two places — worth knowing because it's a pattern you'll likely want when the wallet extension is the primary signer:

- **Wallet button uses `useMidenFiWallet()` + `WalletReadyState`** — see `src/components/AppContent.tsx` in the frontend template. The button gates on `wallet.readyState` so it can render a disabled "Install MidenFi Wallet" state before the extension is detected. `useSigner().connect()` would silently fall through to the adapter's `window.open(adapter.url, ...)` install fallback (Chrome Web Store → Play Store redirect on some platforms); gating on `readyState` avoids that path entirely.
- **Custom transaction flow calls `wallet.requestTransaction(...)` directly** — see `src/hooks/useIncrementCounter.ts` in the frontend template. The counter increment builds a bespoke `TransactionRequest` (via `TransactionRequestBuilder`, a custom `Note` with `NoteAttachment.newNetworkAccountTarget`, etc.) and hands it to the wallet for signing + submission. The React SDK mutation hooks (`useSend`, `useConsume`, ...) don't cover this kind of custom note construction, and the tx is submitted by the wallet rather than the local client — so `useWaitForCommit` doesn't apply either.

## Unified Signer Interface

Works with any signer provider above:
```tsx
import { useSigner } from "@miden-sdk/react";

const { isConnected, connect, disconnect, name } = useSigner();

if (!isConnected) {
  return <button onClick={connect}>Connect {name}</button>;
}
```

## Building a Custom Signer

Implement `SignerContextValue` via `SignerContext.Provider`:

```tsx
import { SignerContext } from "@miden-sdk/react";

<SignerContext.Provider value={{
  name: "MyWallet",
  storeName: `mywallet_${userAddress}`,  // unique per user for DB isolation
  isConnected: true,
  accountConfig: {
    publicKey: userPublicKeyCommitment,  // Uint8Array
    storageMode: "private",
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
- `accountConfig` — Public key commitment + storage mode
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
  accountType: "RegularAccountUpdatableCode",
  storageMode: myStorageMode,
  customComponents: [myDexComponent],
};
```

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
