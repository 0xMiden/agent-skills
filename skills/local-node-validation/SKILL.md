---
name: local-node-validation
description: Validates Miden contracts against a local node. Covers node setup, Rust binary adaptation, state verification, and troubleshooting. Use after MockChain tests pass to verify contracts work against a real node.
---

# Local Node Validation

Validates that contracts working in MockChain also work against a real Miden node. This catches known MockChain/live-node behavior gaps before they become harder to debug in production or the frontend.

## Why This Matters

MockChain simplifies execution in ways that hide real-world failures:

1. **No automatic block production** -- MockChain requires explicit `prove_next_block()`. A live node produces blocks on its own schedule.
2. **No network transport** -- MockChain does not simulate the network transaction builder (ntx-builder) that handles network notes.
3. **No RPC latency or timeouts** -- MockChain executes locally and instantly. Live nodes have gRPC round-trips with configurable timeouts.
4. **No version/genesis validation** -- MockChain skips the protocol-version check that a live node negotiates at connect (a mismatched node is rejected).
5. **Account update block numbers not tracked** -- MockChain returns chain tip instead of actual update block number.
6. **No mempool or batching** -- MockChain does not simulate transaction queuing, batch formation, or block inclusion delays.

## Prerequisites

- [ ] Your MockChain integration tests pass (e.g. `cargo test -p <your-integration-crate> --release`)
- [ ] A v0.15 Miden node available locally. The v0.15 node is **not** a single binary -- it is composed of standalone executables (validator, sequencer, ntx-builder, transaction prover). The client's own test infra installs the full set: `miden-validator`, `miden-node`, `miden-ntx-builder`, `miden-remote-prover` (see `scripts/start-test-node.sh` in the `miden-client` repo). Install them from the node source pinned in your client's `Cargo.lock`, or follow the 0xMiden/node v0.15 quickstart for the authoritative install flow.
- [ ] A working network/integration validation binary exists in your project that you can use as the starting template for the localhost variant

> The local-node launch CLI lives in the 0xMiden/node repo, not in `miden-client`. The commands below are the topology the client's `scripts/start-test-node.sh` drives; confirm exact flags against your installed node's `--help` for the version you run.

## Step 1: Clean State and Start Local Node

**Every node session must start from clean state.** Stale store files and keystore directories cause conflicts, deserialization errors, and misleading test results. Always wipe before starting. (v0.15 artifacts also do not round-trip across versions, so a fresh store is required after any version change.)

The simplest path is the client's bundled helper script, which installs the node binaries (pinned to your `Cargo.lock`), generates genesis, bootstraps each component, and starts the split topology for you:

```bash
# From a checkout of the miden-client repo pinned to your client version:
./scripts/start-test-node.sh            # foreground, streams logs; Ctrl+C stops
# or
./scripts/start-test-node.sh --background   # returns once RPC is ready (used by CI)
```

This brings up the four-component topology and exposes the RPC on `127.0.0.1:57291` (the client default, `MIDEN_NODE_PORT`).

If you run the node binaries directly instead of via the script, the shape is below. Treat it as a reference skeleton, not a copy-paste recipe: it omits details the script handles for you (it does not show generating the genesis config the validator bootstraps from, and it leaves out the shared network-tx auth header that the sequencer and ntx-builder must agree on or the sequencer rejects the ntx-builder's transactions). Verify every subcommand and flag against `--help` for your node version, or just use the script.

```bash
# 1. Bootstrap each component from a generated genesis block
#    (the genesis config/block must be produced first; the helper script
#     builds it from the client repo before this step)
miden-validator   bootstrap --data-directory <data>/validator \
  --genesis-block-directory <data>/genesis --accounts-directory <data>/accounts \
  --genesis-config-file <data>/genesis-config/genesis.toml
miden-node        bootstrap --data-directory <data>/node        --file <data>/genesis/genesis.dat
miden-ntx-builder bootstrap --data-directory <data>/ntx-builder --file <data>/genesis/genesis.dat

# 2. Start the components (validator, then sequencer with the RPC, prover, ntx-builder).
#    The sequencer and ntx-builder additionally need a matching network-tx auth header
#    (--rpc.network-tx-auth-header-value / --rpc.auth-header-value in the script); see the script.
miden-validator start --listen 127.0.0.1:50101 --data-directory <data>/validator
miden-node sequencer --rpc.listen 127.0.0.1:57291 --data-directory <data>/node \
  --validator.url http://127.0.0.1:50101 --ntx-builder.url http://127.0.0.1:50301 \
  --block.interval 3s --batch.interval 1s
miden-remote-prover --kind=transaction --port=50051
miden-ntx-builder start --listen 127.0.0.1:50301 --rpc.url http://127.0.0.1:57291 \
  --tx-prover.url http://127.0.0.1:50051 --data-directory <data>/ntx-builder
```

**This clean-start sequence is mandatory every time.** Do not attempt to reuse state from a previous session.

## Step 2: Add a localhost client helper

Add a `setup_local_client()` function to whichever helper module in your integration / test harness already hosts the network `setup_client()` equivalent. Name and location are up to you -- adjust to your repo's layout.

`.sqlite_store(..)` is **not** an inherent `ClientBuilder` method in v0.15 -- it comes from an extension trait in the `miden-client-sqlite-store` crate. You must bring it into scope or the call fails to compile (method not found):

```rust
use miden_client_sqlite_store::ClientBuilderSqliteExt; // required for .sqlite_store(..)
```

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

Add a local validation binary alongside your existing network/testnet validation binary, mirroring its structure but swapping in `setup_local_client()`. Pick any conventional name for it (for example `validate_local`) -- adjust to your repo's binary layout.

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

Run the node with verbose logging. The helper script honors `RUST_LOG` and writes a per-component log file per service; if you launch the binaries directly, set it on the process you want to inspect (the sequencer carries the RPC):

```bash
RUST_LOG=info ./scripts/start-test-node.sh
# or, running the sequencer directly:
RUST_LOG=info miden-node sequencer --rpc.listen 127.0.0.1:57291 --data-directory <data>/node \
  --validator.url http://127.0.0.1:50101 --ntx-builder.url http://127.0.0.1:50301 \
  --block.interval 3s --batch.interval 1s
```

Look for:
- Transaction acceptance/rejection messages
- Block production confirmations
- Error or warning lines

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| `Unavailable` RPC error | Node not running or wrong port | Start node, verify the sequencer's RPC is listening on 57291 |
| Version mismatch error | Node and client crate versions differ | Run a v0.15 node built from the node source pinned in your client's `Cargo.lock`; the protocol version is negotiated at connect and a mismatch is rejected |
| Transaction rejected | Invalid proof or state | Check contract code, reset node data, try again |
| Account not found after creation | Haven't synced | Call `sync_state()` after account creation |
| Store errors or deserialization failures | Stale state from previous session (or 0.14 artifacts, which do not round-trip) | Wipe the node data, keystore, and client store, then re-bootstrap from a fresh genesis |
| `.sqlite_store(..)` does not compile | Extension trait not in scope | `use miden_client_sqlite_store::ClientBuilderSqliteExt;` |
| Block not produced | Node produces blocks on the sequencer's configured cadence | Submit a transaction; check the sequencer's `--block.interval` (and `--batch.interval`) settings, or consult `miden-node sequencer --help` |
