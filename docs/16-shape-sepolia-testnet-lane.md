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

Even after the better L1 setup, the local execution head stayed flat during repeated checks:

- local `eth_blockNumber` stayed at `30141035`

At the same time, logs showed `op-node` repeatedly trying to feed a newer unsafe payload into `op-reth`:

- repeated target payload around `30406381`
- repeated warning/error shape:
  - `failed to insert unsafe payload`
  - `updated forkchoice, but node is syncing`
  - `failed to check for unsafe L2 blocks to sync`

`op-reth` also kept logging the same received new payload while its own reported latest block remained flat.

## Current interpretation

This test lane is in a **better-than-broken but not-yet-healthy** state:

- good:
  - isolated bring-up works
  - mainnet remained untouched
  - snapshot artifact is present
  - `op-node` is deriving and moving through L1 history
  - public-RPC throttling clearly improved behavior
- not good enough yet:
  - local L2 execution head has not resumed forward movement
  - unsafe payload insertion is repeating while `op-reth` still reports itself syncing

So this lane should be treated as:

- **successful isolation and partial bring-up**
- **successful observability finding**
- **not yet a clean healthy Sepolia convergence result**

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

## Operator takeaway

If this Sepolia lane is retried again, do **not** start from the naive public-endpoint profile.

Start from the throttled L1 profile above, and judge success by:

1. local `eth_blockNumber` actually advancing
2. repeated `optimism_syncStatus` samples showing `current_l1` and L2 fields moving coherently
3. the unsafe payload retry loop disappearing

Until those happen, this lane is a useful proving environment, not proof that Sepolia Reth is fully healthy.
