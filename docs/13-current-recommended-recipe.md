# Current Recommended Recipe

This is the **operator-blunt recipe** we would reuse for the next clean Shape mainnet `op-reth` retry.

It is based on what actually worked on the current VPS, not just on the public docs.

Use this as:
- the quickest starting point for another operator
- a reproducible baseline for a fresh retry
- a reality-checked companion to the official Shape node docs

## Environment this was proven on

Current VPS:
- provider: **Contabo**
- hostname: `vmi3155969`
- virtualization: `KVM`
- OS: `Ubuntu 24.04.4 LTS`
- CPU: `6 vCPU`
- RAM: `17 GiB`
- disk: about `968G` root filesystem
- free disk at sample time: about `59G`

Important note:
- this is **not** Contabo-specific logic
- the same recipe should work on other VPS providers if they give you enough CPU, RAM, disk, and network stability
- what matters is the resource profile and the Shape-specific runtime details, not the Contabo brand itself

Practical sizing note:
- this exact box is enough to run the current parallel experiment
- disk pressure is real
- if someone is building a fresh node for themselves, more free SSD space than this is strongly preferable

## What this recipe assumes

- Shape mainnet
- `op-reth` as the execution client
- `op-node` as the consensus client
- an existing uploaded or otherwise prepared Reth datadir
- a desire to keep any existing geth fallback untouched until Reth proves itself

## Canonical paths

Use these paths:

- upload staging: `/root/shape-mainnet-op-reth-upload`
- runtime Reth data: `/root/shape-mainnet-op-reth-data`
- runtime Reth op-node data: `/root/shape-mainnet-op-node-reth-data`
- runtime config: `/root/.shape-mainnet-op-reth-config`

Reason:
- they keep the Reth track isolated
- they avoid confusing old experiment leftovers with the current runtime

## Versions that currently work

Pinned images from the live working parallel stack:

- `op-reth`: `us-docker.pkg.dev/oplabs-tools-artifacts/images/op-reth:v2.2.2`
- `op-node`: `us-docker.pkg.dev/oplabs-tools-artifacts/images/op-node:v1.18.0`

Do not casually swap versions without recording why.

## Config artifacts to preserve and use explicitly

Keep and reuse explicit config files from the upload or validated runtime set:

- `reth.toml`
- `rollup.json`
- `genesis-l2.json`
- optionally peer metadata such as `known-peers.json`

For this track, the working pattern is to use explicit runtime files rather than blindly trusting only built-in network shortcuts.

## Runtime choices that currently work

## op-reth

Current working `op-reth` runtime characteristics:

- uses `node`
- `--config=/config/reth.uploaded.toml`
- `--chain=/config/genesis-l2.runtime.json`
- `--datadir=/data`
- HTTP on `18545`
- WS on `18546`
- Engine/AuthRPC on `18551`
- P2P on `31303`
- `--rollup.sequencer=https://mainnet.shape.network`
- `--rollup.historicalrpc=https://mainnet.shape.network`
- `--rollup.disable-tx-pool-gossip`
- `--disable-discovery`

Why these matter:
- non-default ports avoid collisions with a live geth rollback stack
- explicit chain file avoids trusting possibly stale built-in chain support
- historical RPC is part of the working live recipe here
- discovery is disabled because execution-layer peer count is not the useful health signal on current Shape mainnet

## op-node

Current working `op-node` runtime characteristics:

- `--l1=<Ethereum mainnet RPC>`
- `--l1.rpckind=alchemy`
- `--l1.beacon=<Ethereum mainnet beacon RPC>`
- `--l2=http://127.0.0.1:18551`
- `--l2.jwt-secret=/shared/jwt.hex`
- `--rollup.config=/config/rollup.runtime.json`
- `--l2.enginekind=reth`
- `--syncmode=consensus-layer`
- `--rpc.addr=127.0.0.1`
- `--rpc.port=19545`
- `--metrics.enabled`
- `--metrics.port=17300`
- explicit Shape bootnodes
- P2P listen ports `19222` TCP/UDP

Why these matter:
- this is the combination that actually produced a live parallel Reth stack on this VPS
- `consensus-layer` is a meaningful departure from the public `op-geth`-centric example and should not be forgotten
- explicit bootnodes are useful and align with the public Shape troubleshooting guidance

## Recommended startup posture

If you are retrying cleanly:

1. Preserve the current geth fallback first.
2. Keep `/root/Upload` preserved if it contains expensive-to-replace source material.
3. Validate the uploaded Reth datadir structurally before startup.
4. If disk is tight, move the validated uploaded datadir into `/root/shape-mainnet-op-reth-data` instead of copying it.
5. Snapshot config artifacts into `/root/.shape-mainnet-op-reth-config`.
6. Remove stale lock files only after confirming no Reth process is using the data.
7. Start `op-reth` with isolated ports and explicit runtime chain/config files.
8. Start `op-node` against the Reth engine with `--l2.enginekind=reth` and `--syncmode=consensus-layer`.
9. Judge health by repeated block-number samples, not by whether the process merely started.

## What to watch after startup

The minimum useful checks are:

- `eth_blockNumber` from `op-reth`
- public Shape `eth_blockNumber`
- lag in decimal blocks
- `eth_syncing`
- `optimism_syncStatus`
- whether `safe_l2` and `unsafe_l2` move together with real execution-head movement

Current important nuance:
- `eth_syncing=false` does **not** automatically mean the whole stack is caught up or ready

## What not to overreact to

## 1. Zero EL peers

On current Shape mainnet:
- zero execution peers is not the main failure signal
- there are currently no EL bootnodes to chase
- do not waste time treating peer-hunting as the first debugging step

## 2. Early flat periods

The first runtime attempt showed:
- the execution head can look flat for a while
- then resume forward movement later

So do not call it dead too quickly.
Measure repeated samples before making the call.

## 3. Snapshot optimism

Public docs make snapshot startup sound smoother than what this Reth migration track actually felt like.
Treat imported-snapshot bring-up as potentially awkward, not guaranteed instant success.

## What is different from the public Shape docs

The public Shape docs are still the canonical public starting point.
But for this exact Reth track, the practical deltas are:

- public docs are centered on `op-geth`; this recipe is for `op-reth`
- public docs show source builds; this recipe uses pinned Docker images
- public docs use default ports; this recipe avoids collisions with an existing geth stack
- public docs show `execution-layer` sync for `op-node`; this recipe currently works with `consensus-layer`
- public docs rely more on public CLI examples; this recipe prefers explicit runtime config artifacts

See also:
- `docs/12-current-working-runtime-vs-official-docs.md`

## Bottom line

If someone wants to try building their own Shape mainnet Reth node from this repo, the current best advice is:

- use a VPS with comfortable SSD headroom
- do not depend on default ports if another node is already present
- pin the currently working images first
- use explicit runtime config files
- use `op-node` with `--l2.enginekind=reth` and `--syncmode=consensus-layer`
- judge success by shrinking lag against live Shape head, not by peer count or startup logs alone
