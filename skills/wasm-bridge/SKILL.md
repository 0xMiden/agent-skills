---
name: wasm-bridge
description: Enforce conventions for the Rust<->JavaScript WASM boundary in the web-sdk repo (crate miden-client-web at crates/web-client, split out of miden-client). Use when exposing Rust methods to JS via the #[js_export] proc-macro, creating newtype wrappers, handling errors across the boundary with JsErr, bridging JS Promises to Rust Futures, or layering the public MidenClient resource API on top of the WASM-bound WebClient.
---

# WASM Bridge Patterns (web-client / miden-client-web)

At v0.15 the web client lives in the dedicated **web-sdk** repo
(`github.com/0xMiden/web-sdk`), split out of `miden-client`. The Rust<->JS
boundary crate is `crates/web-client` (cargo package `miden-client-web`).
Companion workspace crates: `crates/js-export-macro` (the `#[js_export]`
proc-macro) and `crates/idxdb-store` (the IndexedDB store).

The crate dual-targets two binding technologies from one Rust source:
- **browser** (the `browser` feature) via `wasm_bindgen`, error type `JsValue`
- **Node.js** (the `nodejs` feature) via `napi` / `napi-derive`, error type
  `napi::Error`

A platform abstraction layer in `crates/web-client/src/platform.rs` provides
type aliases and helpers so most code is written once. Key aliases:

- `JsErr` — the platform error type (`wasm_bindgen::JsValue` on browser,
  `napi::Error` on nodejs). `from_str_err(msg: &str) -> JsErr` builds one from a
  string.
- `JsU64` — `u64` on browser, `napi::bindgen_prelude::BigInt` on nodejs; both
  surface as a JS `BigInt`. Convert with `js_u64_to_u64` / `u64_to_js_u64`.
- `JsBytes` — `js_sys::Uint8Array` on browser, `napi::bindgen_prelude::Buffer`
  on nodejs. Convert with `bytes_to_js` / `js_to_bytes`.
- `AsyncCell<T>` — interior mutability: `RefCell` on browser, `tokio::sync::Mutex`
  on nodejs; `.lock().await` yields a `DerefMut` guard.

## Exposing Rust Methods to JavaScript

### Method Annotation — `#[js_export]`

The public API is exposed with the custom `#[js_export]` proc-macro from the
`js-export-macro` crate, **not** raw `#[wasm_bindgen]`. `#[js_export]` generates
the dual `wasm_bindgen` (browser) and `napi` (Node.js) annotations from one
attribute, forwarding `constructor` / `js_name` / `getter`. When a signature
contains `JsU64`, the macro splits the impl per platform, replacing `JsU64` with
`u64` (browser) or `BigInt` (nodejs) — so `JsU64` is resolved by the macro and
does not need to be imported in the annotated module. Raw `#[wasm_bindgen]` is
reserved for browser-only members (e.g. synchronous getters that cannot be async).

Apply `#[js_export]` to the struct/enum/impl block, and `#[js_export(js_name =
"camelCase")]` to each method to map snake_case Rust to camelCase JS:

```rust
use js_export_macro::js_export;

use crate::models::account_header::AccountHeader;
use crate::platform::{JsErr, from_str_err};
use crate::{WebClient, js_error_with_context};

#[js_export]
impl WebClient {
    #[js_export(js_name = "getAccounts")]
    pub async fn get_accounts(&self) -> Result<Vec<AccountHeader>, JsErr> {
        let mut guard = self.get_mut_inner().await;
        let client = guard
            .as_mut()
            .ok_or_else(|| from_str_err("Client not initialized"))?;

        let result = client
            .get_account_headers()
            .await
            .map_err(|err| js_error_with_context(err, "failed to get accounts"))?;

        Ok(result.into_iter().map(|(header, _)| header.into()).collect())
    }
}
```

