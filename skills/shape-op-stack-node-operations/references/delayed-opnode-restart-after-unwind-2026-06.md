# Delayed `op-node` restart after offline unwind / replay (2026-06)

Use this note when a Shape mainnet Reth lane has been unwound offline and `op-reth` comes back serving `eth_blockNumber = 0` while replaying execution forward.

## What happened

In the June 2026 unwind-to-`26,385,482` branch:
- pre-unwind canonical head was `29,708,750`
- the repair script immediately restarted both `op-reth` and `op-node`
- the first 30-minute watch looked like total failure because every sampled `eth_blockNumber` stayed `0`
- `optimism_syncStatus` also showed `unsafe_l2 = safe_l2 = finalized_l2 = 0`
- meanwhile `op-reth` logs were actively replaying execution forward underneath that flat RPC surface

Concrete live evidence after the misleading watch:
- `op-reth` logs kept printing `Executed block range ...`
- execution replay had already advanced into the `27.6M+` range
- `op-node` logs showed a repeated reset loop:
  - `Unsafe head is behind known finalized/safe blocks, execution-engine chain must have been rewound without forkchoice update. Attempting recovery now.`
  - `reset: forkchoice update returned unexpected status SYNCING`
- `op-node` still remembered safe/finalized at `26,385,482` while the engine head was still `0`

## Interpretation

This state is **not** the same as the older canonical-stall signature.

It means:
- the unwind succeeded deeply enough that canonical head is temporarily back at genesis-like state
- `op-reth` is still replaying execution forward offline/live after restart
- restarting `op-node` immediately creates noise and reset churn because its remembered safe/finalized heads are far ahead of the engine's temporary head

## Operator rule

After a deep offline unwind on this lane:
1. start `op-reth` first
2. verify replay is actually progressing from logs / CPU / stage movement
3. **keep `op-node` stopped** while `op-reth` replays toward the unwind target or at least past the remembered safe head
4. do **not** rely on `eth_blockNumber` alone as the restart gate; it can stay `0` long after replay is genuinely progressing
5. restart `op-node` only once `op-reth` has clearly crossed the dangerous replay window

Incident-tested proof that the replay was genuinely progressing:
- `Executed block range ...` lines kept advancing
- execution replay reached the full pre-unwind target `29,708,750`
- after that, `op-reth` switched into `StorageHashing`
- logs started printing fresh `Hashed <N> entries` counters while CPU stayed high
- canonical `eth_blockNumber` was **still `0`** during this phase

Operator takeaway:
- `eth_blockNumber >= 26,385,482` is a sufficient signal if it appears, but it is **not** the earliest trustworthy proof
- the stronger branch-specific proof is: execution checkpoint back at `29,708,750` plus live `StorageHashing` progress

## Verification signals that distinguish this from a dead lane

Treat the lane as `replaying, not dead` when all or most of these are true:
- `eth_blockNumber` is still `0` or very low
- `op-reth` container CPU stays high
- `op-reth` logs continue printing fresh `Executed block range ...` lines
- execution ranges increase over time even if RPC head does not yet expose them canonically
- `op-node` logs show reset-loop complaints about unsafe head being behind known safe/finalized heads

Treat it as truly stalled only if replay logs stop advancing too.

## Practical orchestration lesson

A short fixed post-restart watch is the wrong verifier for this branch.

The specific failure from this incident was:
- a 30-minute watch ended too early
- it treated `head = 0` as proof of failure
- it restarted `op-node` while `op-reth` was still legitimately replaying / hashing
- that created the misleading `unsafe head behind known safe/finalized` reset loop noise

Prefer a watcher that:
- leaves `op-node` stopped
- polls **both** canonical RPC state and stage/log progress
- treats rising execution ranges or fresh `StorageHashing` / `Hashed <N> entries` output as real forward motion even if canonical head is still `0`
- restarts `op-node` only after the chosen replay-finished threshold is crossed
- then begins the normal post-start verification window

This is safer than concluding failure from an early `head = 0` sample set while replay is still underway.
