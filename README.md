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
- [Skill](skills/shape-network-mainnet-op-reth-journey/SKILL.md)

## Core safety rule

Do not sacrifice the currently working geth stack just to “see if Reth works.”

Until Reth proves healthy:
- preserve `/root/Upload`
- preserve geth rollback capability
- keep Reth paths and ports isolated

## Key Shape-specific lessons already known

- use **`op-reth`**, not generic `reth`, unless there is a deliberate reason not to
- `op-node --network=shape-mainnet` does not imply `op-reth --chain shape-mainnet` will work
- `Jovian` needs explicit respect in docs, chain spec, and runtime assumptions
- `unsafe_l2` movement is not enough to call Reth healthy
- `net_peerCount = 0` on current Shape mainnet is not the main health discriminator
- re-extracting the same snapshot is not guaranteed to fix a stuck canonical head

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
