---
name: shape-op-stack-node-operations
description: Operate, recover, verify, and document the current self-hosted Shape mainnet Reth stack using op-reth plus op-node.
version: 1.2.0
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
- `references/repair-trie-wrapper-vs-remote-process-watch-2026-06.md` — wrapper-exit ambiguity during offline `repair-trie`, how to identify the real anonymous repair container/PID, and why a VPS-side postwatch is safer than relying on a local watcher.
- `references/reseed-confirmation-and-sigpipe-pitfall-2026-06.md` — destructive reseed confirmation gate, `set -euo pipefail` + `head`/SIGPIPE orchestration trap, and the post-reseed proof checklist.
- `references/shape-mainnet-repair-trie-runbook-2026-06.md` — concise offline `repair-trie` execution order, watcher pattern, and post-repair verification signals from the preserved live-datadir recovery.
- `references/post-reseed-trie-stall-and-repair-trie-confirmation-2026-06.md` — what to do when the same canonical-stall signature survives a fresh snapshot reseed: proof pattern, `stage-checkpoints` command shape, dry-run behavior, and real repair progress signals.
- `references/offline-merkle-stage-rebuild-2026-06.md` — datadir-preserving offline Merkle stage rebuild when `MerkleChangeSets` is the isolated lagging checkpoint after reseed/repair attempts.
- `references/reth-reseed-after-stalled-restart.md` — when to stop iterating on unwind/repair attempts and switch to a clean reseed from the latest downloaded Reth snapshot, while preserving operator notes.
- `references/large-snapshot-staging-and-extraction.md` — durable pattern for staging very large Shape snapshots: prefer local archive reuse, resumable `aria2c`, and offline decompression instead of `curl | tar` streaming.
- `references/ssh-monitoring-and-cron-watchdogs.md` — SSH-first monitoring commands for the live Dockerized stack: `docker logs`, decimal head checks, lag watch loops, when `journalctl` is relevant, and lightweight cron/watchdog patterns.
- `references/post-reseed-flat-head-diagnostic-signals-2026-06.md` — how to classify the deceptive "reseed succeeded but canonical head is still flat" state, including the minimum proof checklist and watcher signals.
- `references/preserved-datadir-overnight-orchestration-2026-06.md` — when the user authorizes unattended recovery work: live port map, preserved-datadir stall proof pattern, provider-vs-node reporting split, and the overnight orchestration decision tree.
- `references/vps-side-recovery-watcher-and-self-match-pitfall-2026-06.md` — prefer a VPS-side watcher over a local SSH wrapper for long offline maintenance, and avoid false `already running` matches from broad `pgrep -af` checks.
- `references/live-merkle-unwind-watchdog-2026-06.md` — command-shape corrections for offline `stage run/drop/unwind` on the current `op-reth` image, plus the split repair-script / cron-watchdog pattern for unattended mainnet recovery.
- `references/delayed-opnode-restart-after-unwind-2026-06.md` — why an immediate `op-node` restart after deep unwind can create a fake `head=0` failure signal while `op-reth` is still replaying, and the safer delayed-restart watcher pattern.

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

For exact SSH-ready monitor commands, use `references/ssh-monitoring-and-cron-watchdogs.md`.

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

### Pitfall: do not assume every Shape lane has the generic Superchain Jovian schedule
When the user asks for "the fork timestamps," first disambiguate **mainnet vs Sepolia** and answer from the lane's actual config source of truth instead of assuming the common Superchain values.

Durable finding from this session:
- current Shape **mainnet** config may stop at `granite_time` and omit `holocene_time`, `isthmus_time`, and `jovian_time`
- current Shape **Sepolia** config can include later forks through `jovian_time`

