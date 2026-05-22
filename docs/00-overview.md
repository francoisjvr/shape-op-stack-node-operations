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
- prepared Reth upload staging dir: `/root/shape-mainnet-op-reth-upload`
- user preference: try Reth first for future Shape experiments
- user wants Reth status reported directly, not legacy geth chatter, unless asked