Rules:
- Annotate with `#[js_export]` (struct/impl) and `#[js_export(js_name = ...)]`
  (methods). Use `#[js_export(constructor)]` for constructors,
  `#[js_export(getter)]` for getters. Use raw `#[wasm_bindgen]` only for
  browser-only items.
- Methods take `&self` (the inner client is behind an `AsyncCell`/lock, so no
  `&mut self`). Acquire the client with `let mut guard =
  self.get_mut_inner().await;` then `let client = guard.as_mut().ok_or_else(||
  from_str_err("Client not initialized"))?;`. `get_mut_inner` returns a
  `DerefMut` guard over `Option<Client<ClientAuth>>`.
- Return `Result<T, JsErr>` — never `Result<T, JsValue>` directly, and never
  panic across the boundary.
- Use `.map_err(|err| js_error_with_context(err, "context"))` for all fallible
  client calls.
- Convert return types via `.into()` (implement `From` on wrapper types).

## Error Handling Across the Boundary

### js_error_with_context

Use the `js_error_with_context` helper (in `crates/web-client/src/lib.rs`) to
chain error sources and attach hints. It returns `JsErr` and splits per
platform; the browser branch additionally attaches a stable machine-readable
`code`:

```rust
pub(crate) fn js_error_with_context<T>(err: T, context: &str) -> JsErr
where
    T: Error + 'static,
{
    let error_message = build_error_chain(context, &err);
    let help = hint_from_error(&err);

    #[cfg(feature = "browser")]
    {
        let js_error: JsValue = JsError::new(&error_message).into();
        if let Some(help) = help {
            let _ = Reflect::set(&js_error, &JsValue::from_str("help"), &JsValue::from_str(&help));
        }
        // Stable, machine-readable code for the ClientError variants JS callers
        // branch on, so they don't depend on (changeable) message text.
        if let Some(code) = code_from_error(&err) {
            let _ = Reflect::set(&js_error, &JsValue::from_str("code"), &JsValue::from_str(code));
        }
        js_error
    }

    #[cfg(feature = "nodejs")]
    {
        let message = match help {
            Some(help) => format!("{error_message} [help: {help}]"),
            None => error_message,
        };
        napi::Error::from_reason(message)
    }
}
```

This:
1. Chains all error sources into one message via `build_error_chain(context,
   &err)` (walks `err.source()`, writing `context: err1: err2: ...`).
2. Extracts an `ErrorHint` from `ClientError` via `hint_from_error` if available.
3. Browser path: attaches `help` (the hint) and `code` (from `code_from_error`,
   which maps the few `ClientError` variants JS callers branch on, e.g.
   `ACCOUNT_NOT_FOUND_ON_CHAIN`, `ACCOUNT_ALREADY_TRACKED`) as properties on the
   JS `Error` via `Reflect::set`.
4. Node.js path: returns `napi::Error::from_reason(...)` with the help inlined
   into the message.

### Error Pattern in Every Method

```rust
client
    .some_operation()
    .await
    .map_err(|err| js_error_with_context(err, "failed to <describe operation>"))?;
```

The context string should be lowercase and describe the failed operation. For
the not-initialized guard, build the error with `from_str_err("Client not
initialized")` (the platform helper), not `JsValue::from_str(...)`.

## Newtype Wrappers

### Pattern

Wrap native Miden types in thin newtypes for JS exposure, annotated with
`#[js_export]`. Fallible construction returns `Result<Self, JsErr>`:

