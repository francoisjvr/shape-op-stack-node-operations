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

Known staging path already prepared:
- `/root/shape-mainnet-op-reth-upload`

Suggested separation model:
- uploaded unpacked Reth source data in staging path
- runtime Reth data path separate from geth data path
- explicit config directory for Reth-specific files

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
