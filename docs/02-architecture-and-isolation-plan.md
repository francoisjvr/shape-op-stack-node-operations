# Architecture and Isolation Plan

The upcoming Shape mainnet Reth work should not stomp on the recovered geth stack.

## Principle

Treat Reth as a parallel migration track until it proves itself.

## Separate concerns

### Existing geth fallback
- preserve current geth datadir and recovery notes
- do not silently reuse it for Reth
- keep it available for rollback and comparison

### New Reth track
Use distinct paths for:
- config
- data
- uploads
- logs if practical
- compose/runtime definitions if practical

## Directory planning

Canonical clean-base layout:
- upload staging: `/root/shape-mainnet-op-reth-upload`
- runtime Reth data: `/root/shape-mainnet-op-reth-data`
- runtime Reth op-node data: `/root/shape-mainnet-op-node-reth-data`
- Reth config: `/root/.shape-mainnet-op-reth-config`

Rules:
- do not reuse old `reth-fresh` naming from previous experiments
- keep uploaded source data separate from runtime datadirs unless there is a deliberate reason to run in place
- keep Reth runtime data separate from geth rollback data
- put chain-spec and rollup artifacts in the dedicated Reth config dir once finalized

## Port planning

Goal:
- avoid collisions with the current live stack
- allow comparison during bring-up if needed

Typical areas to isolate:
- HTTP RPC
- WS RPC
- auth RPC / Engine API
- metrics
- discovery / p2p ports

## Operational stance

Until cutover:
- geth is fallback truth
- Reth is candidate truth
- public Shape RPC is comparison truth
