# Overview

This repo is about one thing:

> bring up a **Shape Network mainnet** `op-reth` + `op-node` stack safely enough that it can eventually replace or supersede the current geth-based setup.

## What success looks like

A healthy Reth result should mean:
- `op-reth` runs stably
- execution `eth_blockNumber` advances in decimal over repeated samples
- lag versus public Shape RPC shrinks or disappears
- `op-node` no longer only shoves `unsafe_l2` forward while execution stays pinned
- the stack survives restart cleanly
- we can explain exactly why it works

## What does not count as success

These states do **not** count:
- `op-node` is noisy and “looks busy” but execution head is flat
- `unsafe_l2` moves while canonical execution stays pinned
- `op-reth` answers some RPCs but remains unusable as a real personal RPC
- the setup only works by destroying the current geth rollback path first

## Why Shape makes this special

Shape mainnet has a few operational properties that punish lazy assumptions:
- current execution peering expectations differ from generic L1 thinking
- docs may lag active fork/runtime reality
- chain-spec handling can matter more than operators expect
- public peer count is not the right success metric

## Most important starting facts

- current fallback geth datadir: `/root/Upload`
- current Reth upload staging dir: `/root/shape-mainnet-op-reth-upload`
- current Reth runtime dir: `/root/shape-mainnet-op-reth-data`
- clean Reth op-node dir: `/root/shape-mainnet-op-node-reth-data`
- clean Reth config dir: `/root/.shape-mainnet-op-reth-config`
- current runtime datadir is the validated uploaded dataset moved into `/root/shape-mainnet-op-reth-data`
- user preference: try Reth first for future Shape experiments
- user wants Reth status reported directly, not legacy geth chatter, unless asked

## Current prep-state rule

Do not resurrect old `reth-fresh` experiment paths.

The current prep baseline is:
- old Reth-only experiment dirs removed
- clean canonical Reth dirs recreated
- uploaded Reth datadir validated and renamed into canonical runtime path instead of copied
- upload staging path recreated empty for future use
- uploaded config artifacts copied into `/root/.shape-mainnet-op-reth-config`
- stale source lock files removed before first startup attempt
- no Reth services started yet
- current geth fallback left untouched
