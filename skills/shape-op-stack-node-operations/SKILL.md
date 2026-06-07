---
name: shape-op-stack-node-operations
description: Operate, recover, verify, and document the current self-hosted Shape mainnet Reth stack using op-reth plus op-node.
version: 1.1.0
author: Ash
license: MIT
platforms: [linux]
metadata:
  hermes:
    tags: [shape, op-stack, op-node, op-reth, reth, docker, recovery, runbook]
---

# Shape Mainnet Reth Node Operations

Use this skill when working on the current self-hosted Shape mainnet Reth stack: health checks, recovery, restarts, config inspection, sync debugging, doc updates, and operator decision-making.

Reference:
- `references/sepolia-public-rpc-throttling-and-progress-signals.md` — Shape Sepolia public L1 RPC choices, `op-node` throttling flags that reduced 429-induced stalls, and the sync-status/log signals that indicate internal derivation progress even while `eth_blockNumber` is still flat.
- `references/unsafe-payload-canonical-stall-2026-06.md` — evidence pattern for the "unsafe payloads keep arriving but canonical head never advances" failure mode, including the MerkleChangeSets gap and trie-repair confirmation path.
- `references/shape-mainnet-repair-trie-runbook-2026-06.md` — concise offline `repair-trie` execution order, watcher pattern, and post-repair verification signals from the preserved live-datadir recovery.
- `references/reth-reseed-after-stalled-restart.md` — when to stop iterating on unwind/repair attempts and switch to a clean reseed from the latest downloaded Reth snapshot, while preserving operator notes.
- `references/large-snapshot-staging-and-extraction.md` — durable pattern for staging very large Shape snapshots: prefer local archive reuse, resumable `aria2c`, and offline decompression instead of `curl | tar` streaming.

## When to use

Trigger this skill when the task involves any of:
- Shape mainnet node health checks
- `op-reth` plus `op-node` runtime inspection
- Dockerized Shape node recovery
- deciding whether a data directory is safe to delete or move
- comparing live Shape behavior to official Shape or Superchain docs
- documenting a known-good Shape node configuration
- legacy `op-geth` comparison or rollback work

## Core operating principles

1. **Live mounts beat folder names.** Never assume a vaguely named directory is disposable. Inspect container mounts first.
2. **Protect uploaded chain state.** If an uploaded directory is mounted as the live datadir, preserve it unless the migration plan explicitly retires it.
3. **Check disk before restart loops.** On Dockerized Shape nodes, `no space left on device` can block restarts even when the protocol layer is otherwise fine.
4. **Back up container truth before recreating anything.** Save `docker inspect` output before removing a broken container.
5. **Verify with multiple signals.** Container uptime alone is not enough. Check block progression, RPC health, logs, and sync status.
6. **Report Shape block numbers in decimal for this user.** Convert hex RPC values before summarizing status.

## Standard workflow

### 1. Inspect before touching
- Check container state, restart count, and recent logs.
- Check root disk usage before restart attempts.
- Inspect bind mounts for `op-reth` and `op-node`.
- Identify which directory is the live execution datadir.

### 2. Confirm current architecture
For the current working Shape pattern, expect:
- `op-reth` and `op-node` running as Docker containers
- host networking
- a JWT file shared between them from the explicit config directory
- `op-node` using external L1 execution plus beacon providers
- `op-node --syncmode=consensus-layer`
- execution state coming from `/root/shape-mainnet-op-reth-data`
- config mounted from `/root/.shape-mainnet-op-reth-config`

### 3. Preserve before cleanup
Before deleting any large directory:
- confirm it is not mounted into the live stack
- confirm it is not the only copy of current chain state
- prefer deleting clearly unused staging or experiment directories first

### 4. If recovery is needed
Recovery order:
1. inspect
2. check disk
3. confirm mounts
4. back up `docker inspect`
5. free only safe disk space
6. retry
7. if the container itself is broken, recreate the container against the same preserved datadir
8. restart `op-node` only after `op-reth` is serving Engine API again

### 5. Verify health
Use all of:
- local `eth_blockNumber`
- public Shape block reference
- latest block hash parity when possible
- `eth_syncing`
- `optimism_syncStatus`
- recent `op-reth` logs
- recent `op-node` logs

## Shape-specific pitfalls

