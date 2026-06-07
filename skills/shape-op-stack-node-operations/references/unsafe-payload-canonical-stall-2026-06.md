# Unsafe payloads advancing while canonical head stays stuck

Use this reference when a Shape `op-reth` + `op-node` lane looks superficially alive because unsafe payloads keep arriving, but the canonical execution head never moves.

## Observed signature

Live pattern from the June 2026 investigation:

- `op-node` repeatedly logs:
  - `Inserting unsafe L2 execution payload to drive EL sync`
  - `Inserted new L2 unsafe block (synchronous)`
- `op-reth` logs matching:
  - `Received new payload from consensus engine`
- `unsafe_l2` keeps increasing
- but local canonical RPC stays pinned:
  - `eth_blockNumber` flat at `29396633`
  - `eth_getBlockByNumber(29396634)` returned `null`
  - `eth_getBlockByHash(<recent unsafe payload hash>)` returned `null`
  - `pending` returned the same old canonical block instead of a newer in-flight block

This means payload delivery is happening, but canonical import is not.

## Strongest DB clue

`eth_syncing` showed nearly every stage at the head except one:

- `Execution`: `29396633`
- `Headers`: `29396633`
- `Bodies`: `29396633`
- `Finish`: `29396633`
- `MerkleExecute`: `29396633`
- `MerkleChangeSets`: `26385482`

Gap from head to `MerkleChangeSets` in that case: `3011151` blocks.

A repeated resample showed the same `MerkleChangeSets` value, so it was not just a transient lagging read.

## Root-cause hypothesis to prefer first

When the above pattern appears, prefer this hypothesis before blaming `op-node` wiring:

- trie / DB consistency issue inside the `op-reth` datadir
- payloads accepted through the engine path are not being promoted into canonical chain state

This hypothesis gained weight because an earlier non-destructive dry run against the same datadir reported repeated trie inconsistencies such as `StorageMissing(...)` from:

- `op-reth db --datadir /data repair-trie --dry-run --chain /config/genesis-l2.runtime.json`

## Minimal verification sequence

1. Compare local `eth_blockNumber` across repeated samples.
2. Query the first missing canonical height directly with `eth_getBlockByNumber(head+1, false)`.
3. Query one recent unsafe payload hash with `eth_getBlockByHash(hash, false)`.
4. Query `pending` and check whether it is genuinely ahead or just mirrors the old canonical head.
5. Inspect `eth_syncing` and note whether `MerkleChangeSets` is uniquely far behind.
6. Run `op-reth db --datadir /data stage-checkpoints get --chain /config/genesis-l2.runtime.json` to confirm the same lag directly from the DB.
7. Run `op-reth db --datadir /data repair-trie --dry-run --chain /config/genesis-l2.runtime.json` as the safest confirmation step before any destructive repair.

## Operator interpretation

Do **not** say the node is recovering just because:

- `unsafe_l2` is climbing
- `op-node` says unsafe insertion succeeded
- `op-reth` says it received new payloads

Only call it real recovery when both happen:

- canonical decimal `eth_blockNumber` advances
- the previously missing canonical block becomes queryable by number/hash

## Extra sync-status nuance seen in the same stall

In the investigated state, `optimism_syncStatus` also looked degraded in a specific way:

- `unsafe_l2` far ahead of local canonical head
- `safe_l2 = 0`
- `finalized_l2 = 0`
- `current_l1 = 0`

That combination is another hint that only the unsafe ingestion side is alive.
