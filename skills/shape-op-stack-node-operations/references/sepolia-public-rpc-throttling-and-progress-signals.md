# Shape Sepolia public-RPC throttling and progress signals

Use this when bringing up an isolated Shape Sepolia `op-reth` + `op-node` pair against public Ethereum Sepolia L1 endpoints.

## Problem pattern

A naive public-RPC setup can fail in two different ways:

1. **Immediate init failure**
   - `op-node` fails during L1 validation with `failed to get L1 chain ID: 429 Too Many Requests: Rate limited`.
2. **Post-init receipt-fetch stalls**
   - `op-node` starts, but logs repeated `Engine temporary error` messages while fetching L1 receipts, often from public endpoints that batch aggressively or enforce low anonymous quotas.

## Known-good public endpoint combo from this session

- L1 execution RPC: `https://ethereum-sepolia-rpc.publicnode.com`
- L1 beacon RPC: `https://ethereum-sepolia-beacon-api.publicnode.com`

Avoid brittle/demo/public endpoints that return frequent 429s during derivation, especially when receipts must be fetched in batches.

## Throttling knobs that materially helped

For public L1 RPCs, start `op-node` conservatively:

- `--l1.max-concurrency=1`
- `--l1.rpc-max-batch-size=1`
- `--l1.rpc-rate-limit=4`
- `--l1.http-poll-interval=24s`
- `--l1.rpckind=basic`

These settings reduced public-RPC pressure enough for `op-node` to finish reset and begin advancing L1 origin.

## Progress signals to trust before the L2 head moves

If `eth_blockNumber` stays flat at first, do **not** conclude the node is dead until you also check:

- `optimism_syncStatus.current_l1.number` across repeated samples
- `optimism_syncStatus.head_l1.number` across repeated samples
- `op-node` logs for:
  - `completed reset of derivation pipeline`
  - `Reset of L1Retrieval done`
  - `Advancing bq origin ... originBehind=true`

This session showed a real intermediate state where:

- local `eth_blockNumber` remained flat
- `current_l1.number` advanced from zero to `10881133`, then `10881135`
- `op-node` logs showed derivation reset completion plus advancing batch-queue origin

Interpretation: the stack is deriving and replaying backlog even if execution head has not yet advanced during the short observation window.

## Operator takeaway

For Shape Sepolia bring-up on public infrastructure:

1. keep the testnet stack isolated from mainnet containers, ports, and datadirs
2. prefer stable publicnode Sepolia execution + beacon endpoints over demo or allnodes-like public endpoints
3. aggressively lower `op-node` L1 concurrency and batching before declaring the stack broken
4. sample `optimism_syncStatus` more than once before judging health
5. distinguish **internal derivation progress** from **visible L2 head movement**
