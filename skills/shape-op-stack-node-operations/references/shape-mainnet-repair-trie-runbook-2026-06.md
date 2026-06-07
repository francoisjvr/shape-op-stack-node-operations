# Shape mainnet op-reth repair-trie runbook — 2026-06

Use this when the Shape mainnet Reth lane shows the deceptive pattern where `op-node` continues feeding unsafe payloads but canonical L2 import stays stuck.

## Trigger pattern

Treat trie repair as justified when these line up together:
- `eth_blockNumber` is flat across repeated samples
- `unsafe_l2` may keep arriving or stay near-tip, but canonical block `head + 1` is still missing
- `eth_getBlockByNumber(<next missing block>)` returns `null`
- `eth_syncing` leaves late stages badly behind, especially `MerkleChangeSets`
- `op-reth db repair-trie --dry-run` reports repeated trie/storage inconsistencies such as `StorageMissing(...)`

## Safe execution order

1. Stop `op-node` first.
   - Prevents more unsafe payload pressure during maintenance.
2. Stop `op-reth`.
   - `repair-trie` should be treated as offline maintenance.
3. Run dry-run against the live preserved datadir with the explicit Shape runtime config.
4. Only if dry-run confirms inconsistency, run the same command without `--dry-run`.
5. Keep a watcher process ready to:
   - detect repair completion
   - start `op-reth`
   - wait for RPC readiness
   - start `op-node`
   - poll `optimism_syncStatus`, `eth_blockNumber`, and stage checkpoints

## Command shape

```bash
docker run --rm --network host \
  -v /root/shape-mainnet-op-reth-data:/data \
  -v /root/.shape-mainnet-op-reth-config:/config:ro \
  us-docker.pkg.dev/oplabs-tools-artifacts/images/op-reth:v2.2.2 \
  db --datadir /data repair-trie --chain /config/genesis-l2.runtime.json --color never
```

Dry-run form adds:

```bash
--dry-run
```

## Operator notes

- Do not switch chainspec/genesis sources during diagnosis or repair; keep using the same explicit local runtime config that the node was already using.
- If repair output is quiet for a long time, confirm the process is still alive and consuming CPU before assuming it is wedged.
- A background watcher is safer than ad-hoc manual restarts because it preserves the restart order and immediately begins verification.

## Verification nuance

Do not require a single perfect signal immediately after restart.

Observed recovery sequence from this session:
- containers stayed up
- `head_l1`, `safe_l1`, and `finalized_l1` kept moving
- `eth_blockNumber` and `unsafe_l2` remained flat at first
- `safe_l2` eventually resumed advancing before canonical head visibly moved again

Practical rule:
- treat resumed `safe_l2` movement as encouraging evidence that the repair helped
- but keep the watcher running until canonical L2 checks also pass: decimal head movement and previously missing block availability by number/hash

## What not to do

- Do not delete or replace the preserved datadir before trying dry-run plus offline repair.
- Do not restart `op-node` before `op-reth` RPC is actually ready.
- Do not declare success just because unsafe payload insertion logs return.