### Pitfall: snapshot restore via `curl | tar` is fragile at Shape snapshot sizes
For ~200 GB Shape Reth archives, streaming download straight into extraction is a reliability trap. A transient HTTP interruption can leave the restore half-complete with no easy resume point.

Preferred restore order for this user:
- first look for an already-downloaded local archive and validate its expected size
- if the archive is incomplete, resume it with `aria2c -c` and keep the matching `.aria2` control file
- only start extraction after the archive is complete
- extract offline with `pigz -dc <archive> | tar -xf -` (or equivalent local decompression), not live network streaming

Operator rules:
- do not throw away a valid local archive just because a previous streamed restore failed
- kill stale readers or inspection processes against the same tarball before the real extract so they do not steal IO/CPU
- keep non-target networks (for example Shape Sepolia) running unless the task explicitly includes touching them
- treat container-start success after reseed as secondary; the first goal is a deterministic, restartable staging/extract path

See `references/large-snapshot-staging-and-extraction.md`.

### Pitfall: throwing away interrupted snapshot downloads
Large Shape snapshot pulls, especially from Alchemy-backed URLs, may interrupt mid-transfer without the archive being unusable. If `aria2c` was used, preserve the partial file and matching `.aria2` control file and retry with `aria2c -c` before restarting from zero. Also verify no stale `aria2c` process is still writing the same target.


### Pitfall: deleting the upload folder
A directory named something like `/root/Upload` may actually be the live datadir. Treat ambiguous names as suspicious until mounts prove otherwise.

### Pitfall: misreading peer count
For current Shape mainnet operation, `net_peerCount = 0` may be expected and should not be treated as a failure by itself.

### Pitfall: trusting embedded defaults too much
If Shape fork behavior or baked-in chain handling looks stale or incomplete, prefer explicit runtime config files for both `op-node` and `op-reth` rather than assuming bundled network config is sufficient.

### Pitfall: the EL-sync stall looks dead before it recovers
A historically recurring failure mode on this Shape Reth lane is:
- `op-reth` head stays flat
- `eth_syncing` shows stages pinned while `MerkleChangeSets` lags behind
- `op-node` logs `failed to insert unsafe payload` / `updated forkchoice, but node is syncing`
- after restart, `optimism_syncStatus` may temporarily show zeroed `current_l1` / `safe_l2` / `finalized_l2`

A safe non-destructive remediation to try before any datadir reset is:
- keep the explicit local Shape chainspec/runtime config
- keep `op-reth --rollup.sequencer` / `--rollup.historicalrpc`
- recreate `op-node` with `--syncmode=execution-layer`
- then verify whether `op-node` starts continuously inserting unsafe payloads again

Important nuance:
- in this recovery mode, `unsafe_l2` may jump to near-tip **before** canonical `eth_blockNumber` moves
- do not call it fixed until the execution head advances in decimal and missing canonical blocks actually appear

### Pitfall: unsafe payload acceptance can still hide a dead canonical chain
A more deceptive variant of the EL-sync stall is when `op-node` appears healthy because it keeps logging:
- `Inserting unsafe L2 execution payload to drive EL sync`
- `Inserted new L2 unsafe block (synchronous)`

and `op-reth` logs matching `Received new payload from consensus engine` heights, **but** the canonical RPC still stays pinned.

Treat this as a likely trie/database consistency problem when the following line up together:
- `eth_blockNumber` is flat across repeated samples
- `eth_getBlockByNumber(head+1)` returns `null`
- `eth_getBlockByHash(<recent unsafe payload hash>)` returns `null`
- `pending` resolves to the same old canonical block instead of a newer in-flight block
- `eth_syncing` shows most stages at the current head while `MerkleChangeSets` is far behind
- `optimism_syncStatus` may still show `unsafe_l2` advancing far ahead of local canonical head

Operator interpretation:
- engine-side payload acceptance alone does **not** prove canonical import
- if `MerkleChangeSets` is millions of blocks behind while `Execution`, `Headers`, `Bodies`, and `Finish` sit at the tip, suspect trie inconsistency before blaming `op-node`
- a durable confirmation path is `op-reth db --datadir /data repair-trie --dry-run --chain /config/genesis-l2.runtime.json`
- if that dry run reports repeated `StorageMissing(...)` or similar trie inconsistencies, treat that as the root cause candidate for the stalled canonical head