```rust
use js_export_macro::js_export;
use miden_client::{Felt as NativeFelt, Word as NativeWord};
use crate::platform::{JsBytes, JsErr, from_str_err, js_u64_to_u64, u64_to_js_u64};

#[derive(Clone)]
#[js_export]
pub struct Word(NativeWord);

#[js_export]
impl Word {
    #[js_export(constructor)]
    pub fn new(u64_vec: Vec<JsU64>) -> Result<Word, JsErr> {
        if u64_vec.len() != 4 {
            return Err(from_str_err(&format!(
                "Word requires exactly 4 elements, got {}",
                u64_vec.len()
            )));
        }
        let fixed_array_u64: [u64; 4] = u64_vec
            .into_iter()
            .map(js_u64_to_u64)
            .collect::<Vec<u64>>()
            .try_into()
            .expect("length checked above");
        let native_felt_vec: [NativeFelt; 4] = fixed_array_u64
            .iter()
            .map(|&v| NativeFelt::new(v)) // fallible on the 0.15 surface
            .collect::<Result<Vec<NativeFelt>, _>>()
            .map_err(|err| from_str_err(&format!("invalid field element: {err}")))?
            .try_into()
            .expect("length checked above");
        Ok(Word(native_felt_vec.into()))
    }

    #[js_export(js_name = "fromHex")]
    pub fn from_hex(hex: String) -> Result<Word, JsErr> {
        let native_word = NativeWord::try_from(hex.as_str())
            .map_err(|err| from_str_err(&format!("Error instantiating Word from hex: {err}")))?;
        Ok(Word(native_word))
    }
}
```

Notes:
- `JsU64` (BigInt-aware) is used for numeric inputs, not `u64`, so full 64-bit
  precision survives the JS `Number`/`BigInt` boundary. The `#[js_export]` macro
  rewrites `JsU64` per platform, so it is referenced unqualified and is not
  imported alongside the `js_u64_to_u64` / `u64_to_js_u64` converters.
- Constructors that can fail (length checks, fallible `Felt::new`) return
  `Result<_, JsErr>`; do not paper over failures with `.unwrap()`.
- `from_hex` takes `String` (not `&str`) and returns `Result<Word, JsErr>`.

### Required Conversions and Accessors

Implement the `From` conversions, and put the internal `as_native` accessor in a
**plain** `impl` block (not under `#[js_export]`, since it is `pub(crate)`):

```rust
impl Word {
    pub(crate) fn as_native(&self) -> &NativeWord {
        &self.0
    }
}

// Native -> Wrapper (by value and by ref)
impl From<NativeWord> for Word {
    fn from(native_word: NativeWord) -> Self { Word(native_word) }
}
impl From<&NativeWord> for Word {
    fn from(native_word: &NativeWord) -> Self { Word(*native_word) }
}

// Wrapper -> Native (by value and by ref)
impl From<Word> for NativeWord {
    fn from(word: Word) -> Self { word.0 }
}
impl From<&Word> for NativeWord {
    fn from(word: &Word) -> Self { word.0 }
}
```

For wrapper newtypes that must be accepted as by-value or `Vec<T>` parameters on
the Node.js side, also invoke `impl_napi_from_value!(Word);` (defined in
`crates/web-client/src/miden_array.rs`; a no-op under the `browser` feature). It
bridges napi-rs v3's missing `FromNapiValue` for `#[napi]` class types.

### Factory Methods

Provide `fromHex()`-style constructors that return `Result<Self, JsErr>` for
user-facing types.

## Data Transfer Objects

For complex data that crosses the WASM boundary, use a dual-platform
`getter_with_clone` / `napi(object)` struct (gated with `cfg_attr`), and map
field names with browser-side `js_name`:

```rust
#[cfg_attr(feature = "browser", wasm_bindgen(getter_with_clone, inspectable))]
#[cfg_attr(feature = "nodejs", napi(object))]
#[derive(Clone)]
pub struct StorageMapEntry {
    #[cfg_attr(feature = "browser", wasm_bindgen(js_name = "root"))]
    pub root: String,
    #[cfg_attr(feature = "browser", wasm_bindgen(js_name = "key"))]
    pub key: String,
    #[cfg_attr(feature = "browser", wasm_bindgen(js_name = "value"))]
    pub value: String,
}
```

