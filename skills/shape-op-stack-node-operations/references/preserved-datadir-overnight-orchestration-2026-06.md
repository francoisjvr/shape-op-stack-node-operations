# Preserved-datadir overnight recovery orchestration (June 2026)

Use this reference when a Shape mainnet `op-reth` lane is stalled, the preserved datadir still matters, and the user has explicitly authorized autonomous monitoring/fixing while they are away.

## Observed proof pattern before offline maintenance
- Local `eth_blockNumber` pinned at `29708750` across repeated samples.
- Public Shape head kept advancing; lag widened instead of shrinking.
- `eth_syncing = false` even while stalled.
- `eth_getBlockByNumber(head+1)` returned `null`.
- `optimism_syncStatus` still showed moving L1 visibility while `unsafe_l2`, `safe_l2`, and `finalized_l2` stayed flat.
- `op-reth db ... stage-checkpoints get` showed:
  - `Execution=29708750`
  - `Finish=29708750`
  - `Headers=29708750`
  - `Bodies=29708750`
  - `SenderRecovery=29708750`
  - `MerkleExecute=29708750`
  - `MerkleChangeSets=26385482`

Interpretation:
- treat this as a preserved-datadir canonical stall with isolated Merkle/trie suspicion
- do not waste time blaming the L1 provider path once the local multiplexer/proxy is clearly healthy

## Live ports that mattered on this lane
- `op-node` rollup RPC: `127.0.0.1:19545`
- `op-reth` HTTP RPC: `127.0.0.1:18545`
- local execution multiplexer: `127.0.0.1:18580`
- local beacon multiplexer: `127.0.0.1:18581`

Operator rule:
- verify these from the VPS itself
- loopback-bound ports are not valid targets for direct external probes from the operator machine

## Least-destructive sequence used
1. Confirm the provider cutover separately from node recovery.
2. Back up live container inspect state before offline work.
3. Stop `op-node`, then stop `op-reth`.
4. Launch offline `repair-trie --dry-run` against the preserved datadir first.
5. Let the dry-run run long enough to become decisive; do not abort just because the wrapper goes quiet.
6. If the dry-run proves trie damage, run real offline `repair-trie`.
7. If the dry-run completes cleanly and `MerkleChangeSets` is still the isolated lagging checkpoint, pivot to the offline targeted Merkle rebuild path.
8. Restart `op-reth`, wait for local RPC readiness, then restart `op-node`.
9. Verify actual canonical recovery with repeated decimal head samples, `head+1` existence, lag trend, and `optimism_syncStatus`.

## Reporting split that helped
Keep two separate statuses in operator updates:
- `L1 provider path healthy again`
- `node still stalled / recovering / healthy`

Do not flatten them into one result. In this session, the fresh-key Alchemy multiplexer cutover was successful while the canonical node remained stalled.

## Automation lesson
For overnight incident work, a small autonomous orchestrator is justified when all of the following are true:
- the user explicitly authorized continued autonomous action
- the next action depends on a long-running remote offline command
- completion may happen after the current session or SSH wrapper dies
- there is a clear least-destructive decision tree already established

Minimum orchestrator responsibilities:
- watch the real remote repair process/container, not just the local wrapper
- choose the next branch (`real repair-trie` vs offline Merkle rebuild) from the dry-run result
- restart services in the correct order
- wait for RPC readiness before moving to the next service
- run the canonical recovery probes before reporting success
- leave a morning status report or scheduled follow-up in place

## Verification after restart
Require all of these before calling the lane recovered:
- local decimal `eth_blockNumber` moves past the pinned head
- the first missing canonical block becomes queryable
- lag vs public Shape shrinks over repeated samples
- `optimism_syncStatus` advances instead of staying flat
- recent `op-node` logs no longer reduce to the same missing-unsafe-range / forkchoice-syncing stall loop

## Pitfalls
- Do not probe `127.0.0.1:*` RPCs directly from outside the VPS and then misclassify `connection refused` as a node failure.
- Do not decide an offline repair died just because the local SSH wrapper timed out.
- Do not keep the lane down forever after a clean dry-run if the evidence now points toward the Merkle-only rebuild branch instead of confirmed trie damage.
- Do not report a successful provider cutover as if it fixed canonical sync when L2 head movement proof is still absent.
- Do not use a short fixed `head=0` watch as proof of failure after a deep unwind. In the June 2026 branch, `op-reth` legitimately replayed all the way back to `Execution=29,708,750` and then continued into `StorageHashing` while canonical `eth_blockNumber` still stayed `0`.
- Do not restart `op-node` just because a timer expired. In the same branch, the correct move was to keep `op-node` down, let `op-reth` finish the dangerous replay/hashing window, and only then treat restart as eligible.