Verification rule for this failure mode:
- do not call the node recovered just because `unsafe_l2` is climbing or because `op-node` says payload insertion succeeded
- require real canonical movement: decimal `eth_blockNumber` advance **and** the first missing canonical block becomes queryable by number/hash

If dry-run confirms trie damage, use the least-destructive offline repair sequence against the preserved live datadir:
1. stop `op-node` first so it stops feeding new unsafe payloads
2. stop `op-reth`
3. run `op-reth db --datadir /data repair-trie --chain /config/genesis-l2.runtime.json --dry-run --color never`
4. only if dry-run reports repeated trie/storage inconsistencies such as `StorageMissing(...)`, run the same command again **without** `--dry-run`
5. keep the explicit local Shape runtime config mount in place; do not switch chainspec sources mid-repair
6. restart `op-reth` first and wait for RPC readiness before restarting `op-node`

Execution guardrails for the real repair:
- treat `repair-trie` as an offline maintenance step, not an in-place live fix
- prefer a background watcher that can automatically restart `op-reth`, wait for RPC readiness, then restart `op-node` and run verification probes
- if the repair is launched via `docker run --rm ... | tee ...`, do **not** decide completion from `docker ps` alone; watch the actual `op-reth db repair-trie` process and/or the `mdbx.dat` lock holder, because a naive container-only watcher can race and start the main service too early
- while repair is running, verify liveness from process state / CPU usage rather than assuming silence means failure

Post-repair verification nuance from this session:
- recovery evidence may arrive in layers rather than all at once
- `safe_l2` can start advancing before `eth_blockNumber`, `unsafe_l2`, or `finalized_l2` visibly move again
- if `safe_l2` resumes moving after repair while the containers stay healthy, keep the watcher running until canonical head checks complete instead of declaring failure too early

Escalation rule from the next session:
- if a deep unwind replays directly back into the same poisoned head, or a repair attempt still leaves canonical import stuck, stop iterating on subtle DB recovery once a fresh snapshot is already fully downloaded
- for this user, prefer the low-friction operator move: clean reseed from the newest verified snapshot, keep a timestamped notes directory, and stop abandoned watchers/temporary containers before extraction
- also kill stale archive-inspection readers against the same tarball so they do not steal IO/CPU from the real restore
- see `references/reth-reseed-after-stalled-restart.md` for the concrete reseed checklist

### Pitfall: misreading post-fork fee behavior
On current Shape mainnet, do not assume a quiet chain still accepts near-zero gas pricing. Recent live RPC checks showed an active minimum base fee floor around `0.01 gwei`, with practical tx pricing around `0.011 gwei` and `0.001 gwei` priority fee. If transactions suddenly start failing as underpriced after fork activation, verify:
- `baseFeePerGas`
- `eth_gasPrice`
- `eth_maxPriorityFeePerGas`
- block `extraData` min-base-fee encoding when relevant

Also distinguish fee-floor changes from operator-fee changes. In the observed Shape mainnet state during this session, operator-fee params were still zero, so the effective minimum was explained by the base-fee floor rather than a new operator surcharge.

### Pitfall: public RPC failures can masquerade as fee problems
If public Shape RPC behavior looks inconsistent, probe the private RPC before blaming the chain fee model. During this session, the public RPC returned `403` while the private RPC was healthy and accepted traffic. For fee estimation, cost debugging, and transaction sends, prefer the RPC that is actually serving current chain data.

### Pitfall: blank `docker logs` on `op-reth`
`op-reth` may appear healthy and serve RPC while `docker logs shape-mainnet-op-reth` stays blank if stdout logging was not enabled explicitly. In Dockerized Shape setups, include:
- `--log.stdout.format=log-fmt`
- `--log.stdout.filter=info`

If logs are blank, treat that as an observability/config issue first, not automatic evidence that the node is idle or crashed.

## Documentation expectations
When the user asks for notes or a repo:
- write an operator-first runbook, not a vague summary
- include exact flags, paths, ports, and restart order
- explicitly call out differences between official docs and the working live setup
- include redaction-safe reference config examples
- include a recovery section ordered from least destructive to most destructive