Rules:
- Use the dual-platform `#[cfg_attr(feature = "browser", wasm_bindgen(...))]` +
  `#[cfg_attr(feature = "nodejs", napi(object))]` form — never a bare
  `#[wasm_bindgen(getter_with_clone)]`.
- `getter_with_clone` auto-generates JS getters; `inspectable` improves console
  inspection. `inspectable` can also stand alone (without `getter_with_clone`)
  on an opaque wrapper class, again via the dual form `#[cfg_attr(feature =
  "browser", wasm_bindgen(inspectable))]` + `#[cfg_attr(feature = "nodejs",
  napi)]` (note: bare `napi`, not `napi(object)`, for a class that wraps a
  native handle rather than a plain-data object).
- Field names: snake_case in Rust, camelCase via browser-side `js_name`.
- Serialize complex values to hex strings or `JsBytes`/`Vec<u8>` where needed.

## Promise Handling (idxdb-store pattern)

When calling JS functions from Rust that return Promises (the IndexedDB store,
in `crates/idxdb-store`), use these helpers (`crates/idxdb-store/src/promise.rs`):

```rust
/// Awaits a JavaScript Promise and returns the raw JsValue.
pub(crate) async fn await_js_value(promise: Promise, ctx: &str) -> Result<JsValue, StoreError> {
    JsFuture::from(promise)
        .await
        .map_err(|js_error| StoreError::DatabaseError(format!("{ctx}: {js_error:?}")))
}

/// Awaits a JavaScript Promise and deserializes into T.
pub(crate) async fn await_js<T>(promise: Promise, ctx: &str) -> Result<T, StoreError>
where
    T: DeserializeOwned,
{
    let js_value = await_js_value(promise, ctx).await?;
    from_value(js_value)
        .map_err(|err| StoreError::DatabaseError(format!("failed to deserialize ({ctx}): {err:?}")))
}

/// Awaits a JavaScript Promise and discards the result.
pub(crate) async fn await_ok(promise: Promise, ctx: &str) -> Result<(), StoreError> {
    let _ = await_js_value(promise, ctx).await?;
    Ok(())
}
```

Rules:
- Always provide a context string describing what the await is for.
- Use `await_js::<T>()` when you need to deserialize the result.
- Use `await_ok()` when you only care about success/failure.
- Use `serde_wasm_bindgen::from_value()` for deserialization, not `serde_json`.
  (At v0.15 `Promise` is imported via `wasm_bindgen_futures::js_sys::Promise`.)

## Importing JS Functions from Rust

Declare external JS functions with `#[wasm_bindgen(module = "...")]` (browser /
idxdb-store side):

```rust
#[wasm_bindgen(module = "/src/js/utils.js")]
extern "C" {
    #[wasm_bindgen(js_name = logWebStoreError)]
    fn log_web_store_error(error: JsValue, error_context: alloc::string::String);
}

#[wasm_bindgen(module = "/src/js/schema.js")]
extern "C" {
    /// Opens the database and registers it in the JS registry.
    #[wasm_bindgen(js_name = openDatabase)]
    fn open_database(network: &str, client_version: &str) -> js_sys::Promise;
}
```

Rules:
- Module path is relative to the crate root.
- Function names are snake_case in Rust, mapped via `js_name`.
- Return `js_sys::Promise` for async operations.
- Pass simple types across the boundary: `&str`, `JsValue`, `Vec<u8>`, `u32`.

## JS Wrapper Layer

The web-client crate ships **two** JS layers under `crates/web-client/js/`:

