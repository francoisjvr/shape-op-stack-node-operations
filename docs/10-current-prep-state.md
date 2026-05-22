# Current Prep State

This file records the server-side Reth prep that was completed **without starting any Reth services**.

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

Clean canonical base paths:
- `/root/shape-mainnet-op-reth-upload`
- `/root/shape-mainnet-op-reth-data`
- `/root/shape-mainnet-op-node-reth-data`
- `/root/.shape-mainnet-op-reth-config`

## What was deliberately not touched

- `/root/Upload`
- current geth runtime
- current geth JWT
- current geth/op-node containers

## Why this matters

The next upload should land in a known-clean staging area instead of inheriting ambiguous leftovers from prior Reth experiments.

That reduces confusion around:
- stale chain-spec artifacts
- stale JWT/config leftovers
- stale op-node data
- accidentally blaming new data for old experimental residue

## Next step after upload finishes

When the new Reth data upload is complete:
- inspect the uploaded structure
- verify size and completeness sanity
- decide whether to run from staged data or copy into the canonical runtime datadir
- finalize chain-spec and config files in `/root/.shape-mainnet-op-reth-config`
- only then begin the actual Reth bring-up sequence