## Known-good pattern from the current Reth runtime
A proven working branch now includes:
- `op-reth:v2.2.2`
- `op-node:v1.18.0`
- `op-node --syncmode=consensus-layer`
- `op-node --l2.enginekind=reth`
- explicit runtime files in `/root/.shape-mainnet-op-reth-config`
- `op-reth` datadir at `/root/shape-mainnet-op-reth-data`
- `op-node` datadir at `/root/shape-mainnet-op-node-reth-data`
- `op-reth --rollup.sequencer=https://mainnet.shape.network`
- `op-reth --rollup.historicalrpc=https://mainnet.shape.network`
- health verification via local-vs-public block comparison, block-hash parity, and rollup sync checks

Important version-specific pitfalls:
- `op-node v1.18.0` does **not** accept `--rollup.historicalrpc`; that flag belongs on `op-reth` / `op-geth`, not on `op-node`.
- `op-node v1.18.0` also rejects `--datadir`. For persistent op-node state on this stack, mount the host op-node directory at `/data` in the container and use explicit paths such as `--safedb.path=/data/opnode_safedb`, `--p2p.priv.path=/data/opnode_p2p_priv.txt`, and `--p2p.peerstore.path=/data/opnode_peerstore_db`.
- In the June 2026 mainnet recovery, switching `op-node` between `execution-layer` and `consensus-layer` modes did **not** unstick a canonically pinned snapshot by itself; use sync-mode flips only as short discriminators, not as a substitute for a clean snapshot-first reinstall when the user explicitly wants the Shape runbook followed.

Operator preference / incident posture:
- If Morpheus says to use the canonical Shape runbook/book, stop side experiments quickly and realign to the repo-prescribed flow instead of continuing with canaries or alternate branches.
- For mainnet rebuilds, prefer the book's snapshot-first sequence in order: confirm the latest official snapshot source, clear non-book mainnet containers, reset the canonical paths (`/root/shape-mainnet-op-reth-upload`, `/root/shape-mainnet-op-reth-data`, `/root/shape-mainnet-op-node-reth-data`, `/root/.shape-mainnet-op-reth-config`), stage the snapshot, promote it, copy source artifacts to runtime config, generate a fresh JWT, then start the compose-defined stack and verify repeated samples.
- Treat canaries / mode flips / isolated ports as secondary discriminators only after the straight runbook reinstall path has been tried or the user explicitly asks for experimentation.

- If a repaired snapshot still shows `unsafe_l2` movement without canonical advancement, an empty-datadir / from-genesis canary is a valid next discriminator. But interpret it carefully: if the canary tracks L1 and receives payloads while canonical `head` remains `0`, that result means the issue is **not cleanly snapshot-only**. See `references/mainnet-empty-datadir-canary-2026-06.md`.

## Support files
- `references/shape-mainnet-geth-recovery-2026-05.md` — legacy recovery notes that explain how the old geth path failed and why the current Reth-first runtime became the preferred operator path.
- `references/shape-mainnet-reth-runtime-2026-05.md` — validated live-runtime details, doc-alignment notes, and push-scope lessons from the current Reth-first operator path.
- `references/unsafe-payload-canonical-stall-2026-06.md` — condensed evidence and probe sequence for diagnosing when unsafe payloads advance but canonical block import remains stuck.

## Workflow guardrails
- When a Shape node skill, repo doc, or runbook is partially updated but still carries legacy geth framing, finish the cleanup proactively. Do not stop at "correct enough" if the remaining drift is obvious and safely fixable.
- When the task is to align docs or skills with the live server, distinguish between three targets before speaking confidently: local working tree, committed local repo, and pushed GitHub remote. Do not say "the repo is updated" if the work only exists locally.
- If a doc or repo belongs to Shape Network upstream but the authenticated GitHub identity only has read access there, publish the finalized update to the user's own repo/fork instead of implying the upstream repo was updated.
- When the user asks for troubleshooting, research, or provenance checking on a Shape node question, keep executing the next concrete verification step instead of narrating possible next steps. Treat "I can check X next" as a smell unless the user explicitly asked for options.
- When a live Shape config disagrees with public docs or the public Superchain Registry, verify provenance directly from the running host before theorizing. Prefer this order: explicit config dir -> extracted datadir copies -> published Shape docs artifacts -> Superchain Registry. If the live config is byte-identical to snapshot-bundled files, record that plainly as the observed source of truth.

## Documentation and runbook maintenance
When the task is partly or wholly documentation, keep the runbook aligned to the live operator reality rather than to stale generic OP Stack guidance.

