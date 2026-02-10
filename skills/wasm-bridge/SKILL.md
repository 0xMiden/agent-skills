---
name: wasm-bridge
description: Enforce conventions for the Rust-to-JavaScript WASM boundary in miden-client (web-client crate). Use when exposing Rust methods to JS via wasm_bindgen, creating newtype wrappers, handling errors across the boundary, or bridging JS Promises to Rust Futures.
---

# WASM Bridge Patterns (web-client)

## Exposing Rust Methods to JavaScript

### Method Annotation

Use `#[wasm_bindgen]` on both the struct and impl block. Convert snake_case to camelCase with `js_name`:

```rust
#[wasm_bindgen]
impl WebClient {
    #[wasm_bindgen(js_name = "getAccounts")]
    pub async fn get_accounts(&mut self) -> Result<Vec<AccountHeader>, JsValue> {
        if let Some(client) = self.get_mut_inner() {
            let result = client
                .get_account_headers()
                .await
                .map_err(|err| js_error_with_context(err, "failed to get accounts"))?;

            Ok(result.into_iter().map(|(header, _)| header.into()).collect())
        } else {
            Err(JsValue::from_str("Client not initialized"))
        }
    }
}
```

Rules:
- Always check `self.get_mut_inner()` first — return `Err(JsValue::from_str("Client not initialized"))` if None
- Use `.map_err(|err| js_error_with_context(err, "context"))` for all error conversion
- Convert return types via `.into()` (implement `From` on wrapper types)
- Use `#[wasm_bindgen(constructor)]` for constructors
- Return `Result<T, JsValue>` — never panic across the WASM boundary

## Error Handling Across the Boundary

### js_error_with_context

Always use the `js_error_with_context` helper to chain error sources and attach hints:

```rust
fn js_error_with_context<T>(err: T, context: &str) -> JsValue
where
    T: Error + 'static,
{
    let mut error_string = context.to_string();
    let mut source = Some(&err as &dyn Error);
    while let Some(err) = source {
        write!(error_string, ": {err}").expect("writing to string should always succeed");
        source = err.source();
    }

    let help = hint_from_error(&err);
    let js_error: JsValue = JsError::new(&error_string).into();

    if let Some(help) = help {
        let _ = Reflect::set(&js_error, &JsValue::from_str("help"), &JsValue::from_str(&help));
    }

    js_error
}
```

This:
1. Chains all error sources into one message string
2. Extracts `ErrorHint` from `ClientError` if available
3. Attaches hint as a `help` property on the JS Error object via `Reflect::set`

### Error Pattern in Every Method

```rust
client
    .some_operation()
    .await
    .map_err(|err| js_error_with_context(err, "failed to <describe operation>"))?;
```

The context string should be lowercase and describe the failed operation.

## Newtype Wrappers

### Pattern

Wrap native Miden types in thin newtypes for WASM exposure:

```rust
#[wasm_bindgen]
#[derive(Clone)]
pub struct Word(NativeWord);

#[wasm_bindgen]
impl Word {
    #[wasm_bindgen(constructor)]
    pub fn new(u64_vec: Vec<u64>) -> Word {
        let fixed_array_u64: [u64; 4] = u64_vec.try_into().unwrap();
        let native_felt_vec: [NativeFelt; 4] = fixed_array_u64
            .iter()
            .map(|&v| NativeFelt::new(v))
            .collect::<Vec<NativeFelt>>()
            .try_into()
            .unwrap();
        Word(native_felt_vec.into())
    }

    #[wasm_bindgen(js_name = "fromHex")]
    pub fn from_hex(hex: &str) -> Result<Word, JsValue> {
        let native_word = NativeWord::try_from(hex).map_err(|err| {
            JsValue::from_str(&format!("Error instantiating Word from hex: {err}"))
        })?;
        Ok(Word(native_word))
    }

    pub(crate) fn as_native(&self) -> &NativeWord {
        &self.0
    }
}
```

### Required Conversions

Every newtype wrapper must implement:

```rust
// Native → Wrapper
impl From<NativeWord> for Word {
    fn from(native_word: NativeWord) -> Self {
        Word(native_word)
    }
}

// &Wrapper → Native
impl From<&Word> for NativeWord {
    fn from(word: &Word) -> Self {
        word.0
    }
}
```

And provide an internal accessor:

```rust
pub(crate) fn as_native(&self) -> &NativeType {
    &self.0
}
```

### Factory Methods

Provide `fromHex()` constructors that return `Result<Self, JsValue>` for user-facing types.

## Data Transfer Objects

For complex data that crosses the WASM boundary (e.g., sync updates), use `getter_with_clone` structs:

```rust
#[wasm_bindgen(getter_with_clone)]
#[derive(Clone)]
pub struct JsStateSyncUpdate {
    #[wasm_bindgen(js_name = "blockNum")]
    pub block_num: u32,

    #[wasm_bindgen(js_name = "flattenedNewBlockHeaders")]
    pub flattened_new_block_headers: FlattenedU8Vec,

    #[wasm_bindgen(js_name = "newBlockNums")]
    pub new_block_nums: Vec<u32>,
}
```

Rules:
- Use `#[wasm_bindgen(getter_with_clone)]` to auto-generate JS getters
- Use `#[wasm_bindgen(inspectable)]` for types that benefit from console inspection
- Field names: snake_case in Rust, camelCase via `js_name`
- Serialize complex types to hex strings or `Vec<u8>`
- Implement `from_*()` builder methods to convert from native types

## Promise Handling (idxdb-store pattern)

When calling JS functions from Rust that return Promises, use these helpers:

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
- Always provide a context string describing what the await is for
- Use `await_js::<T>()` when you need to deserialize the result
- Use `await_ok()` when you only care about success/failure
- Use `serde_wasm_bindgen::from_value()` for deserialization, not `serde_json`

## Importing JS Functions from Rust

Declare external JS functions with `#[wasm_bindgen(module = "...")]`:

```rust
#[wasm_bindgen(module = "/src/js/schema.js")]
extern "C" {
    #[wasm_bindgen(js_name = openDatabase)]
    fn open_database(network: &str, client_version: &str) -> js_sys::Promise;
}

#[wasm_bindgen(module = "/src/js/utils.js")]
extern "C" {
    #[wasm_bindgen(js_name = logWebStoreError)]
    fn log_web_store_error(error: JsValue, error_context: alloc::string::String);
}
```

Rules:
- Module path is relative to the crate root
- Function names are snake_case in Rust, mapped via `js_name`
- Return `js_sys::Promise` for async operations
- Pass simple types across the boundary: `&str`, `JsValue`, `Vec<u8>`, `u32`

## JS Wrapper Layer

The `crates/web-client/js/index.js` wrapper:

- Uses `static async createClient()` factory method
- Lazy-loads WASM via `getWasmWebClient()`
- Uses `Proxy` to forward unknown method calls to the WASM WebClient
- Handles Web Worker communication with `postMessage()` and `Map<requestId, {resolve, reject}>`
- Deserializes errors preserving the full chain (name, message, stack, cause, help)

When adding new simplified JS wrappers, follow this pattern:
```javascript
// In the Proxy handler or on the WebClient class
async simplifiedMethod(stringArg) {
  const wasmClient = await this.getWasmWebClient();
  const nativeId = AccountId.fromHex(stringArg);  // Convert string → native type
  const result = await wasmClient.existingMethod(nativeId);
  return result;  // Return JS-friendly value
}
```
