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

## 6. Fake progress is common

The single most important Reth warning pattern is:
- `unsafe_l2` moves
- execution head does not

That is not success.
