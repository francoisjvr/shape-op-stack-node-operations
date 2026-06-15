# Reth reseed after repeated stalled-recovery attempts

Use this note when the Shape mainnet Reth lane has already consumed time on unwind / `repair-trie` / restart experiments and a fresh snapshot is already fully present on disk.

## Trigger

Switch to a clean reseed when these line up:
- canonical `eth_blockNumber` stays pinned at the same decimal height across repeated samples
- the first missing canonical block is still absent
- a deep unwind replays back into the same bad head
- `repair-trie` does not produce timely canonical recovery, or the user explicitly says to stop messing around and reseed
- a newer snapshot tarball is already fully downloaded and verified clean enough to use

## Decision rule

Do **not** keep iterating on subtle DB repair paths once the cheaper operator decision is obvious: use the latest snapshot and preserve notes.

For this user, prefer the low-friction recovery path when a current snapshot is already available.

## Pre-flight

1. Keep a timestamped notes directory such as `/root/shape-node-backups/reseed-<timestamp>/`.
2. Save:
   - `meta.json`
   - `docker inspect` for `shape-mainnet-op-reth` and `shape-mainnet-op-node-reth`
   - a copy of the active config directory snapshot
3. Stop or kill any background watcher from abandoned repair attempts so it does not restart containers mid-reseed.
4. Remove any temporary repair container left behind by earlier offline work.
5. Kill stale archive-inspection readers against the same tarball so the real extract gets disk and CPU priority.
   - When doing this over SSH from a wrapper that itself mentions the archive path, avoid broad `pkill -f` patterns that can match your live orchestration command and kill the session you are using to recover the node.
   - Prefer listing candidate PIDs first and killing only the confirmed stale readers.

## Canonical reseed sequence

1. Keep non-target lanes (for example sepolia) untouched.
2. Intentionally stop the mainnet `op-node` and `op-reth` containers.
3. Preserve the old datadir according to available disk and operator intent.
4. Extract the latest snapshot into the canonical runtime datadir using the real archive, not a copy-paste path guess.
5. Start `op-reth` first and wait for RPC readiness.
6. Start `op-node` only after Engine API readiness is confirmed.
7. Watch recovery until the following are true:
   - canonical decimal head advances beyond the old stuck head
   - the first previously missing canonical block becomes queryable
   - `safe_l2` / `current_l1` are no longer pinned at zero

## Verification nuance

A successful reseed is not just "containers are up". Require block movement and queryability.

Good post-reseed checks:
- local `eth_blockNumber`
- `eth_getBlockByNumber` for the first formerly missing height
- `optimism_syncStatus`
- recent `op-node` logs for resumed derivation
- recent `op-reth` logs for import progress

## Notes discipline

Keep the notes directory even when the reseed works immediately. Minimum useful contents:
- timestamp
- snapshot filename and byte size
- whether the `.aria2` sidecar was gone
- disk free before extraction
- preserved backup paths
- background session IDs / watcher IDs that were stopped
- exact verification head before and after

## Session-specific example

In this session the productive move was:
- stop the old repair watcher
- remove the leftover repair container
- stop two archive-inspection probes that were still reading the tarball
- begin extracting the latest `reth-20260603100001.tar.gz` snapshot into the live Shape mainnet path
- keep notes under a fresh `reseed-...` directory while mainnet stayed intentionally down during the extract
