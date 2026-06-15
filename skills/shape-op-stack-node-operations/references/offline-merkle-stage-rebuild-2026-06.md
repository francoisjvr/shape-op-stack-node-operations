# Offline targeted Merkle stage rebuild after reseed/repair stalls

Use this when the Shape mainnet Reth lane still shows the canonical-stall signature **without** strong fresh trie-damage evidence on the current datadir.

## When this path fit

Observed decision pattern from the June 2026 incident:
- `op-node` provider / mode experiments did not unstick canonical import
- `repair-trie` had already been tried on the same datadir lineage
- the lane still presented the familiar proof set:
  - decimal `eth_blockNumber` pinned
  - public Shape head advancing away
  - `op-node` / RPC unhealthy or exited after failed restart attempts
  - `stage-checkpoints` showed `Execution`, `Finish`, and `MerkleExecute` at about the same high head
  - **only** `MerkleChangeSets` remained far behind

Concrete checkpoint shape from the incident:
- `Execution`: `29,403,858`
- `Finish`: `29,403,858`
- `MerkleExecute`: `29,403,858`
- `MerkleChangeSets`: `26,385,482`

Interpretation:
- this is a strong hint that the least-destructive next repair is an **offline Merkle stage rebuild** on the preserved datadir, rather than another mode flip or an immediate full datadir wipe.

## Operator rule

For Morpheus, prefer the lowest-friction non-destructive step first:
1. stop the mainnet lane
2. inspect `stage-checkpoints`
3. if `MerkleChangeSets` is the lone major laggard while the other execution checkpoints sit near the same tip, try a targeted offline Merkle stage rebuild
4. only escalate to destructive datadir replacement when this path is exhausted or contradicted by new evidence

## Command shape

Run against the stopped preserved datadir, using the live runtime config mount:

```bash
op-reth stage run \
  --datadir /data \
  --chain /config/genesis-l2.runtime.json \
  --from <MerkleChangeSets checkpoint> \
  --to <Execution/Finish checkpoint> \
  --batch-size 5000 \
  --commit \
  --checkpoints \
  merkle
```

Incident-tested example:

```bash
op-reth stage run \
  --datadir /data \
  --chain /config/genesis-l2.runtime.json \
  --from 26385482 \
  --to 29403858 \
  --batch-size=5000 \
  --commit \
  --checkpoints \
  merkle
```

## Execution notes

- treat this as an **offline** maintenance action; do not run it with `op-reth` / `op-node` still serving the lane
- keep the explicit local Shape runtime config in place; do not swap config provenance mid-repair
- a small temporary swap file can be a reasonable risk reducer before the run if RAM headroom is tight; record that as temporary incident scaffolding, not a permanent requirement
- prefer VPS-side orchestration / postwatch for restart sequencing if the rebuild may outlive the local Hermes wrapper session

## Verification after the rebuild

Restart order:
1. `op-reth`
2. wait for RPC readiness
3. `op-node`

Do not call it fixed until all of these are true:
- local decimal `eth_blockNumber` advances past the formerly pinned head
- lag versus public Shape shrinks over repeated samples
- `optimism_syncStatus` progresses instead of staying flat
- the lane stays up after restart rather than immediately exiting again

## What this reference adds to the umbrella skill

This is the gap between the existing `repair-trie` guidance and the destructive reseed path: a datadir-preserving offline Merkle checkpoint catch-up when `MerkleChangeSets` is the isolated lagging stage.