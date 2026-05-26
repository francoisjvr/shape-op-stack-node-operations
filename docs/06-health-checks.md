# Health Checks

## Minimum set

These examples use the **current isolated parallel-Reth ports**, not the old default geth-style ones.

## Current ports for the parallel Reth track

- `op-reth` HTTP RPC: `18545`
- `op-reth` WS RPC: `18546`
- `op-reth` Engine/Auth RPC: `18551`
- `op-node` RPC: `19545`
- `op-node` metrics: `17300`

### Local execution head
```bash
curl -s -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}' \
  http://127.0.0.1:18545
```

### Public Shape head
```bash
curl -s -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}' \
  https://mainnet.shape.network
```

Provider note:
- the public endpoint is fine for quick manual checks
- for repeated sampling, automation, or heavier throughput, prefer a provider endpoint
- current Shape docs explicitly steer production/high-throughput usage toward a node provider such as Alchemy, and in practice a paid provider tends to behave better than the public endpoint

### Sync state
```bash
curl -s -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","method":"eth_syncing","params":[],"id":1}' \
  http://127.0.0.1:18545
```

### Rollup sync status
```bash
curl -s -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","method":"optimism_syncStatus","params":[],"id":1}' \
  http://127.0.0.1:19545
```

## Interpretation

Healthy trend:
- execution head rises
- lag shrinks
- logs show real forward work

Unhealthy trend:
- execution head flat
- only `unsafe_l2` rises
- logs repeat forkchoice/payload/canonical-state complaints

## Comparator rule

If the current geth node is available, use it as a comparator during Reth bring-up:
- compare geth execution head versus Reth execution head
- compare whether both behave plausibly against public Shape RPC
- use geth as rollback truth if Reth behavior is ambiguous

Do not treat geth as a peer source for Reth.

## Sampling rule

Never decide from one sample.

Take at least 3 samples, spaced 2 to 5 minutes apart.
