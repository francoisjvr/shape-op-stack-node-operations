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

## Practical use

Use the free PublicNode pair as a temporary fallback when a paid L1 provider is quota-exhausted, but frame it correctly:
- good for removing obvious provider-side rate-limit failures
- not proof of fast recovery
- requires post-cutover observation before declaring the Shape node healthy again
