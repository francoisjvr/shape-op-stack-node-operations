# Post-reseed flat-head diagnostic signals (June 2026)

Use this after a **fresh official snapshot reseed** when `op-reth` and `op-node` both restart cleanly but the lane still does not actually recover.

## The deceptive pattern

A clean reseed can still land on the same bad state:
- containers start normally
- local RPC answers
- `safe_l2` is nonzero again
- `op-node` does not look obviously dead
- but canonical execution head stays flat across repeated samples

In the observed Shape mainnet case after reseeding from the latest downloaded official Reth snapshot:
- local head stayed pinned at `29403655`
- public head was `29783741`
- lag remained `380086`
- `MerkleChangeSets` stayed pinned at `26385482`
- `safe_l2` was nonzero (`29403061`) but flat
- the former missing target block `29777771` was still not queryable

## Operator interpretation

Do **not** classify a reseed as healthy just because:
- extraction succeeded
- services restarted
- RPC came back
- `safe_l2` repopulated from zero

If all of the following are true together, treat the reseed as still unhealthy and likely still in the same canonical-import failure family:
- repeated decimal `eth_blockNumber` samples are flat
- public Shape head keeps moving away
- `MerkleChangeSets` remains pinned far behind the canonical head
- `safe_l2` is present but not advancing
- a previously missing canonical block is still absent by number

## Minimum post-reseed proof checklist

Require all of these before calling the lane recovered:
1. repeated decimal local head samples increase
2. local/public lag shrinks rather than stays flat
3. `safe_l2` or `finalized_l2` advances over repeated samples
4. `MerkleChangeSets` advances if it was previously pinned far behind
5. the formerly missing canonical block becomes queryable by number/hash

## Reporting rule for this user

Split the result into two layers:
- `reseed/extract/restart succeeded`
- `node health still failed` or `node health recovered`

That prevents a technically true but operationally misleading report like "the restart worked" when the chain is still pinned.

## Lightweight watcher pattern

For unattended recovery monitoring, keep a background watcher running after reseed that auto-reports on any of:
- local head movement
- `safe_l2` / `finalized_l2` movement
- `MerkleChangeSets` movement
- the first missing canonical block becoming queryable
- timeout with no movement

This is more useful than watching container uptime alone.
