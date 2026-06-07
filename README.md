# Shape OP Stack Node Operations

Reth-first operator repo for the **primary Shape Network mainnet** `op-reth` + `op-node` stack.

This is the canonical mainnet operations repo for bring-up, health checks, cutover, rollback, and recovery on the current Shape Reth lane.

The old geth runbook remains useful only for:
- archival incident history
- rollback context while any legacy lane still exists
- understanding why `/root/Upload` mattered on the old stack

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
- [Current recommended recipe](docs/13-current-recommended-recipe.md)
- [First-time operator quickstart](docs/14-first-time-operator-quickstart.md)
- [Copy/paste bring-up checklist](docs/15-copy-paste-bring-up-checklist.md)
- [Shape Sepolia testnet lane](docs/16-shape-sepolia-testnet-lane.md)
- [Shape RPC and Sepolia provider notes](docs/17-shape-rpc-and-sepolia-provider-notes.md)
- [Skill](skills/shape-op-stack-node-operations/SKILL.md)

## Practical files

For operators who want something more concrete than prose:
- `examples/.env.example`
- `examples/docker-compose.recommended.yml`
- `config/docker-compose.reference.yml`
- `templates/experiment-log-template.md`

## RPC provider note

Current Shape docs now make the provider split more explicit, including for **Shape Sepolia**:
- public RPC exists and is useful for light/manual checks
- for production or high-throughput testing, Shape points operators toward a node provider such as Alchemy

Operationally, that matches the blunt lesson here too:
- a paid provider endpoint tends to behave better than the public endpoint when you need repeated comparisons, automation, or steadier throughput

See:
- `docs/17-shape-rpc-and-sepolia-provider-notes.md`

## Core safety rule

Reth is the main path.

Use standard Reth runtime paths for the live node:
- `/root/shape-mainnet-op-reth-data`
- `/root/shape-mainnet-op-node-reth-data`
- `/root/.shape-mainnet-op-reth-config`

`/root/Upload` should be treated as optional support storage:
- backup copy
- download cache
- transfer landing zone

If any legacy geth lane still exists during transition:
- preserve `/root/Upload` until you have verified it is no longer needed
- keep rollback capability only as long as the sunset period requires
- keep Reth paths and ports isolated from old services

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

Companion archival repo:
- [francoisjvr/shape-mainnet-node-runbook](https://github.com/francoisjvr/shape-mainnet-node-runbook)

Use that repo only for:
- the already-proven geth recovery history
- legacy rollback notes while geth is being sunset
- incident history and why `/root/Upload` mattered on the old stack

Use this repo for:
- the live and future Shape mainnet Reth setup
- the canonical operator path
- setup, health checks, and cutover criteria for `op-reth` + `op-node`