1. **`WebClient`** (`js/index.js`) — the WASM-bound class re-exported as
   `WasmWebClient` (`export { WebClient as WasmWebClient, MockWebClient as
   MockWasmWebClient }`). It wraps the `WebClient` Rust struct and adds JS-side
   concerns:
   - `_serializeWasmCall` queue that linearizes WASM calls (the inner client is
     behind a lock, so the JS side must not interleave async calls).
   - `syncState()` is wrapped in the exported `withSyncLock(dbId, methodId, fn)`
     helper (`js/syncLock.js`, Web Locks via `navigator.locks`) to coalesce
     concurrent syncs and serialize them across tabs:
     `return await withSyncLock(dbId, methodId, async () =>
     this._serializeWasmCall(...))`.
   - method-classification sets (`SYNC_METHODS`, `READ_METHODS`,
     `WRITE_METHODS`) consumed by the proxy and enforced by
     `scripts/check-method-classification.js`. (`SYNC_METHODS` is a historical
     misnomer — it groups methods safe to bind raw.)
2. **`MidenClient`** (`js/client.js`) — the public, resource-based wrapper that
   owns a `WebClient` instance and exposes typed sub-objects: `client.accounts`,
   `client.transactions`, `client.notes`, `client.tags`, `client.settings`,
   `client.compile` (a `CompilerResource`, hence the property is `compile`
   though the file is `compiler.js`), and `client.keystore`. Each resource lives
   under `js/resources/<name>.js`.

`index.js` injects the WASM constructor and the `getWasm` initializer into
`MidenClient` via static fields to break the import cycle:

```javascript
MidenClient._WasmWebClient = WebClient;
MidenClient._MockWasmWebClient = MockWebClient;
MidenClient._getWasmOrThrow = getWasmOrThrow;
```

There is **no** `safe-arrays.js` module. The wasm-bindgen array wrappers
(`NoteArray`, `OutputNoteArray`, `AccountArray`, `ForeignAccountArray`, ...) are
generated by the `declare_js_miden_arrays!` macro (defined in
`crates/web-client/src/miden_array.rs`, invoked in
`crates/web-client/src/models/mod.rs`), and their constructor **consumes** its
elements. To keep an element usable afterwards, construct the array empty and
`push` by reference instead of passing elements to the constructor:

```javascript
// NoteArray constructor consumes its elements; use push(&note) to keep
// `note` valid so it can be returned to the caller.
const ownOutputs = new wasm.NoteArray();
ownOutputs.push(note);
```

### Adding a method

When extending the SDK, choose the layer based on whether the work is
**Rust-side** or **glue/shape**:

- **Rust-side logic** (new RPC call, new transaction request type, storage
  access): expose a method on the WASM `WebClient` impl with `#[js_export(js_name
  = "camelCase")]`, then surface it from the matching resource in
  `js/resources/`. Update the method-classification sets in `index.js` so the
  linter (`scripts/check-method-classification.js`) accepts it.
- **JS-side ergonomics** (option-bag normalization, account-ref resolution, type
  coercion): keep the work in the resource module and call the existing WASM
  method.

Resource methods follow this shape:

```javascript
// crates/web-client/js/resources/accounts.js
async get(ref) {
  this.#client.assertNotTerminated();
  const wasm = await this.#getWasm();
  const id = resolveAccountRef(ref, wasm);   // accepts string | AccountId | Account | AccountHeader
  const account = await this.#inner.getAccount(id);
  return account ?? null;
}
```

Rules:

- Always call `this.#client.assertNotTerminated()` at entry — late callbacks on
  a torn-down client otherwise panic with "null pointer passed to rust".
- Resolve account/note/storage refs through the helpers in
  `crates/web-client/js/utils.js` (e.g. `resolveAccountRef`,
  `resolveStorageMode`), imported from a resource as `../utils.js`, so callers
  can pass any natural form (hex, bech32 address, WASM type). (There is no
  `utils.js` inside `js/resources/` — that directory holds only the seven
  resource files: accounts, compiler, keystore, notes, settings, tags,
  transactions.)
- Return WASM-owned objects (e.g. `Account`, `AccountHeader`) directly when
  callers will use them again — wrapping them in plain JS DTOs forces another
  WASM round-trip and breaks identity for code that compares by reference.