Operator rule:
- for mainnet, verify against the live runtime file first (`/root/.shape-mainnet-op-reth-config/rollup.runtime.json`) and cross-check the published registry entry (`superchain/configs/mainnet/shape.toml`) before quoting fork timestamps
- for Sepolia, verify the active snapshot/runtime rollup file actually mounted by the lane, not a remembered constant from an earlier session
- do not import `jovian_time = 1764691201` from unrelated Superchain chains into Shape unless the Shape lane config explicitly contains it

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
- on this stack the repair often runs inside an anonymous `docker run --rm` container with a random name rather than the steady-state `shape-mainnet-op-reth` container name; inspect `docker ps -a`, `ps`, and parent/child process ancestry before assuming the repair died just because the named service container is exited
- wrapper processes are fragile: a local/background Hermes process or SSH wrapper can be killed while the remote `repair-trie` continues happily. Treat wrapper exit codes like `143`/`-15` as ambiguous until you verify the remote `op-reth db repair-trie` PID or container directly
- when orchestration is likely to outlive the local session, prefer installing a small VPS-side postwatch script that waits for the repair PID/container to disappear, then starts `shape-mainnet-op-reth`, waits for RPC readiness, starts `shape-mainnet-op-node-reth`, and runs the canonical recovery probes. This avoids losing the finish sequence when the local watcher dies
- if the maintenance logic itself may need to branch (`dry-run` -> real `repair-trie` vs Merkle rebuild), prefer moving that watcher logic onto the VPS and letting Hermes keep only a lightweight completion notifier
- avoid broad `pgrep -af '<script-name>'` launch guards for the VPS-side watcher; they can match the shell wrapper performing the check and create a false `already running` result. Confirm the exact long-lived Python PID with `ps` or by checking the launched PID directly
- while repair is running, verify liveness from process state / CPU usage, MDBX lock ownership, and fresh `docker logs` / log-file `mtime`, rather than assuming silence in the local wrapper means failure
- if a repeat real-repair run on the same datadir only shows `inconsistent_nodes=0` with no `StorageMissing(...)`-style signals and the ETA explodes into a day-scale sweep, abort that rerun and bring the main lane back up; that pattern is evidence you no longer have a confirmed trie-damage repro on the current state, so continuing the offline pass is usually just extended downtime, not a justified fix

Post-repair verification nuance from this session:
- recovery evidence may arrive in layers rather than all at once
- `safe_l2` can start advancing before `eth_blockNumber`, `unsafe_l2`, or `finalized_l2` visibly move again
- if `safe_l2` resumes moving after repair while the containers stay healthy, keep the watcher running until canonical head checks complete instead of declaring failure too early
- a deceptive partial-win variant is: repair finishes, canonical head advances a few hundred blocks, `safe_l2` / `finalized_l2` catch up to that new head, and then the node flatlines again with `eth_syncing = false`, `head + 1` still `null`, and `op-node` repeatedly requesting a huge missing unsafe range while `op-reth` only logs repeated far-future payload receipts
- classify that signature as `partial recovery, stall persists`, not as a healthy catch-up state
- a different but important **converging** signature is: `eth_syncing = false`, recent common-block hash parity already matches public, `op-reth` continues logging canonical commits, and local decimal head samples keep rising while lag shrinks overall — even if `op-node` still intermittently logs `failed to insert unsafe payload`, `updated forkchoice, but node is syncing`, or missing-unsafe-range warnings
- operator rule: do not overreact to those lingering `op-node` sync-pressure warnings by themselves once canonical head movement and parity are already proven; classify the lane as `converging`, not `stalled`, and keep monitoring repeated head samples plus lag trend

Escalation rule from the next session:
- if a deep unwind replays directly back into the same poisoned head, or a repair attempt still leaves canonical import stuck, stop iterating on subtle DB recovery once a fresh snapshot is already fully downloaded
- before escalating to a destructive reseed on a preserved datadir, inspect `stage-checkpoints`; if `Execution` / `Finish` / `MerkleExecute` are clustered near tip and `MerkleChangeSets` alone is far behind, prefer an offline targeted Merkle stage rebuild first
- for this user, prefer the low-friction operator move sequence: preserved-datadir discriminator -> targeted offline Merkle rebuild when justified -> only then clean reseed from the newest verified snapshot
- keep a timestamped notes directory, and stop abandoned watchers/temporary containers before extraction
- also kill stale archive-inspection readers against the same tarball so they do not steal IO/CPU from the real restore
- see `references/offline-merkle-stage-rebuild-2026-06.md` and `references/reth-reseed-after-stalled-restart.md`

Post-reseed escalation nuance from the follow-up session:
- a fresh snapshot reseed can still come back with the **same** canonical-stall signature immediately after both containers restart
- if post-reseed probes still show all of the following together, do **not** just wait on the flat head:
  - decimal `eth_blockNumber` pinned at the snapshot head across repeated samples
  - `eth_getBlockByNumber(head+1)` returns `null`
  - the formerly missing target block still returns `null`
  - `MerkleChangeSets` remains millions of blocks behind head
  - `op-node` requests a huge missing unsafe range and logs `updated forkchoice, but node is syncing`
- in that case, treat trie inconsistency as still live in the reseeded datadir and move back to offline `repair-trie --dry-run`, then real offline `repair-trie` if dry-run confirms repeated `StorageMissing(...)`
- see `references/post-reseed-trie-stall-and-repair-trie-confirmation-2026-06.md`

