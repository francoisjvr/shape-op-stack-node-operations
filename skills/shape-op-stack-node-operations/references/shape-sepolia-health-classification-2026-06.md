# Shape Sepolia health classification when head looks good but safe/finalized is stalled

This reference captures a deceptive Shape Sepolia state observed during live verification.

## Observed signals

Runtime on the VPS:
- `shape-sepolia-op-reth` and `shape-sepolia-op-node-reth` both up for weeks
- local `eth_syncing = false`
- local execution head advanced over repeated samples
- local head was only a couple of L2 blocks behind public `https://sepolia.shape.network`
- sampled canonical block by number matched the public block hash exactly

But rollup/L1 signals showed the lane was not fully healthy:
- `current_l1` was pinned at `10978200`
- local `head_l1` was `10978202`
- public Ethereum Sepolia L1 was `11046635`
- `safe_l2` and `finalized_l2` were stuck at `30754574`
- local unsafe/canonical head was around `31181378`
- safe lag was about `426804` L2 blocks

Recent `op-node` logs explained the stall:
- `429 Too Many Requests`
- Alchemy message: monthly capacity limit exceeded

## Interpretation

Do **not** classify the Sepolia node as healthy just because:
- containers are up
- `eth_syncing` is false
- `eth_blockNumber` is moving
- local canonical head is near public tip
- a sampled canonical block hash matches public

That combination only proves the execution side is still ingesting / tracking recent unsafe-canonical blocks.
It does **not** prove the rollup node is keeping L1 derivation current or that safe/finalized progression is intact.

The right classification for this pattern is:
- execution / unsafe head: healthy or near-healthy
- safe/finalized rollup status: unhealthy / stalled
- overall lane status: not fully healthy

## Minimal verification set

Check all of these before calling Shape Sepolia healthy:
1. `eth_blockNumber`
2. repeated `eth_blockNumber` after a short delay
3. `eth_syncing`
4. `optimism_syncStatus`
5. public Shape Sepolia `eth_blockNumber`
6. public Ethereum Sepolia L1 head
7. recent `op-node` logs for provider quota / 429 errors

## Operator rule

If unsafe/canonical head is near tip but `safe_l2`, `finalized_l2`, or `current_l1` are pinned far behind, treat the issue as an L1 derivation/provider problem first.

First suspect:
- exhausted L1 provider quota
- especially Alchemy monthly-capacity exhaustion on Sepolia

## Reporting rule for this user

Summaries should say explicitly:
- whether execution head is alive and near public tip
- whether safe/finalized is advancing
- whether the node is therefore fully healthy or only partially healthy

Do not flatten this into a single green/red answer without stating which side is degraded.
