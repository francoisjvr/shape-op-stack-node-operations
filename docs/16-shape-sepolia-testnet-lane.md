# Shape Sepolia Testnet Lane

This note records the **isolated Shape Sepolia proving lane** that was brought up on the same VPS **without touching the live mainnet Reth stack**.

This is not the canonical mainnet recipe.
It is the operator note for a safe test lane used to validate how `op-reth` + `op-node` behave on Shape Sepolia.

## Goal

Use Shape Sepolia to answer three practical questions before changing anything important on mainnet:

1. can an isolated `op-reth` + `op-node` pair start cleanly on this VPS
2. can `op-node` derive forward against public Ethereum Sepolia L1 endpoints without immediately rate-limiting itself to death
3. does `op-reth` eventually advance local execution head, not just accept repeated payload attempts

## Isolation rule

Mainnet services remained untouched while this lane was running:

- `shape-mainnet-op-reth`
- `shape-mainnet-op-node-reth`

The testnet lane used separate containers, paths, and ports:

- `shape-sepolia-op-reth`
- `shape-sepolia-op-node-reth`
- Reth data: `/root/shape-sepolia-op-reth-data`
- op-node data: `/root/shape-sepolia-op-node-reth-data`
- config: `/root/.shape-sepolia-op-reth-config`
- snapshot staging: `/root/shape-sepolia-op-reth-upload`

## Snapshot artifact

Latest downloaded snapshot artifact observed during this run:

- `/root/shape-sepolia-op-reth-upload/reth-20260520220001.tar.gz`

Observed runtime datadir size during verification:

- `/root/shape-sepolia-op-reth-data` about `41G`

## Config provenance actually observed

The live config did **not** come from the current public Shape docs artifacts unchanged.

Observed on the VPS:

- `/root/.shape-sepolia-op-reth-config/rollup.snapshot.json`
- `/root/.shape-sepolia-op-reth-config/genesis-l2.snapshot.json`
- `/root/.shape-sepolia-op-reth-config/reth.snapshot.toml`

Those three files are byte-identical to the same files inside the extracted runtime datadir:

- `/root/shape-sepolia-op-reth-data/rollup.json`
- `/root/shape-sepolia-op-reth-data/genesis-l2.json`
- `/root/shape-sepolia-op-reth-data/reth.toml`

So the live lane is effectively using the snapshot-bundled config, copied back out into the explicit config directory.

That matters because the current public Shape Sepolia docs still point to:

- rollup: `https://arweave.net/gSb3hOzLaBIBy-AyWA2O2_03sfXJ-IwPqmV6Mhstmj8`
- genesis: `https://arweave.net/nw6opum2ALT9T39TSAAK_Wq0Z75aZJWd6J2RmiWHm_s`

And the public rollup artifact currently differs materially from the live lane:

- docs rollup has `holocene_time = 1739880000`
- docs rollup omits `chain_op_config`
- docs rollup omits `isthmus_time`
- docs rollup omits `jovian_time`

But the live snapshot-backed rollup includes:

- `holocene_time = 1739880000`
- `isthmus_time = 1763650800`
- `jovian_time = 1777552200`
- `chain_op_config` with EIP-1559 elasticity/denominator fields

The current public OP Superchain Registry also does not validate the live lane's later Sepolia schedule. At the time of writing:

- `superchain/configs/sepolia/shape.toml` only carries through `granite_time`
- `superchain/configs/sepolia/superchain.toml` publishes Sepolia-wide defaults including:
  - `holocene_time = 1732633200`
  - `isthmus_time = 1744905600`
  - `jovian_time = 1763568001`

So the live lane should be understood as using a newer or different snapshot-bundled Shape Sepolia config than the public docs and public registry currently expose.

## Testnet ports used

The isolated Sepolia lane used its own local ports:

- `op-reth` HTTP RPC: `28545`
- `op-reth` WS RPC: `28546`
- `op-reth` Engine/Auth RPC: `28551`
- `op-node` RPC: `29545`
- `op-node` metrics: `27300`
- `op-reth` P2P: `32303`
- `op-node` P2P: `29222` TCP/UDP

## L1 endpoints that behaved better than the first attempt

The first public-RPC attempt hit immediate or repeated L1 rate-limit failures.
A more conservative setup behaved materially better:

- L1 execution RPC: `https://ethereum-sepolia-rpc.publicnode.com`
- L1 beacon RPC: `https://ethereum-sepolia-beacon-api.publicnode.com`

And `op-node` was deliberately throttled:

- `--l1.max-concurrency=1`
- `--l1.rpc-max-batch-size=1`
- `--l1.rpc-rate-limit=4`
- `--l1.http-poll-interval=24s`
- `--l1.rpckind=basic`
- `--syncmode=consensus-layer`
- `--l2.enginekind=reth`

## What improved after throttling

Before throttling, the lane was effectively dead-on-arrival because `op-node` kept hitting L1 receipt or validation failures from public endpoints.