Authoring rules for this subclass:
- prefer canonical Shape docs pages over brittle direct asset URLs
- when a provider download link rotates, link the docs page first and the docs source/provider landing page second
- keep path roles explicit: runtime, staging, config, cache, uploaded snapshot
- define health in terms of sync progress and block movement, not merely container start success
- preserve Reth-first framing unless the user explicitly asks for legacy geth context
- keep Shape block numbers human-readable decimal in operator-facing docs

Good documentation edits include:
- a quick "Where to get the current snapshot" section near bootstrap instructions
- a short explanation of why stable docs/source/provider links are safer than hardcoded snapshot archives
- recovery steps ordered from least destructive to most destructive

See `references/shape-official-snapshot-sources.md` for the absorbed runbook-authoring notes.

## Additional references
- `references/shape-mainnet-fee-floor-and-rpc-notes-2026-06.md` — private-vs-public RPC findings, live Shape fee-floor observations, and why post-fork underpriced txs can happen even on a quiet network.
- `references/shape-rpc-access-vantage-notes-2026-06.md` — why official public Shape RPC can return `403` from the VPS while private/local RPC remains healthy, and how to verify parity from an alternate vantage point.
- `references/shape-sepolia-config-provenance.md` — evidence trail for when live Shape Sepolia rollup/genesis files diverge from public Shape docs artifacts and the public Superchain Registry.
- `references/free-l1-provider-cutover-and-reset-behavior-2026-06.md` — validated free Ethereum mainnet execution+beacon fallback endpoints for the Shape VPS, conservative `op-node` throttling flags, and the observed long post-cutover rewind behavior.

### Pitfall: free-provider cutover can remove 429s but still leave the node temporarily worse
If `op-node` is stalling on L1 quota exhaustion, swapping to reachable free public Ethereum providers can be a valid stopgap. In this session, the Shape VPS could reach:
- `https://ethereum-rpc.publicnode.com`
- `https://ethereum-beacon-api.publicnode.com`

Conservative flags that were valid for `op-node v1.18.0` and worth trying first:
- `--l1.rpckind=basic`
- `--l1.max-concurrency=1`
- `--l1.rpc-max-batch-size=1`
- `--l1.rpc-rate-limit=4`
- `--l1.http-poll-interval=24s`

Validated Alchemy mainnet replacements from this session:
- execution RPC: `https://eth-mainnet.g.alchemy.com/v2/<api-key>`
- beacon API: `https://eth-mainnetbeacon.g.alchemy.com/v2/<api-key>`

Beacon API verification pitfall:
- `GET /eth/v1/node/health` returned `400` on Alchemy's beacon endpoint in this session (`Unsupported method`), so use endpoints like `GET /eth/v1/beacon/headers/head`, `GET /eth/v1/node/version`, or `GET /eth/v1/config/spec` to confirm reachability instead of assuming the standard health route works there.

But do **not** equate "429s disappeared" with recovery. Recreating `op-node` for the provider swap triggered a long `Walking back L1Block by hash` rewind. During that window:
- local Shape head stayed flat
- public head kept moving away
- `unsafe_l2`, `safe_l2`, `finalized_l2`, and `current_l1` stayed zero for an extended period
- only `head_l1`, and later `safe_l1` / `finalized_l1`, repopulated first

Operator rule: after a provider cutover, do not call it fixed until `current_l1` and `unsafe_l2` repopulate or the local Shape head resumes moving.

### Pitfall: public parity checks can depend on vantage point
The same official Shape public RPC may behave differently depending on where you call it from. In this session, the VPS itself received `HTTP 403` from both `https://mainnet.shape.network` and `https://sepolia.shape.network`, while the Hermes host could query those endpoints normally and confirm parity. In a later automation check, the mainnet endpoint also proved sensitive to request shape: adding a non-empty `User-Agent` made public probes succeed from a vantage where a bare probe looked falsely broken.

Do not classify the node as unhealthy just because the node host cannot reach the public endpoint. Split the problem into:
- local/private RPC health
- local node progress
- public endpoint reachability from this specific machine

If the VPS gets `403` from the public endpoint:
- retry once with a non-empty `User-Agent` before classifying the public RPC as unusable from that vantage
- verify local/private head movement first
- verify `eth_syncing`
- verify `optimism_syncStatus`
- verify recent logs
- when parity against public is still needed, query the public endpoint from another allowed vantage point and compare block number/hash there
