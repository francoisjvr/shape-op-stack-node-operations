# Free L1 provider cutover and reset behavior (2026-06)

Use this note when a Shape mainnet `op-node` is healthy enough to serve current unsafe head, but `safe` / `finalized` progression stalls because the configured L1 provider has exhausted quota.

## Trigger pattern

Observed on the Shape mainnet Reth stack:
- `op-reth` stayed healthy and matched public Shape head closely.
- `unsafe_l2` kept advancing.
- `safe_l2` and `finalized_l2` stayed pinned.
- `op-node` logs showed repeated L1 fetch failures from Alchemy:
  - `429 Too Many Requests`
  - `Monthly capacity limit exceeded`

Interpretation:
- This is not an execution-layer failure.
- It is an L1 data-availability / derivation-input problem for `op-node`.

## Reachable free-provider pair validated from the Shape VPS

The following endpoints were reachable from `178.18.241.113` during this session:

- L1 execution RPC: `https://ethereum-rpc.publicnode.com`
- L1 beacon API: `https://ethereum-beacon-api.publicnode.com`

Basic validation succeeded:
- JSON-RPC `eth_chainId` on PublicNode execution returned mainnet chain ID.
- Beacon `eth/v1/beacon/genesis` on PublicNode returned a valid Ethereum mainnet genesis payload.

## Conservative public-RPC throttling that was applied

When replacing a quota-exhausted paid provider with free public infrastructure, start `op-node` conservatively:

- `--l1.rpckind=basic`
- `--l1.max-concurrency=1`
- `--l1.rpc-max-batch-size=1`
- `--l1.rpc-rate-limit=4`
- `--l1.http-poll-interval=24s`

These flags are low-throughput on purpose. The goal is to stop repeated provider errors first, then observe whether derivation recovers.

## Important cutover behavior

Switching providers by recreating `op-node` can cause a long derivation rewind / reset window.

Observed behavior after the cutover:
- Alchemy quota errors disappeared.
- `op-node` restarted cleanly.
- `op-reth` stayed up.
- `op-node` entered a long `Walking back L1Block by hash` rewind sequence.
- For more than 10 minutes after restart, `optimism_syncStatus` still showed:
  - `unsafe_l2 = 0`
  - `safe_l2 = 0`
  - `finalized_l2 = 0`
  - `current_l1 = 0`
- `head_l1` continued to advance, and later `safe_l1` / `finalized_l1` repopulated before L2 fields did.
- During that reset window, local Shape head stayed flat and fell behind public head.

Interpretation:
- Removing the quota error does **not** mean the node is immediately healthier from an operator point of view.
- A free-provider cutover can be a valid stopgap, but it may temporarily degrade the node while `op-node` replays derivation history.

## Operator rule

After an L1-provider cutover, do **not** report success just because:
- the container is up,
- 429s stopped,
- `head_l1` is moving,
- or `optimism_syncStatus.current_l1` / `unsafe_l2` merely become non-zero again.

This session showed a sharper pitfall: `current_l1` and `unsafe_l2` repopulated, but the local Shape head still stayed pinned at `29,396,633`, `op-reth` kept warning `Beacon client online, but no consensus updates received for a while`, and `op-node` later returned temporary insertion failures such as:
- `failed to insert unsafe payload ... updated forkchoice, but node is syncing`
- `Engine temporary error ... 503 Service Unavailable: Request cancelled by healthcheck because node http://ethereum-geth:8545 died`

Treat reset completion as a **midpoint**, not success. For operator-facing health, wait for one of these stronger signals:
- local Shape head resumes moving,
- `unsafe_l2` increases beyond the pre-cutover stuck height,
- `safe_l2` / `finalized_l2` advance beyond their pre-cutover values,
- or the node regains close parity with public head.

If only `head_l1` moves, or `unsafe_l2` repopulates without any further block progress, report the cutover as **incomplete / still rewinding or still unable to drive L2 forward**, not fixed.

## Version-specific flag pitfall

Do not try to solve this class of stall by adding `--rollup.historicalrpc` to `op-node v1.18.0`.

Observed directly in this session:
- `op-node v1.18.0` exited with `flag provided but not defined: -rollup.historicalrpc`
- the container fell into a restart loop until the flag was removed

For the current Shape Reth stack, `--rollup.sequencer=https://mainnet.shape.network` and `--rollup.historicalrpc=https://mainnet.shape.network` belong on `op-reth`, not on `op-node`.

## Proxy-specific pitfalls discovered during fallback wiring

