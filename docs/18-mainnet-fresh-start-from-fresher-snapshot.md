# Mainnet Fresh-Start Rebuild Plan From a Fresher Snapshot

This is the plan for the **next** Shape mainnet retry after the stalled branch was torn down.

## Current hold state

Right now the operator posture is intentionally simple:
- all mainnet Shape node containers are removed
- old mainnet snapshot/data directories are deleted
- the node stays **off** until a fresher snapshot is available and the operator asks for a new bring-up

Do **not** try to revive the old stalled datadir.

## Goal

Bring the node back from a clean base using a newer snapshot, while keeping the L1 provider path boring and predictable.

## L1 provider rule

Keep `op-node` pointed at the stable local loopback URLs:
- execution: `http://127.0.0.1:18580`
- beacon: `http://127.0.0.1:18581`

Back that local multiplexer with the **two fresh paid keys first**.
Do not put the old near-exhausted key back in front.

## Canonical paths to recreate when the new snapshot is ready

- snapshot staging: `/root/shape-mainnet-op-reth-upload`
- runtime Reth data: `/root/shape-mainnet-op-reth-data`
- runtime op-node data: `/root/shape-mainnet-op-node-reth-data`
- runtime config: `/root/.shape-mainnet-op-reth-config`

## Clean restart sequence

### 1. Confirm the new snapshot is actually newer

Before downloading anything:
- record the published snapshot timestamp
- record the published snapshot block height if available
- make sure it is newer than the failed branch we just discarded

If that proof is weak, wait.

### 2. Recreate only the canonical directories

Create fresh empty paths:
- `/root/shape-mainnet-op-reth-upload`
- `/root/shape-mainnet-op-reth-data`
- `/root/shape-mainnet-op-node-reth-data`
- `/root/.shape-mainnet-op-reth-config`

Do not mix in old canary paths, old backups, or old pre-reseed directories.

### 3. Stage the new snapshot cleanly

Use the clean staging path for the new archive/download.
Prefer a resumable download flow.
Do not reuse the deleted archive state as if it were trustworthy.

### 4. Extract offline and validate the datadir

Before any startup:
- finish the extract fully
- verify the extracted tree looks like a real Reth datadir
- check ownership and permissions
- make sure no stale lock file or old process is attached

### 5. Rebuild config from clean source artifacts

Populate `/root/.shape-mainnet-op-reth-config` with the runtime files you actually intend to use:
- `reth.uploaded.toml`
- `genesis-l2.runtime.json`
- `rollup.runtime.json`
- `jwt.hex`
- any validated peer metadata if still needed

Generate a fresh JWT if there is any doubt about the old one.

### 6. Rebuild the L1 multiplexer first

Before touching `op-reth` or `op-node`:
- configure the multiplexer to serve `127.0.0.1:18580` and `127.0.0.1:18581`
- put the two fresh paid keys first
- leave old/near-exhausted paths behind the fresh keys or out of active rotation
- verify the proxy path itself before calling the provider side ready

Minimum proof:
- multiplexer service is active
- proxied execution request returns `200`
- proxied beacon request returns `200`
- health output shows the expected fresh-key labels

### 7. Render config before startup

Use:
- `examples/.env.example`
- `examples/docker-compose.recommended.yml`

The next `.env` should keep:
- `L1_RPC_URL=http://127.0.0.1:18580`
- `L1_BEACON_URL=http://127.0.0.1:18581`

### 8. Start only `op-reth` first

Do not start both services together.

First prove:
- container comes up cleanly
- `eth_blockNumber` answers on `127.0.0.1:18545`
- `eth_syncing` answers
- logs look alive instead of instantly replaying the old dead pattern

### 9. Start `op-node` second

Only after `op-reth` is answering locally:
- start `op-node`
- verify JWT wiring
- verify `optimism_syncStatus` answers on `127.0.0.1:19545`
- verify the reset-loop signature does not return immediately

### 10. Use boring proof, not vibes

Do not call it healthy because the containers started.
Require:
- repeated decimal `eth_blockNumber` samples
- the next canonical block becoming queryable
- shrinking lag versus public Shape
- `optimism_syncStatus` moving in a sane way

## First things to avoid on the next retry

- do not reuse the old stalled datadir
- do not restart `op-node` early just because a timer expired
- do not treat `head=0` or a flat head over a short window as the whole story
- do not move back to a direct remote beacon URL when the loopback multiplexer is available
- do not put the old near-exhausted key back in front of the two fresh keys

## Practical repo files for the next bring-up

- `docs/13-current-recommended-recipe.md`
- `docs/15-copy-paste-bring-up-checklist.md`
- `examples/.env.example`
- `examples/docker-compose.recommended.yml`
- `skills/shape-op-stack-node-operations/references/alchemy-key-rotation-multiplexer-cutover-2026-06.md`

## Bottom line

The next retry should be a **clean rebuild from a fresher snapshot**, with the **local loopback multiplexer** in front of the **two fresh paid keys**, and with `op-reth` proven alive **before** `op-node` is allowed back in.
