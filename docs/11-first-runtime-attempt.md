# First Runtime Attempt

## Goal

Start Shape mainnet `op-reth` plus `op-node` in parallel against the uploaded Reth datadir without touching the live geth rollback stack.

## Date/time window

- start: `2026-05-22 16:09 UTC`
- latest sample in this note: `2026-05-22 16:14 UTC`

## Runtime versions

- op-reth: `us-docker.pkg.dev/oplabs-tools-artifacts/images/op-reth:v2.2.2`
- op-node: `us-docker.pkg.dev/oplabs-tools-artifacts/images/op-node:v1.18.0`

## Data source

- runtime Reth datadir: `/root/shape-mainnet-op-reth-data`
- preserved uploaded originals in config dir:
  - `/root/.shape-mainnet-op-reth-config/reth.uploaded.toml`
  - `/root/.shape-mainnet-op-reth-config/rollup.uploaded.json`
  - `/root/.shape-mainnet-op-reth-config/genesis-l2.uploaded.json`
  - `/root/.shape-mainnet-op-reth-config/known-peers.uploaded.json`

## Paths used

- live geth rollback datadir: `/root/Upload`
- parallel Reth runtime datadir: `/root/shape-mainnet-op-reth-data`
- parallel Reth op-node datadir: `/root/shape-mainnet-op-node-reth-data`
- runtime config dir: `/root/.shape-mainnet-op-reth-config`

## Ports used

- live geth HTTP: `8545`
- live geth WS: `8546`
- live geth authrpc: `8551`
- live op-node RPC: `9545`
- parallel op-reth HTTP: `18545`
- parallel op-reth WS: `18546`
- parallel op-reth authrpc: `18551`
- parallel op-reth P2P: `31303`
- parallel op-node RPC: `19545`
- parallel op-node metrics: `17300`
- parallel op-node P2P: `19222`

## Starting health baseline

- local geth fallback head before first Reth start: `28860665`
- public Shape head before first Reth start: `28860661`
- root free disk before first Reth start: about `59G`

## Important config finding before startup

Built-in `op-reth` chain `shape` in `v2.2.2` does **not** expose the current Shape fork set we need.

Observed from `dump-genesis --chain shape`:
- `graniteTime: 1727370000`
- `holoceneTime: null`
- `isthmusTime: null`
- `jovianTime: null`

Observed from the uploaded datadir files:
- uploaded `genesis-l2.json` and `rollup.json` included:
  - `granite: 1727370000`
  - `holocene: 1740139200`
  - `isthmus: 1774530000`
  - `jovian: 1778157000`

Observed from the currently working geth/op-node stack:
- explicit overrides in live runtime were:
  - `granite: 1727370000`
  - `holocene: 1739880000`
  - `isthmus: 1774530000`
  - `jovian: 1778157001`

Decision taken for this first runtime attempt:
- do **not** trust built-in `--chain=shape` alone
- do **not** use uploaded fork times verbatim without adjustment
- create runtime files that mirror the currently working live overrides:
  - `/root/.shape-mainnet-op-reth-config/genesis-l2.runtime.json`
  - `/root/.shape-mainnet-op-reth-config/rollup.runtime.json`

Runtime config checksums:
- `genesis-l2.runtime.json`: `d3dec0f6e4b11d84393c75535948077cd31121c044ac99c885e44e2550b3f21a`
- `rollup.runtime.json`: `1dfabd08d04d1560684d4290bb535943649e36ed6b2c96ef254db726fb176ec2`

## Actions taken

1. Captured live baseline:
   - geth container healthy
   - op-node container healthy
   - root free space about `59G`
   - geth head `28860665`
   - public Shape head `28860661`
