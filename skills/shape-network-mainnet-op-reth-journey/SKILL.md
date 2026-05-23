---
name: shape-network-mainnet-op-reth-journey
description: Use when setting up, operating, or troubleshooting the primary Shape Network mainnet op-reth plus op-node stack. op-geth is sunset and retained only as archival rollback context where still relevant.
version: 1.1.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [shape-network, shape-mainnet, op-reth, op-node, reth, migration, docker, rollback]
    related_skills: [shape-network-mainnet-node-recovery, shape-ai-agent, hermes-agent]
---

# Shape Network Mainnet op-Reth Journey

## Overview

This skill is for the **primary Shape Network mainnet Reth stack**.

Use it when the job is:
- set up Reth as the main Shape mainnet path
- operate or troubleshoot the live Reth stack
- verify whether Reth is actually healthy
- document cutover, retirement of legacy geth, or rollback conditions if any old lane still exists

This is explicitly Shape-specific.

Documentation/reporting standard for this class of task:
- err toward more operator detail, not less
- record staged expectations, rollback constraints, and concrete failure classification
- interpret zero EL peers as expected current topology unless Shape changes the network model

Reference:
- `references/shape-mainnet-reth-reality-notes.md`
- `references/shape-mainnet-el-peering-note.md`
- `references/clean-preupload-baseline.md`
- `references/runtime-datadir-promotion.md`
- `references/first-parallel-runtime-attempt.md`
- `references/current-working-runtime-vs-official-docs.md`
- `references/current-recommended-recipe.md`

## When to Use

Use it when:
- a dedicated Reth bring-up, setup, or runtime fix is the main task
- `op-reth` plus `op-node` is the canonical Shape mainnet path being documented
- any remaining geth lane is only fallback, rollback, or archival context
- you need mainnet operating discipline, not legacy geth incident-response discipline

Do not use it when:
- the primary need is just reviving the old geth node
- the task is generic Ethereum/geth troubleshooting rather than Shape mainnet Reth

## Core rules

1. Treat `op-reth` plus `op-node` as the main Shape mainnet path.
2. Do not treat `unsafe_l2` movement as success.
3. Report Shape block numbers in decimal.
4. Treat Shape mainnet specifics as first-class constraints.
5. Keep repo docs / skill notes synchronized with what was actually observed.
6. Use explicit standard paths and keep `/root/Upload` as backup, download cache, or transfer landing zone rather than the default live Reth mount.
7. When a fresh uploaded datadir arrives, validate it structurally before any bring-up and prefer in-place use over a full copy when disk is tight.
8. If the uploaded datadir becomes the runtime datadir, record that rename/move decision in repo docs immediately and snapshot the uploaded config files separately.
9. Treat `op-geth` as sunset; keep it only as rollback or archival context if it still exists on the box.

## Shape-specific realities

- `Jovian` matters.
- docs may lag runtime truth.
- zero EL peers is not the main success metric.
- current Shape mainnet EL peering is intentionally not enabled because sequencing is centralized.
- there are currently no EL bootnodes, and ELs should have no peers under the present network model.
- Shape indicated EL bootnodes should only appear once EL-sync is enabled later; until then, do not treat peer-hunting as a productive debugging path.
- chain-spec handling may matter more than operators expect.
- current `op-reth` built-in `shape` chain support may lag the live Shape fork set; verify `dump-genesis --chain shape` before trusting it.

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
4. if the upload is structurally sound and disk is tight, move or rename the uploaded datadir into the canonical runtime path instead of copying it
5. snapshot uploaded config artifacts into the canonical config dir, including `reth.toml`, `rollup.json`, `genesis-l2.json`, and any peer metadata worth preserving
6. remove stale source lock files only after confirming no Reth process is using the datadir
7. define exact Reth/node versions and chain-spec strategy
8. bring up Reth in isolation
9. sample health repeatedly
10. classify honestly: healthy, converging, fake movement, or broken