### Pitfall: `repair-trie` dry-run can look idle before it proves the case
If you stop the dry-run too early, you can miss the confirmation you actually needed.

Observed pattern from the reseed follow-up:
- startup logs appear first
- the command may sit quietly while scanning
- only later does it print repeated `StorageMissing(...)` inconsistencies

Operator rule:
- once the lane matches the canonical-stall signature, give `repair-trie --dry-run` enough time to scan before concluding it found nothing
- verify liveness from remote process/container state if the parent SSH wrapper is interrupted
- if you kill the local wrapper, check whether the remote `docker run ... repair-trie` kept running and kill that real repair container/process explicitly before relaunching orchestration

### Pitfall: `stage-checkpoints` flag placement differs from the obvious guess
On this `op-reth` build, `--datadir` belongs to the parent `db` command, not the `stage-checkpoints get` subcommand.

Use:
- `op-reth db --datadir /data --chain /config/genesis-l2.runtime.json stage-checkpoints get --color never`

Do not rely on:
- `op-reth db stage-checkpoints get --datadir /data ...`

### Pitfall: op-reth stage command shapes are easy to get subtly wrong during live recovery
On the current `op-reth:v2.2.2` Shape image, stage maintenance is **not** invoked through `op-reth db ... stage ...` and `docker run <image> op-reth ...` is also wrong.

Operator rules:
- use `op-reth stage run ... merkle`, not `op-reth db ... stage run merkle`
- inside `docker run --rm <image> ...`, invoke `stage ...` directly because the image entrypoint is already `op-reth`
- check `stage unwind to-block --help` before launching an unwind if you are rusty on the exact target syntax for the current build
- if offline stage work emits `Storage settings mismatch detected ... stored=storage_v2 false ... requested=storage_v2 true`, treat that as a warning first and judge the branch by the actual stage exit/result, not by the warning alone
- after a deep offline unwind, do **not** blindly restart `op-node` alongside `op-reth`; if the engine comes back at `head = 0` while replaying execution forward, an immediate `op-node` restart can create a misleading reset loop (`unsafe head behind known safe/finalized`, forkchoice `SYNCING`) and make a healthy replay look dead
- in that branch, leave `op-node` stopped and watch `op-reth` replay progress first (logs, CPU, stage movement, rising head once canonicalization catches up), then restart `op-node` only after `eth_blockNumber` has recovered to a sane threshold such as the unwind target / remembered safe head
- for unattended repair on this user’s live lane, keep the branching repair flow in one SSH-driven script and the health reporting in a separate cron watchdog

See `references/live-merkle-unwind-watchdog-2026-06.md`.

### Pitfall: an offline `stage run ... merkle --commit` failure is not automatically a reason to keep the lane down
A targeted offline merkle-stage rebuild can fail immediately at the stuck `MerkleChangeSets` checkpoint with a state-root validation error such as:
- `stage encountered an error in block #<checkpoint>: validation error: mismatched block state root`

Operator rule from this session:
- if the offline pass exits on that verification error, do **not** assume the datadir is now worse just because `postwatch` skipped the restart
- first bring `op-reth` back up on the preserved datadir, then recreate `op-node` with the intended L1 throttles, and verify live behavior from repeated decimal head samples
- if canonical `eth_blockNumber` resumes advancing and lag vs public Shape starts shrinking, classify the offline stage pass as `failed`, but the lane as `recovering`
- only keep the lane down for more offline surgery if the restarted node is still flat or regresses

### Pitfall: a clean reseed can still leave the canonical head pinned
Do not treat `snapshot extracted + services restarted + RPC back` as proof of recovery.

A deceptive post-reseed pattern seen on this Shape mainnet stack was:
- local canonical head stayed flat across repeated samples even after clean extraction and restart
- public Shape head kept advancing away
- `safe_l2` was nonzero again but still flat
- `MerkleChangeSets` stayed pinned far behind
- the formerly missing canonical target block still returned `null`

Verification rule after reseed:
- sample decimal `eth_blockNumber` repeatedly, not once
- compare against public head and confirm the lag is shrinking
- check whether `safe_l2` / `finalized_l2` are actually advancing
- if `MerkleChangeSets` was the suspicious lagging stage before reseed, confirm it moves too
- probe the first formerly missing canonical block by number/hash and require it to become queryable before calling the lane healthy

Reporting rule for this user:
- split `reseed/extract/restart succeeded` from `node healthy again`
- if restart succeeded but the proof checklist above fails, report the reseed as operationally incomplete rather than recovered

