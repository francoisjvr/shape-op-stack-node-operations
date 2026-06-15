# Post-reseed trie stall and repair-trie confirmation — 2026-06

Use this note when a Shape mainnet `op-reth` lane still shows the deceptive canonical-stall pattern even after a fresh snapshot reseed.

## Observed sequence

1. Fresh snapshot restored cleanly enough for `op-reth` RPC to come up.
2. `op-node` resumed feeding unsafe payloads near network tip.
3. Canonical execution head stayed pinned at the snapshot tip (`29403655` in this incident).
4. `eth_getBlockByNumber(head + 1)` remained `null`.
5. `optimism_syncStatus` showed `unsafe_l2` near tip, while `safe_l2` and `current_l1` stayed at `0`.
6. `eth_syncing` stayed flat, with `MerkleChangeSets` lagging far behind (`26385482` here) while `Execution`/`Finish` stayed parked at the same snapshot-tip block.
7. `op-reth` logs kept printing `Received new payload from consensus engine ...`, but periodic status lines still showed `latest_block=29403655`.

## Decision rule

Do **not** keep iterating on restart-only checks once all of the following are true after reseed:
- `op-reth` accepts new payloads
- canonical `eth_blockNumber` is flat
- `head + 1` is missing by number
- `eth_syncing` late-stage checkpoints are flat

At that point, stop the lane and run offline trie diagnosis on the preserved live datadir.

## Confirmed dry-run signal

A decisive dry-run signal was:
- `db --datadir /data repair-trie --dry-run --chain /config/genesis-l2.runtime.json --color never`
- repeated `WARN Inconsistency found: StorageMissing(...)`

That output is enough to justify the live offline `repair-trie` pass against the same datadir and chainspec.

## Safe execution order

1. Stop `shape-mainnet-op-node-reth` first.
2. Stop `shape-mainnet-op-reth` second.
3. Run `repair-trie --dry-run` against the live datadir with the explicit runtime config bind-mounted read-only.
4. If dry-run reports `StorageMissing(...)` / trie inconsistencies, run the same command without `--dry-run`.
5. Only after repair completes:
   - start `shape-mainnet-op-reth`
   - wait for RPC readiness on `127.0.0.1:18545`
   - start `shape-mainnet-op-node-reth`
   - verify canonical head movement, `head+1` availability, and shrinking lag vs public Shape

## Operational notes

- Treat the reseeded datadir as worth repairing before discarding it again; a clean reseed can still carry a trie inconsistency worth fixing in place.
- `repair-trie` progress can be bursty and misleading early on. In this incident it moved from low single digits to ~40% with many more `StorageMissing(...)` findings surfacing mid-run; do not assume the first ETA is stable.
- Keep the verification watcher strict: unsafe-payload flow alone is not recovery.
