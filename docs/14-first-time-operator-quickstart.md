# First-Time Operator Quickstart

This page is for the person who wants the **least confusing safe path**.

If you are a noob, use this order and resist the temptation to improvise.

## What you are trying to do

You are bringing up a **parallel Shape mainnet Reth stack**:
- `op-reth` for execution
- `op-node` for consensus/derivation
- while keeping any existing geth node intact as rollback

This is **not** a cutover guide.
It is a "stand it up safely and see if it is actually healthy" guide.

## Before you touch anything

Make sure you have all of these:

- a VPS with decent SSD space
- your L1 execution RPC URL
- your L1 beacon RPC URL
- a validated Shape Reth datadir
- explicit config artifacts:
  - `reth.toml`
  - `rollup.json`
  - `genesis-l2.json`
- a JWT secret file shared between `op-reth` and `op-node`

If you do **not** have those yet, stop here and fix that first.

## The safest mental model

Treat the machine as having **two lanes**:

1. **existing geth lane**
   - already known fallback
   - do not disturb it

2. **new Reth lane**
   - isolated ports
   - isolated data paths
   - isolated config files
   - allowed to fail without taking down the fallback

If you blur those lanes, you create the kind of mess that wastes half a day.

## Use the practical files in this repo

Start from these files:

- `examples/.env.example`
- `examples/docker-compose.recommended.yml`
- `docs/13-current-recommended-recipe.md`

Suggested workflow:

1. copy `examples/.env.example` to `.env`
2. replace the placeholder RPC URLs and bootnodes
3. confirm the host paths match your machine
4. confirm the config filenames match the files you actually have
5. only then use the compose file as your starting point

## Things a noob should NOT do

- do **not** point Reth at the existing geth datadir
- do **not** reuse default ports if geth is already running on the box
- do **not** trust peer count as the main health signal on current Shape mainnet
- do **not** assume `eth_syncing=false` means success
- do **not** declare victory because containers started
- do **not** delete uploaded source data casually if it was expensive to obtain
- do **not** blindly trust built-in chain shortcuts if your explicit runtime files disagree

## Minimum safe startup order

1. preserve rollback
   - keep the current geth stack intact
   - keep `/root/Upload` if it still matters as source material

2. validate the Reth dataset
   - check structure
   - check size sanity
   - check config artifacts exist

3. stage canonical paths
   - `/root/shape-mainnet-op-reth-data`
   - `/root/shape-mainnet-op-node-reth-data`
   - `/root/.shape-mainnet-op-reth-config`

4. prepare `.env`
   - fill in real RPC URLs
   - fill in validated bootnodes
   - confirm port choices

5. start `op-reth`
   - isolated ports
   - explicit config and chain files

6. start `op-node`
   - `--l2.enginekind=reth`
   - `--syncmode=consensus-layer`

7. sample health repeatedly
   - never from one sample only

## How to decide whether it is working

Good signs:
- local execution head rises
- lag versus public Shape head shrinks
- `safe_l2` and `unsafe_l2` move in a way that matches real execution progress

Bad signs:
- execution head stays flat for repeated samples
- only `unsafe_l2` moves
- logs keep complaining about payloads, forkchoice, or canonical state

## The trap that gets people

The biggest trap is this:

> "The processes started, so I must be close."

No.

For this track, **startup is not success**.
Success is:
- execution head moving
- lag shrinking
- behavior staying sane over repeated checks

## After startup, check these first

Use `docs/06-health-checks.md`.

If you only check one thing, check `eth_blockNumber` repeatedly and compare it to public Shape RPC in **decimal**.

## Bottom line

If you are unsure, be boring:
- isolate everything
- keep rollback
- use explicit files
- pin versions
- measure repeatedly
- do not freestyle around ports, chain files, or datadirs