For the concrete signal set and watcher pattern, see `references/post-reseed-flat-head-diagnostic-signals-2026-06.md`.

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
- `op-node` persistent state mounted from the host `OP_NODE_DATA_DIR` into `/data` (current live mainnet stack observed at `/root/shape-mainnet-op-node-data`)
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

- If a repaired snapshot still shows `unsafe_l2` movement without canonical advancement, an empty-datadir / from-genesis canary is a valid next discriminator. But interpret it carefully: if the canary tracks L1 and receives payloads while canonical `head` remains `0`, that result means the issue is **not cleanly snapshot-only**. If the primary lane also only advances a few hundred blocks after `repair-trie` and then flatlines again, classify that as `partial recovery, broader canonicalization/runtime issue still present`, not success. See `references/mainnet-empty-datadir-canary-2026-06.md`.

- If Morpheus explicitly says to focus on mainnet only, stop preserving secondary Shape lanes as untouchable background state. Decommission Sepolia canaries/containers/paths that are only consuming ports or operator attention, then keep subsequent recovery and monitoring scoped to mainnet unless Morpheus later asks to restore Sepolia.

## Support files
- `references/shape-mainnet-geth-recovery-2026-05.md` — legacy recovery notes that explain how the old geth path failed and why the current Reth-first runtime became the preferred operator path.
- `references/shape-mainnet-reth-runtime-2026-05.md` — validated live-runtime details, doc-alignment notes, and push-scope lessons from the current Reth-first operator path.
- `references/unsafe-payload-canonical-stall-2026-06.md` — condensed evidence and probe sequence for diagnosing when unsafe payloads advance but canonical block import remains stuck.
- `references/post-reseed-flat-head-diagnostic-signals-2026-06.md` — post-reseed proof checklist for distinguishing a successful restart from a genuinely healthy canonical chain.

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
- `references/l1-provider-multiplexer-and-failure-matrix-2026-06.md` — stable local loopback proxy pattern for beacon/execution key rotation, why `l1.beacon-fallbacks` is only supplemental, and the power-cut vs storage-failure failure matrix.
- `references/alchemy-key-rotation-multiplexer-cutover-2026-06.md` — live cutover checklist for replacing a near-exhausted Alchemy key with fresh keys behind the local multiplexer, including systemd unit ownership and split verification/reporting.
- `references/shape-sepolia-health-classification-2026-06.md` — how to classify the Sepolia lane when execution head is near public tip but `current_l1` / `safe_l2` / `finalized_l2` are stalled by L1 provider quota exhaustion.
- `references/repo-canonicalization-and-metadata-pass-2026-06.md` — second-pass repo polish after canonical rename: README framing, doc numbering, practical-file surfacing, GitHub description/homepage/topics, and post-rename verification.

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

Provider-restoration nuance from the next session:
- when rotating back from throttled free/public L1 providers to a paid Alchemy key, verify the proxy path separately from node recovery
- for the proxy path, require: local proxy health endpoint success, a direct proxied RPC/beacon request returning `200`, and the response/header evidence showing the request really hit the intended Alchemy upstream instead of a fallback
- for node recovery, still require actual L2 progress: repeated decimal `eth_blockNumber` samples, `optimism_syncStatus`, and local-vs-public Shape head comparison
- if the new key is clearly live and `current_l1` resumes moving but `unsafe_l2` / canonical `eth_blockNumber` remain pinned, treat the provider swap as successful but **not** the root-cause fix
- in that situation, keep the report split into two layers for this user: `L1 provider path healthy again` vs `node still unhealthy`
- once paid Alchemy is confirmed back in the path, re-evaluate whether temporary free-tier safety throttles (`--l1.max-concurrency=1`, very low rate limit, batch size 1, long poll interval) are still justified; relaxing those is a valid next discriminator, but do not claim recovery before the head actually moves
- if Morpheus asks how to avoid future key-rotation restarts, prefer a stable local loopback proxy/multiplexer in front of multiple beacon keys (and optionally execution keys too) instead of teaching `op-node` individual provider keys directly
- treat `--l1.beacon-fallbacks` as supplemental only; OP docs describe it as fallback endpoints for blob sidecars not available at the primary beacon endpoint, not as the full general key-rotation architecture
- be explicit that the proxy pattern still requires one initial controlled `op-node` restart to move from direct remote URLs to local loopback URLs, but future key rotations should then become proxy-only changes
- when the user asks about VPS power cuts, answer in two layers: provider resilience vs storage resilience. Power cut with intact datadirs means reboot-and-resume, not automatic start-from-zero; disk loss or severe datadir corruption is the case that can still force repair/reseed

