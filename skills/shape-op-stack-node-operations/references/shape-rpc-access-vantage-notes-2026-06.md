# Shape RPC access vantage notes — 2026-06

## Why this matters

Shape RPC health can differ by client vantage point. During this session:

- the VPS at `178.18.241.113` could query the private mainnet RPC at `http://127.0.0.1:18545` and the externally exposed private RPC at `http://178.18.241.113:18545`
- the same VPS received `HTTP 403` when querying the official public endpoints:
  - `https://mainnet.shape.network`
  - `https://sepolia.shape.network`
- from the Hermes host, those same public endpoints returned normal JSON-RPC responses and matched local node state

## Operator interpretation

Do not treat `403` from the node host to the official public Shape RPC as proof that the node is unhealthy or that chain data is bad.

First separate three questions:

1. is the local node advancing
2. is the private/local RPC serving correct data
3. is the public endpoint reachable from this specific machine

Only the first two are required to judge node health.

## Practical verification pattern

When the node host gets `403` from the official public RPC:

- verify local/private RPC head movement first
- verify `eth_syncing`
- verify `optimism_syncStatus`
- verify local log progress
- if you still need public parity, query the public endpoint from a different vantage point that is allowed, then compare block number and block hash at a common height

## Transaction-sending implication

If transactions fail or behave inconsistently through the public endpoint, retry against the private RPC before blaming fee settings or chain conditions. In this session, transaction sends succeeded via the private RPC after the public RPC path was suspected.