After throttling:

- `op-node` initialized successfully
- derivation pipeline reset completed
- `current_l1` started advancing in repeated `optimism_syncStatus` samples
- logs showed repeated `Advancing bq origin ... originBehind=true`
- finalized/safe/head L1 fields became populated instead of staying zeroed forever

That means the lane became **meaningfully alive**, not just superficially started.

## What did not become healthy yet

With the first throttled public-endpoint profile, the local execution head still stayed flat during repeated checks:

- local `eth_blockNumber` stayed at `30141035`

At the same time, logs showed `op-node` repeatedly trying to feed a newer unsafe payload into `op-reth`:

- repeated target payload around `30406381`
- repeated warning/error shape:
  - `failed to insert unsafe payload`
  - `updated forkchoice, but node is syncing`
  - `failed to check for unsafe L2 blocks to sync`

`op-reth` also kept logging the same received new payload while its own reported latest block remained flat.

## What materially improved after switching to dedicated Alchemy Sepolia L1 endpoints

Later, the lane was switched from PublicNode to the user's dedicated Alchemy Sepolia endpoints:

- execution RPC: `https://eth-sepolia.g.alchemy.com/v2/<KEY>`
- beacon RPC: `https://eth-sepoliabeacon.g.alchemy.com/v2/<KEY>`
- `--l1.rpckind=alchemy`

That change materially altered the behavior.

Observed after the switch:

- execution `eth_blockNumber` resumed advancing
- `eth_syncing` returned `false`
- `safe_l2` and `unsafe_l2` advanced instead of staying pinned
- the lane moved from the old flat point around `30,141,035` up through `30,145,561`, then `30,146,002`, then `30,146,254+` during live verification
- `op-node` peer stats remained healthy while derivation continued

A bounded live check confirmed real forward execution movement over time:

- first local execution height: `30,145,561`
- later local execution height: `30,146,254`
- 45-second follow-up still moved forward: `30,146,001+` to `30,146,254+` while `safe_l2` / `unsafe_l2` continued climbing

At that point the lane was no longer in the old "payloads arrive but canonical head never moves" failure mode.

## Current interpretation

This lane should now be treated as:

- **successfully isolated**
- **past the earlier flat-head stall**
- **actively converging but still behind tip**

During the verification window, the node was still roughly `~265k` L2 blocks behind public Shape Sepolia tip, so the right classification is:

- **healthy catching-up sync**
- **not yet fully caught up / finished**

The most important root-cause takeaway from this lane is that the first public-endpoint profile was good enough to prove the config path, but not good enough to sustain healthy Sepolia convergence. Switching to dedicated Alchemy Sepolia execution + beacon endpoints was the change that finally restored real execution-head movement.
## Minimal verification commands for this lane

### Local testnet execution head
```bash
curl -s -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}' \
  http://127.0.0.1:28545
```

### Testnet rollup sync status
```bash
curl -s -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","method":"optimism_syncStatus","params":[],"id":1}' \
  http://127.0.0.1:29545
```

### Recent testnet op-node logs
```bash
docker logs --tail 80 shape-sepolia-op-node-reth
```

### Recent testnet op-reth logs
```bash
docker logs --tail 80 shape-sepolia-op-reth
```

## 2026-05-26 follow-up monitor check

A later bounded live check against the isolated Sepolia lane confirmed that the earlier flat-head failure mode has still not returned.

Observed over a 65-second window:

- first local execution height: `30276177`
- second local execution height: `30276761`
- execution head advance during window: `584` blocks
- first `safe_l2`: `30276177`
- second `safe_l2`: `30276762`
- `safe_l2` advance during window: `585` blocks
- first `unsafe_l2`: `30276177`
- second `unsafe_l2`: `30276762`
- `unsafe_l2` advance during window: `585` blocks
- public Shape Sepolia tip during second sample: `30426186`
- resulting local execution lag during second sample: `149425` blocks

Recent logs also stayed on the healthy pattern:

- `shape-sepolia-op-reth` continued logging repeated `Block added to canonical chain`
- `shape-sepolia-op-node-reth` continued logging repeated `Inserted new L2 unsafe block`
- recent logs did **not** show the old retry/stall markers:
  - `failed to insert unsafe payload`
  - `updated forkchoice, but node is syncing`
  - `failed to check for unsafe L2 blocks to sync`

So this lane should still be classified as **healthy catch-up**, not stalled and not fully caught up yet.

## Operator takeaway

If this Sepolia lane is retried again, do **not** start from the naive public-endpoint profile.

Start from the throttled L1 profile above, and judge success by:

1. local `eth_blockNumber` actually advancing
2. repeated `optimism_syncStatus` samples showing `current_l1` and L2 fields moving coherently
3. the unsafe payload retry loop disappearing

Until those happen, this lane is a useful proving environment, not proof that Sepolia Reth is fully healthy.
