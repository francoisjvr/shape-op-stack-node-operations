# Current Prep State

This file records the server-side Reth prep that was completed **without starting any Reth services**.

## Current state summary

The VPS is no longer just at pre-upload baseline.

It is now at **post-upload prepared state**:
- uploaded Reth datadir was inspected and looked structurally real
- uploaded datadir was moved into canonical runtime path `/root/shape-mainnet-op-reth-data`
- upload staging path `/root/shape-mainnet-op-reth-upload` was recreated empty
- uploaded config artifacts were copied into `/root/.shape-mainnet-op-reth-config`
- stale lock files from the source datadir were removed before first startup
- no Reth services have been started yet

## What was removed

Old Reth-only experiment paths removed:
- `/root/.shape-mainnet-reth-fresh-config`
- `/root/shape-mainnet-reth-fresh`
- `/root/shape-mainnet-reth-fresh-opnode-data`

Old ad hoc Reth prep/extract scripts and logs removed:
- `finish_reth_extract_only.*`
- `prepare_shape_reth_snapshot.*`
- `resume_reth_download_extract.*`
- `retry_reth_extract.*`

## What now exists

Canonical base paths:
- `/root/shape-mainnet-op-reth-upload`
- `/root/shape-mainnet-op-reth-data`
- `/root/shape-mainnet-op-node-reth-data`
- `/root/.shape-mainnet-op-reth-config`

Runtime/path meaning now:
- `/root/shape-mainnet-op-reth-data` contains the validated uploaded Reth datadir
- `/root/shape-mainnet-op-reth-upload` is empty again and reserved for future staging

Observed sizes at rename time:
- `/root/shape-mainnet-op-reth-data`: about `114G`
- `/root/shape-mainnet-op-reth-upload`: `4.0K` after recreation
- root free space remained about `59G`, so copying the datadir was intentionally avoided

Top-level runtime datadir contents:
- `blobstore/`
- `db/`
- `static_files/`
- `invalid_block_hooks/`
- `reth.toml`
- `rollup.json`
- `genesis-l2.json`
- `known-peers.json`

Config snapshots copied into `/root/.shape-mainnet-op-reth-config`:
- `reth.uploaded.toml`
- `rollup.uploaded.json`
- `genesis-l2.uploaded.json`
- `known-peers.uploaded.json`

Config snapshot checksums:
- `reth.uploaded.toml`: `4d9846d6d8f9a41b7787072e167288c4ddbc96c90aba7074247ad65c571ab6ab`
- `rollup.uploaded.json`: `ec63d2ca50665eb574a945efdde101dc13088985318af03d68571615b2ccff16`
- `genesis-l2.uploaded.json`: `879ee4132d2bd89c7a8af6c3cb3c5c3dfabf5ba7ddfc379a6ea876982aa1a041`
- `known-peers.uploaded.json`: `b7fc9db656a340839b51ec4f0aec5191a5bdc8e71a42055fb8b93b3c8c003e00`

Lock cleanup completed:
- removed `/root/shape-mainnet-op-reth-data/db/lock`
- removed `/root/shape-mainnet-op-reth-data/db/mdbx.lck`
- removed `/root/shape-mainnet-op-reth-data/static_files/lock`

## What was deliberately not touched

- `/root/Upload`
- current geth runtime
- current geth JWT
- current geth/op-node containers

## Why this matters

The upload is now sitting in the canonical runtime path without wasting another ~114G on a duplicate copy.

The next upload can still land in a known-clean staging area instead of inheriting ambiguous leftovers from prior Reth experiments.

That reduces confusion around:
- stale chain-spec artifacts
- stale JWT/config leftovers
- stale op-node data
- accidentally blaming new data for old experimental residue

## Next step from here

Before first Reth startup:
- inspect the uploaded `reth.toml` and decide what to keep versus override
- finalize explicit chain/config files in `/root/.shape-mainnet-op-reth-config`
- define the exact `op-reth` and `op-node` launch recipe against `/root/shape-mainnet-op-reth-data`
- compare first runtime execution head samples against current geth and public Shape RPC
- only then begin the actual Reth bring-up sequence

That first runtime attempt has now been executed and is recorded in `docs/11-first-runtime-attempt.md`.
