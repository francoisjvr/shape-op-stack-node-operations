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

## Fast workflow

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

## Verification checklist

- [ ] geth rollback preserved
- [ ] Reth paths isolated
- [ ] execution head sampled repeatedly
- [ ] public comparison sampled repeatedly
- [ ] fake movement ruled out
- [ ] cutover not declared prematurely