2. Inspected live geth/op-node docker args to reuse JWT path, L1 RPCs, and known-good Shape overrides.
3. Confirmed built-in `op-reth --chain shape` lacked `holocene`, `isthmus`, and `jovian` in `dump-genesis` output.
4. Wrote runtime config files in `/root/.shape-mainnet-op-reth-config` using the live working override values.
5. First `op-reth` start failed because `--disable-discovery` and `--disable-dns-discovery` cannot be combined.
6. Second `op-reth` start failed because port `30303` was already occupied by the live geth container.
7. Fixed first-attempt recipe by:
   - removing `--disable-dns-discovery`
   - changing parallel op-reth P2P port to `31303`
8. Started parallel `op-reth` successfully.
9. Verified parallel `op-reth` served RPC on `http://127.0.0.1:18545`.
10. Started parallel `op-node` against:
   - `--l2=http://127.0.0.1:18551`
   - `--rollup.config=/config/rollup.runtime.json`
   - `--l2.enginekind=reth`
11. Sampled `eth_blockNumber`, `eth_syncing`, and `optimism_syncStatus` repeatedly.

## Final first-attempt runtime recipe

### op-reth

Key runtime choices:
- `--config=/config/reth.uploaded.toml`
- `--chain=/config/genesis-l2.runtime.json`
- `--datadir=/data`
- `--http.port=18545`
- `--ws.port=18546`
- `--authrpc.port=18551`
- `--port=31303`
- `--rollup.sequencer=https://mainnet.shape.network`
- `--rollup.historicalrpc=https://mainnet.shape.network`
- `--rollup.disable-tx-pool-gossip`
- `--disable-discovery`

### op-node

Key runtime choices:
- `--rollup.config=/config/rollup.runtime.json`
- `--l2=http://127.0.0.1:18551`
- `--l2.enginekind=reth`
- `--syncmode=consensus-layer`
- `--rpc.port=19545`
- `--p2p.listen.tcp=19222`
- `--p2p.listen.udp=19222`

## Observed head movement

Repeated samples showed:
- parallel op-reth execution head: `28556991`
- later parallel op-reth execution head: `28556991`
- live geth head at same point: `28860947`
- public Shape head at same point: `28860996`

Current observed execution lag:
- versus live geth: `303956` blocks
- versus public Shape RPC: `304005` blocks

So the parallel execution head was **flat** during this observation window.

## Observed `eth_syncing`

`op-reth` returned a syncing object, but it was not advancing during the sample window:
- `startingBlock = currentBlock = highestBlock = 28556991`
- stage `MerkleChangeSets` lagged behind at `26319946`
- other stages reported `28556991`

This means the node is alive and serving RPC, but the execution head did not actually move yet.

## Observed `optimism_syncStatus`

Early sample:
- most L2 fields were still zeroed while the node was just coming up

Later sample:
- `unsafe_l2.number = 28556991`
- `safe_l2.number = 28556085`
- `current_l1.number` advanced from `25101207` to `25101316`
- `head_l1.number` was around `25151922`

So `op-node` is doing derivation work and advancing L1 origin tracking, but that has **not** translated into execution head movement yet.

## Logs worth keeping

First failure:
- `the argument '--disable-discovery' cannot be used with '--disable-dns-discovery'`

Second failure:
- `address 0.0.0.0:30303 (listener service) is already in use (os error 98)`

Important live error after startup:
- `failed to insert unsafe payload`
- `err: updated forkchoice, but node is syncing`

Repeated pattern from op-node:
- requesting engine missing unsafe L2 block range from `28556991` toward about `28860925`
- failing because the engine still considers itself syncing

## Outcome classification

- **alive but not converging yet**

More specifically:
- this is **not** a clean startup/config crash anymore
- this is **not** healthy yet
- this is **not** enough to claim success
- the main blocker is still flat execution head while op-node tries to feed missing unsafe payloads

## Current running state

As of this note:
- `shape-mainnet-op-reth` is running
- `shape-mainnet-op-node-reth` is running
- live geth stack remains untouched in parallel

## Next decision

- continue investigating why the engine stays in syncing state at `28556991`
- focus on why `op-reth` serves the snapshot but does not advance execution while op-node requests the missing unsafe range
- avoid calling this healthy until execution head moves in decimal over repeated samples
