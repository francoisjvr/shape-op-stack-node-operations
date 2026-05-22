# Health Checks

## Minimum set

### Local execution head
```bash
curl -s -H 'Content-Type: application/json'   -d '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}'   http://127.0.0.1:8545
```

### Public Shape head
```bash
curl -s -H 'Content-Type: application/json'   -d '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}'   https://mainnet.shape.network
```

### Sync state
```bash
curl -s -H 'Content-Type: application/json'   -d '{"jsonrpc":"2.0","method":"eth_syncing","params":[],"id":1}'   http://127.0.0.1:8545
```

### Rollup sync status
```bash
curl -s -H 'Content-Type: application/json'   -d '{"jsonrpc":"2.0","method":"optimism_syncStatus","params":[],"id":1}'   http://127.0.0.1:9545
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