### Pitfall: direct L1 provider key swaps keep reintroducing avoidable `op-node` restarts
If `op-node` points directly at a remote beacon or execution provider URL, every key rotation or upstream cutover tempts the operator to edit `op-node` config again and restart the rollup node.

Prefer this architecture when the goal is to survive provider/key churn without another disruptive restart cycle:
- keep `op-node` pointed at one stable local loopback URL for beacon, for example `http://127.0.0.1:3500`
- optionally keep `op-node` pointed at one stable local loopback URL for execution too, for example `http://127.0.0.1:8547`
- rotate real upstream keys behind that local proxy/multiplexer
- use labeled upstreams so logs/health output identify the selected provider without leaking API keys
- fail over on timeout, connection failure, `429`, and `5xx`
- expose a local health endpoint for the proxy itself, but still verify real node recovery separately with block movement and sync status

Operator rule:
- a proxy or multiplexer reduces **provider-path** restart pressure; it does **not** protect against disk loss or severe datadir corruption
- report power-cut scenarios honestly: intact persistent datadirs mean reboot + resume, while storage failure can still require repair or reseed
- if the current Alchemy key is near quota exhaustion, do not leave it first in rotation just because it still works; move the fresh paid keys to the front and remove the near-exhausted key from the active live env
- once the multiplexer becomes authoritative, disable older direct failover units so there is only one active provider-control plane
- during the one-time cutover from direct remote URLs to loopback URLs, verify the proxy path first with direct proxied execution/beacon requests and labeled health output, then restart/recreate `op-node`
- after that restart, keep the status report split: `provider path switched successfully` is separate from `node fully healthy again`; a reset/rewind window can make the first true before the second

For the concrete service names and checklist from the live mainnet cutover, see `references/alchemy-key-rotation-multiplexer-cutover-2026-06.md`.

### Pitfall: Sepolia can look healthy at the head while safe/finalized is still dead
On the Shape Sepolia lane, do not classify the node as healthy from `eth_blockNumber` alone.

A deceptive but important pattern is:
- containers are up
- local `eth_syncing` is `false`
- local execution head advances over repeated samples
- local head is only a few L2 blocks behind public `https://sepolia.shape.network`
- a sampled canonical block hash matches public

while at the same time:
- `current_l1` is pinned far behind live Ethereum Sepolia L1
- `safe_l2` and `finalized_l2` are frozen hundreds of thousands of L2 blocks behind local head
- recent `op-node` logs show L1 provider quota failures such as `429 Too Many Requests`
- the upstream message may explicitly say the Alchemy monthly capacity limit was exceeded

Operator interpretation:
- execution / unsafe-canonical ingestion may still be near-tip
- safe/finalized derivation can still be unhealthy
- overall node status is therefore **not fully healthy**

Verification rule:
- before calling Sepolia healthy, check repeated `eth_blockNumber`, `eth_syncing`, `optimism_syncStatus`, public Shape Sepolia head, public Ethereum Sepolia L1 head, and recent `op-node` logs together
- in reports for this user, split the status explicitly into execution-head health vs safe/finalized health instead of flattening it to a single green result

See `references/shape-sepolia-health-classification-2026-06.md`.

### Pitfall: live loopback RPCs must be checked from the VPS itself
When Shape services bind their RPCs to `127.0.0.1`, a direct probe from the operator machine will fail with connection-refused even if the node is otherwise healthy.

Operator rule:
- treat loopback-bound ports as VPS-local checks
- if an external probe to `http://<vps-ip>:<loopback-port>` fails, do **not** classify that alone as node failure
- run the health probe over SSH/on-host instead
- keep the lane's current port map explicit in notes/runbooks so you do not accidentally query the wrong service:
  - `op-node` rollup RPC: `127.0.0.1:19545`
  - `op-reth` HTTP RPC: `127.0.0.1:18545`
  - execution multiplexer: `127.0.0.1:18580`
  - beacon multiplexer: `127.0.0.1:18581`
- when turning that probe into unattended monitoring, deploy the cron script from `~/.hermes/scripts/`, not `~/.hermes/runtime/`, or Hermes cron will reject the job path

### Pitfall: preserved-datadir stalls need a split verdict, not one blended status
A fresh-key/provider cutover can be fully successful while the canonical L2 lane remains stalled for unrelated trie/Merkle reasons.

Operator rule for this user:
- report `provider path healthy again` separately from `node healthy again`
- once the provider path is proven good, stop attributing a flat canonical head to the key rotation itself
- if the user explicitly authorizes unattended incident work, it is valid to leave a small orchestrator plus a scheduled morning report in place so the least-destructive next repair branch can continue without waiting for the user to wake up
