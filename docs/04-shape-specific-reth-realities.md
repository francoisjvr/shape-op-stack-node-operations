# Shape-specific Reth Realities

These are the things most likely to bite if you treat Shape like generic Ethereum or a generic OP Stack chain.

## 1. Prefer `op-reth`

When targeting Shape mainnet in the OP Stack context, default to `op-reth` rather than generic `reth`.

## 2. Built-in names may differ across components

Observed behavior showed:
- `op-node --network=shape-mainnet` can be fine
- `op-reth --chain shape-mainnet` is not the same thing
- built-in chain handling may instead expect `shape`

## 3. Jovian matters

The chain crossed Jovian.

Practical consequence:
- a stale chain spec or stale assumptions can leave you debugging nonsense that looks like networking or sync trouble

## 4. Docs may lag runtime truth

Shape docs are useful, but they are not an oracle for what the chain is doing right now.

## 5. Zero execution peers is not the whole story

On current Shape mainnet, zero EL peers is not the clean universal sign of brokenness people expect.

More specifically, a Shape developer contact said the execution layer does not currently have peering enabled because sequencing is centralized. That means:
- there are no EL bootnodes right now
- ELs should have no peers right now
- EL bootnodes are expected only later, when Shape enables EL sync

Practical consequence:
- do not waste hours trying to "fix" EL peering that is intentionally absent
- if execution head is stuck, look at chain spec, hardfork handling, canonical-head state, engine wiring, or bad data before blaming missing peers

## 6. Zero peers does not automatically kill the Reth plan

Zero EL peers on current Shape mainnet does **not** by itself mean `op-reth` cannot be a working node.

The actual viability questions are:
- can `op-reth` read and serve the uploaded execution state correctly
- can `op-node` talk to `op-reth` correctly over Engine API
- does execution `eth_blockNumber` advance over repeated samples
- does lag versus public Shape RPC shrink

## 7. Geth helps as comparator, not as peer source

The current geth node is still useful during the Reth journey, but mainly as:
- rollback anchor
- control sample for execution head comparisons
- reference point for expected Shape behavior

Do not assume the geth node will:
- supply EL peers to `op-reth`
- make Shape EL peering exist before Shape enables it
- rescue a bad Reth canonical-head state automatically

## 8. Fake progress is common

The single most important Reth warning pattern is:
- `unsafe_l2` moves
- execution head does not

That is not success.
