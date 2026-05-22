---
name: shape-network-mainnet-op-reth-journey
description: Use when preparing, staging, or troubleshooting a Shape Network mainnet op-reth plus op-node deployment while preserving the current geth rollback path.
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [shape-network, shape-mainnet, op-reth, op-node, reth, migration, docker, rollback]
    related_skills: [shape-network-mainnet-node-recovery, shape-ai-agent, hermes-agent]
---

# Shape Network Mainnet op-Reth Journey

## Overview

This skill is for the **Shape Network mainnet Reth track**.

Use it when the job is no longer “fix the current geth stack” and has become:
- prepare Reth safely
- stand up Reth in parallel
- measure whether Reth is actually healthy
- decide whether to cut over or roll back

This is explicitly Shape-specific.

Documentation/reporting standard for this class of task:
- err toward more operator detail, not less
- record staged expectations, rollback constraints, and concrete failure classification
- interpret zero EL peers as expected current topology unless Shape changes the network model

Reference:
- `references/shape-mainnet-reth-reality-notes.md`
- `references/clean-preupload-baseline.md`

## When to Use

Use it when:
- the user wants to work on Reth later today
- a dedicated Reth bring-up is the main task
- the geth path is already good enough as fallback
- you need migration discipline, not just incident recovery discipline

Do not use it when:
- the primary need is just reviving the current geth node
- the system is still in immediate outage mode and rollback has not been stabilized

## Core rules

1. Preserve the current geth rollback path.
2. Do not treat `unsafe_l2` movement as success.
3. Report Shape block numbers in decimal.
4. Treat Shape mainnet specifics as first-class constraints.
5. Prefer `op-reth` over generic `reth` unless there is a deliberate reason otherwise.

## Shape-specific realities

- `Jovian` matters.
- docs may lag runtime truth.
- zero EL peers is not the main success metric.
- current Shape mainnet EL peering is intentionally not enabled, sequencing is centralized, and there are no EL bootnodes yet.
- chain-spec handling may matter more than operators expect.

## Canonical clean-base paths

For the next Reth bring-up track, prefer these canonical paths:
- upload staging: `/root/shape-mainnet-op-reth-upload`
- runtime Reth data: `/root/shape-mainnet-op-reth-data`
- runtime Reth op-node data: `/root/shape-mainnet-op-node-reth-data`
- Reth config: `/root/.shape-mainnet-op-reth-config`

Avoid reviving old `reth-fresh` experiment paths when preparing a clean retry.

In prep-only mode, stale Reth-only scripts/logs from earlier extraction attempts should also be cleared so the next upload starts from a genuinely clean baseline.

## Fast workflow

Prep-only variant before upload completes:
1. confirm geth rollback path stays untouched
2. remove stale Reth-only experiment dirs and ad hoc prep leftovers
3. recreate or confirm canonical clean-base Reth dirs
4. do **not** start Reth services yet
5. wait for upload completion, then validate data before bring-up

Full bring-up variant:
1. capture rollback state
2. verify storage and staging paths
3. validate source data
4. define exact Reth/node versions and chain-spec strategy
5. bring up Reth in isolation
6. sample health repeatedly
7. classify honestly: healthy, converging, fake movement, or broken

## Red flags

- execution head flat while `unsafe_l2` moves
- repeated forkchoice/canonical-head complaints
- chain-spec/fork mismatch symptoms
- pressure to discard geth before Reth proves itself

## Interpretation rule for peer count

If `net_peerCount = 0` on current Shape mainnet:
- do not treat that alone as failure
- do not burn time hunting nonexistent EL bootnodes
- shift debugging to execution head movement, chain spec, engine wiring, fork handling, and data quality

Zero EL peers also does **not** automatically mean the Reth plan is impossible.

Instead ask:
- can `op-reth` serve execution state correctly
- can `op-node` drive it correctly over Engine API
- does execution head actually move
- does lag against public Shape RPC improve

## Role of the current geth node

Treat the current geth node as:
- rollback safety
- a control sample for execution-head comparisons
- a reference for known-good Shape behavior

Do not treat the current geth node as:
- a source of EL peers for Reth
- proof that Reth is healthy

## Verification checklist

- [ ] geth rollback preserved
- [ ] Reth paths isolated
- [ ] execution head sampled repeatedly
- [ ] public comparison sampled repeatedly
- [ ] fake movement ruled out
- [ ] cutover not declared prematurely