When putting a local failover proxy in front of paid + free L1 execution providers, do **not** assume HTTP status alone is enough for failover.

Observed directly during this session:
- Alchemy sometimes returned throughput / quota failures inside a **HTTP 200 JSON-RPC error body** rather than a plain 429.
- A proxy that only failed over on HTTP 4xx/5xx left `op-node` effectively stuck on the unhealthy primary.
- The fix was to treat response bodies containing rate-limit / capacity-limit signals (e.g. `Monthly capacity limit exceeded`, `compute units per second capacity`, `Too Many Requests`) as retryable and fall through to the next upstream.

Also observed:
- `https://eth.drpc.org` on the free tier rejected JSON-RPC batches larger than 3 with errors like `Batch of more than 3 requests are not allowed on free tier`.

Operator implication:
- If dRPC is part of the fallback chain, keep `--l1.rpc-max-batch-size=1` (or at minimum <= 3), otherwise the tertiary fallback can fail even when connectivity is fine.
- PublicNode may still 429 under bursty load, so low-concurrency / low-rate settings remain useful when free providers are expected to carry traffic.

## Loosening throttles after paid-provider recovery

Once a valid paid Alchemy key is restored behind the local failover proxy, it is reasonable to loosen the emergency public-RPC throttles, but do **not** expect that change alone to cure a node that is already stuck.

Observed directly in this session after changing from:
- `--l1.max-concurrency=1`
- `--l1.rpc-max-batch-size=1`
- `--l1.rpc-rate-limit=4`

to:
- `--l1.max-concurrency=10`
- `--l1.rpc-max-batch-size=3`
- `--l1.rpc-rate-limit=15`

Results:
- recreating `op-node` triggered another full derivation reset / rewind
- `current_l1` resumed inching forward
- local `eth_blockNumber` still stayed pinned
- `safe_l1` / `finalized_l1` temporarily dropped back to zero during the reset window
- `op-node` continued reporting `failed to insert unsafe payload ... updated forkchoice, but node is syncing`

Operator implication:
- throttle loosening is safe as a throughput experiment once the paid provider is back
- but if the local L2 head remains flat afterwards, the real blocker is probably **not** the L1 rate limit anymore

## Fast operator path for future L1 API / key rotations

For this Shape host, the low-friction recovery model should be:

1. **Keep `op-node` pinned to stable local proxy URLs** rather than editing upstream provider URLs into the container directly.
   - execution proxy: `http://127.0.0.1:18580`
   - beacon proxy: `http://127.0.0.1:18581`
2. Change provider URLs / keys in the runbook `.env`, not in the `op-node` command line.
3. Restart only the two proxy services first:
   - `systemctl restart shape-mainnet-l1-rpc-failover.service`
   - `systemctl restart shape-mainnet-l1-beacon-failover.service`
4. Verify proxy health before touching the node:
   - `curl http://127.0.0.1:18580/__proxy/health`
   - `curl http://127.0.0.1:18581/__proxy/health`
5. Leave `op-reth` alone unless there is separate evidence it is unhealthy.

If the node still wedges after a provider rotation, the next simple reset should be **op-node state only**, not a full Reth reseed:
- stop / recreate only `op-node`
- wipe only the small derivation state under `/root/shape-mainnet-op-node-data/`:
  - `opnode_safedb/`
  - `opnode_peerstore_db/`
- keep `opnode_p2p_priv.txt`
- do **not** touch `/root/shape-mainnet-op-reth-data`

Operational framing:
- changing the L1 API key should normally be a **proxy reload**, not a mainnet rebuild
- if recovery needs more than proxy restart + op-node-only reset, you are no longer in the "simple API swap" failure class

## Additional diagnostic signal: `eth_syncing`

If `op-node` keeps saying `updated forkchoice, but node is syncing`, query the local execution client directly with `eth_syncing`.

Observed directly in this session on `op-reth`:
- `startingBlock = currentBlock = highestBlock = 29,777,770`
- one internal stage (`MerkleChangeSets`) still lagged far behind at `26,385,482`
- delta: `3,392,288` blocks

Interpretation:
- the execution client can still consider itself internally mid-sync / mid-stage even when the exposed head is flat
- repeated unsafe-payload insertion failures from `op-node` may therefore reflect a local `op-reth` state problem, not an L1 provider throughput problem

## Practical use

Use the free PublicNode pair as a temporary fallback when a paid L1 provider is quota-exhausted, but frame it correctly:
- good for removing obvious provider-side rate-limit failures
- not proof of fast recovery
- requires post-cutover observation before declaring the Shape node healthy again
