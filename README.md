# Shape Mainnet op-Reth Journey

Detailed operator repo for bringing up **Shape Network mainnet** on `op-reth` + `op-node` without destroying the currently recovered geth fallback.

This repo is intentionally separate from the geth recovery runbook.

Reason:
- the geth repo records what already worked
- this repo tracks the Reth journey, experiments, guardrails, and final setup path we want to use next

## Scope

This repo covers:
- Shape mainnet `op-reth` planning and bring-up
- chain-spec and hardfork realities specific to Shape
- preflight checks before touching the VPS
- expected failure modes during Reth sync
- exact decision criteria for cutover vs rollback
- future operational notes once a stable Reth stack exists

## Start here

- [Overview](docs/00-overview.md)
- [Preflight checklist](docs/01-preflight-checklist.md)
- [Architecture and isolation plan](docs/02-architecture-and-isolation-plan.md)
- [Decision tree](docs/03-decision-tree.md)
- [Shape-specific Reth realities](docs/04-shape-specific-reth-realities.md)
- [Bring-up plan](docs/05-bring-up-plan.md)
- [Health checks](docs/06-health-checks.md)
- [Failure patterns](docs/07-failure-patterns.md)
- [Cutover and rollback](docs/08-cutover-and-rollback.md)
- [Known unknowns](docs/09-known-unknowns.md)
- [Current prep state](docs/10-current-prep-state.md)
- [First runtime attempt](docs/11-first-runtime-attempt.md)
- [Current working runtime vs official docs](docs/12-current-working-runtime-vs-official-docs.md)
- [Skill](skills/shape-network-mainnet-op-reth-journey/SKILL.md)

## Core safety rule

Do not sacrifice the currently working geth stack just to “see if Reth works.”

Until Reth proves healthy:
- preserve `/root/Upload`
- preserve geth rollback capability
- keep Reth paths and ports isolated

## Current prepared base state

The VPS has now moved from clean pre-upload baseline to **post-upload prep state** without starting any Reth services.

Canonical paths now are:
- upload staging: `/root/shape-mainnet-op-reth-upload`
- Reth runtime data: `/root/shape-mainnet-op-reth-data`
- Reth op-node data: `/root/shape-mainnet-op-node-reth-data`
- Reth config: `/root/.shape-mainnet-op-reth-config`

Current observed state:
- the uploaded Reth datadir was structurally validated
- because root free space was only about `59G`, the uploaded datadir was **moved** into `/root/shape-mainnet-op-reth-data` instead of copied
- `/root/shape-mainnet-op-reth-upload` was recreated as an empty future staging path
- uploaded `reth.toml`, `rollup.json`, `genesis-l2.json`, and `known-peers.json` were copied into `/root/.shape-mainnet-op-reth-config`
- stale source lock files were removed before any first local startup attempt
- first parallel runtime attempt now exists and is recorded in `docs/11-first-runtime-attempt.md`

Old experimental `reth-fresh` paths were removed first so the upload landed on a known clean base instead of ambiguous leftovers.

## Key Shape-specific lessons already known

- use **`op-reth`**, not generic `reth`, unless there is a deliberate reason not to
- `op-node --network=shape-mainnet` does not imply `op-reth --chain shape-mainnet` will work
- `Jovian` needs explicit respect in docs, chain spec, and runtime assumptions
- `unsafe_l2` movement is not enough to call Reth healthy
- `net_peerCount = 0` on current Shape mainnet is not the main health discriminator
- re-extracting the same snapshot is not guaranteed to fix a stuck canonical head

## What zero EL peers does and does not mean

Current Shape guidance implies zero EL peers does **not** automatically disqualify Reth.

It means the health model is different:
- do not ask whether Reth discovered peers
- ask whether `op-reth` can serve execution state correctly
- ask whether `op-node` can drive `op-reth` correctly over Engine API
- ask whether execution `eth_blockNumber` actually advances
- ask whether lag versus public Shape RPC shrinks

## How the current geth node helps

The current geth node is useful as:
- rollback safety
- a known-good comparator for execution head and sync behavior
- a reference for JWT handling, Shape-specific assumptions, and what healthy execution progress looks like

The geth node is **not** expected to:
- provide EL peers to Reth
- enable Shape EL peering by itself
- prove Reth healthy just because geth is healthy

## Relationship to the geth runbook

Companion repo:
- `shape-mainnet-node-runbook`

Use that repo for:
- the already-proven geth recovery
- the current stable fallback
- incident history and why `/root/Upload` matters

Use this repo for:
- the upcoming Reth journey
- fast-moving migration notes
- experiments, decision branches, and future cutover criteria
