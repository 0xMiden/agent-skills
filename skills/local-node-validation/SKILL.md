---
name: local-node-validation
description: Validates Miden contracts against a local node. Covers node setup, Rust binary adaptation, state verification, and troubleshooting. Use after MockChain tests pass to verify contracts work against a real node.
---

# Local Node Validation

Validates that contracts working in MockChain also work against a real Miden node. This catches known MockChain/live-node behavior gaps before they become harder to debug in production or the frontend.

## Why This Matters

MockChain simplifies execution in ways that hide real-world failures:

1. **No automatic block production** -- MockChain requires explicit `prove_next_block()`. A live node produces blocks on its own schedule.
2. **No network transport** -- MockChain does not simulate the network transaction builder that handles network notes.
3. **No RPC latency or timeouts** -- MockChain executes locally and instantly. Live nodes have gRPC round-trips with configurable timeouts.
4. **No version/genesis validation** -- MockChain skips the `Accept` header version check that live nodes enforce.
5. **Account update block numbers not tracked** -- MockChain returns chain tip instead of actual update block number.
6. **No mempool or batching** -- MockChain does not simulate transaction queuing, batch formation, or block inclusion delays.

## Prerequisites

- [ ] Your MockChain integration tests pass (e.g. `cargo test -p <your-integration-crate> --release`)
- [ ] `miden-node` installed: `cargo install miden-node --locked`
- [ ] A working network/integration validation binary exists in your project that you can use as the starting template for the localhost variant

## Step 1: Clean State and Start Local Node

**Every node session must start from clean state.** Stale store files and keystore directories cause conflicts, deserialization errors, and misleading test results. Always wipe before starting.

```bash
# 1. Wipe all state from previous runs
rm -rf local-node-data/ local-keystore/ local-store.sqlite3

# 2. Bootstrap fresh node
mkdir -p local-node-data
miden-node bundled bootstrap \
  --data-directory local-node-data \
  --accounts-directory .

# 3. Start node (keep running in separate terminal)
miden-node bundled start \
  --data-directory local-node-data \
  --rpc.url http://0.0.0.0:57291
```

**This clean-start sequence is mandatory every time.** Do not attempt to reuse state from a previous session.

## Step 2: Add a localhost client helper

Add a `setup_local_client()` function to whichever helper module in your integration / test harness already hosts the network `setup_client()` equivalent. Name and location are up to you — adjust to your repo's layout.

```rust
pub async fn setup_local_client() -> Result<ClientSetup> {
    let endpoint = Endpoint::new("http".into(), "localhost".into(), Some(57291));
    let timeout_ms = 10_000;
    let rpc_client = Arc::new(GrpcClient::new(&endpoint, timeout_ms));

    let keystore_path = std::path::PathBuf::from("../local-keystore");
    let keystore = Arc::new(FilesystemKeyStore::new(keystore_path)
        .context("Failed to initialize local keystore")?);

    let store_path = std::path::PathBuf::from("../local-store.sqlite3");

    let client = ClientBuilder::new()
        .rpc(rpc_client)
        .sqlite_store(store_path)
        .authenticator(keystore.clone())
        .in_debug_mode(true.into())
        .build()
        .await
        .context("Failed to build local Miden client")?;

    Ok(ClientSetup { client, keystore })
}
```

Use separate paths (`local-keystore/`, `local-store.sqlite3`) to avoid contaminating testnet state.

## Step 3: Create a local validation binary

Add a local validation binary alongside your existing network/testnet validation binary, mirroring its structure but swapping in `setup_local_client()`. Pick any conventional name for it (for example `validate_local`) — adjust to your repo's binary layout.

The binary must:
1. Call `setup_local_client()` instead of the network setup function
2. Sync state: `client.sync_state().await?`
3. Build contracts (same as the existing binary)
4. Create accounts, create notes, submit transactions
5. Sync again after each transaction submission
6. Wait for transaction inclusion (poll `sync_state` until account state updates)
7. Verify final state matches MockChain test expectations
8. Print clear pass/fail for each verification step

Key differences from the network binary:
- Localhost endpoint (port 57291)
- Separate keystore and store paths
- Must handle block production timing (sync + wait between submissions)

## Step 4: Run and Verify

Ensure clean client state before running (the node should already be clean from Step 1):
```bash
rm -rf local-keystore/ local-store.sqlite3
cargo run --bin <your-local-validation-binary> --release
```

### Verification Checklist

- [ ] `sync_state()` succeeds (node reachable, no version mismatch)
- [ ] Account creation succeeds (account appears after sync)
- [ ] Note publication succeeds (transaction accepted by node)
- [ ] Note consumption succeeds (state transitions as expected)
- [ ] Final state matches MockChain test expectations
- [ ] No RPC timeout errors
- [ ] Node logs show no errors

## Step 5: Inspect Node Logs

Run the node with verbose logging:
```bash
RUST_LOG=info miden-node bundled start \
  --data-directory local-node-data \
  --rpc.url http://0.0.0.0:57291
```

Look for:
- Transaction acceptance/rejection messages
- Block production confirmations
- Error or warning lines

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| `Unavailable` RPC error | Node not running or wrong port | Start node, verify port 57291 |
| Version mismatch error | Node and client crate versions differ | Rebuild node from same miden-node version as client deps |
| Transaction rejected | Invalid proof or state | Check contract code, reset node data, try again |
| Account not found after creation | Haven't synced | Call `sync_state()` after account creation |
| Store errors or deserialization failures | Stale state from previous session | Wipe everything: `rm -rf local-node-data/ local-keystore/ local-store.sqlite3` and re-bootstrap |
| Block not produced | Node produces blocks when transactions arrive | Submit a transaction; check `--block-producer.block-interval` setting |