## Red flags

- execution head flat while `unsafe_l2` moves
- repeated forkchoice/canonical-head complaints
- chain-spec/fork mismatch symptoms
- pressure to discard geth before Reth proves itself
- startup failures from forgotten port collisions with the live geth stack, especially default P2P port `30303`
- `eth_syncing=false` on op-reth while op-node still rejects newest unsafe payload insertion
- `op-node` repeatedly requesting a large missing unsafe range while `op-reth` stays at a flat execution head and reports it is still syncing

## First parallel bring-up pitfalls

When bringing up `op-reth` beside a live geth rollback stack:

1. Do not trust built-in `op-reth --chain shape` blindly. Verify it first with `dump-genesis --chain shape` or equivalent and compare the exposed fork times against the currently working Shape runtime.
2. If built-in chain support lags the live fork set, create explicit runtime chain files from the uploaded `genesis-l2.json` and `rollup.json`, then align them to the currently working live overrides before first startup.
3. Do not combine `--disable-discovery` with `--disable-dns-discovery` on `op-reth`; current images reject that flag pair.
4. If the live geth stack is still running on host networking, move parallel `op-reth` off default P2P port `30303` before treating the startup as a deeper failure.
5. Classify the first attempt honestly: if `op-reth` serves RPC and `op-node` derives, but the execution head stays flat over repeated decimal block-number samples, call it stalled or only weakly converging, not healthy.

## Interpretation rule for peer count

If `net_peerCount = 0` on current Shape mainnet:
- do not treat that alone as failure
- do not burn time hunting nonexistent EL bootnodes
- remember the current network expectation is zero EL peers until Shape enables EL-sync and later introduces EL bootnodes
- shift debugging to execution head movement, chain spec, engine wiring, fork handling, and data quality

Zero EL peers also does **not** automatically mean the Reth plan is impossible.

Instead ask:
- can `op-reth` serve execution state correctly
- can `op-node` drive it correctly over Engine API
- does execution head actually move
- does lag against public Shape RPC improve

## Prep-only upload validation

When the user says the Reth data upload is finished and wants a look without starting services:

1. Verify the uploaded datadir is structurally real, not a placeholder. Expect at minimum `db/`, `static_files/`, `blobstore/`, `invalid_block_hooks/`, `reth.toml`, `rollup.json`, and `genesis-l2.json`.
2. Confirm `db/mdbx.dat` exists with substantial size and that header / receipt / transaction static files all exist in comparable counts.
3. Compare the highest static-file range against a live Shape height sample and the preserved geth comparator. Use decimal block numbers in the report.
4. Inspect `rollup.json` for Shape-specific upgrade times like holocene / isthmus so the dataset is not obviously using the wrong chain config.
5. Treat lock files such as `db/lock`, `db/mdbx.lck`, or `static_files/lock` as review / cleanup items before first startup, not automatic proof of corruption.
6. If free disk is lower than the uploaded datadir size, do not copy the upload into a second runtime datadir. Prefer either using the uploaded path directly or moving it into the canonical runtime path.
7. Keep the existing geth stack intact as rollback/control sample unless the user explicitly chooses otherwise.
8. Before first local startup, snapshot uploaded `reth.toml` / `rollup.json` / `genesis-l2.json` into the canonical config dir and remove stale source lock files if no Reth process is running.

Reference: `references/uploaded-datadir-validation.md`


## Role of any remaining geth lane

Treat any remaining geth node as:
- rollback safety during transition
- a control sample for execution-head comparisons
- archival reference for known-good Shape behavior

Do not treat any remaining geth node as:
- the main Shape mainnet path
- a source of EL peers for Reth
- proof that Reth is healthy

## Verification checklist

- [ ] geth rollback preserved
- [ ] Reth paths isolated
- [ ] execution head sampled repeatedly
- [ ] public comparison sampled repeatedly
- [ ] fake movement ruled out
- [ ] cutover not declared prematurely
