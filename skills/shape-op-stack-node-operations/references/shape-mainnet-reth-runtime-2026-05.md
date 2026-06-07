# Shape Mainnet Reth Runtime Notes — 2026-05

## Validated live runtime

Checked against the working server after Reth restoration.

- `op-reth` image: `us-docker.pkg.dev/oplabs-tools-artifacts/images/op-reth:v2.2.2`
- `op-node` image: `us-docker.pkg.dev/oplabs-tools-artifacts/images/op-node:v1.18.0`
- `op-node --syncmode=consensus-layer`
- `op-node --l2.enginekind=reth`
- current ports in use: `18545`, `18546`, `18551`, `19545`, `17300`, `19222`
- current runtime paths:
  - `/root/shape-mainnet-op-reth-data`
  - `/root/shape-mainnet-op-node-reth-data`
  - `/root/.shape-mainnet-op-reth-config`
- runtime config files in play:
  - `reth.uploaded.toml` / runtime equivalent
  - `rollup.runtime.json`
  - `genesis-l2.runtime.json`
  - shared `jwt.hex`

## Health signals that mattered

Do not treat "container is up" as success. Confirm all of:

- `eth_blockNumber` advances over repeated samples
- `eth_syncing` returns `false`
- `optimism_syncStatus` shows sane `unsafe_l2`, `safe_l2`, and `finalized_l2`
- recent `op-node` logs show continued unsafe block insertion / payload processing
- latest local block hash matches public Shape over repeated checks when possible

Shape-specific nuance:

- `net_peerCount = 0` on the execution client can still be expected on current Shape mainnet
- lagging `safe_l2` or `finalized_l2` behind `unsafe_l2` is not by itself failure

## Documentation / publishing lessons

Three different states must be distinguished explicitly:

1. local file edited
2. local git commit created
3. remote GitHub updated

Do not collapse those into one sentence like "the repo is updated" unless the remote really has the commit.

## Repo-scope lesson

There were two doc targets during this work:

- upstream Shape docs repos (`shape-network/docs`) — local edits were possible, push access was not
- user-owned repo (`francoisjvr/shape-mainnet-node-runbook`) — push access existed and was the correct publication target

When in doubt, verify push scope before promising that GitHub itself is up to date.

## User workflow preference reinforced by this session

For obvious cleanup work inside a current class of task, do not ask whether to finish the cleanup. Finish it and report the result. In this session the right move was to fully remove lingering geth framing from the Shape node skill instead of asking first.
