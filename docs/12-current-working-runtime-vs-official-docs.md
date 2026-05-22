# Current Working Runtime vs Official Docs

This note compares the **currently working parallel Shape mainnet Reth track** on the VPS against the public Shape docs.

Primary public reference:
- Shape Docs, `Technical Details -> Run a Node`
- URL: `https://docs.shape.network/technical-details/run-a-node`

This is not a complaint about the public docs.
It is an operator note about what is currently working **for this exact VPS and migration track**.

## What currently works on our VPS

Observed from the live parallel Reth stack on `178.18.241.113`:

- container `shape-mainnet-op-reth` is up
- container `shape-mainnet-op-node-reth` is up
- `op-reth` image: `us-docker.pkg.dev/oplabs-tools-artifacts/images/op-reth:v2.2.2`
- `op-node` image: `us-docker.pkg.dev/oplabs-tools-artifacts/images/op-node:v1.18.0`
- `op-reth` is serving:
  - HTTP on `18545`
  - WS on `18546`
  - Engine/AuthRPC on `18551`
- `op-node` RPC is on `19545`
- `op-node` is wired with `--l2.enginekind=reth`
- the stack uses explicit runtime config files instead of trusting only built-in network shortcuts:
  - `/config/reth.uploaded.toml`
  - `/config/genesis-l2.runtime.json`
  - `/config/rollup.runtime.json`
- the uploaded Reth datadir is the runtime datadir at `/root/shape-mainnet-op-reth-data`
- the uploaded datadir was **moved**, not copied, because disk pressure made a second full copy unattractive

Live sample captured during this note:

- `op-reth eth_blockNumber = 28622780`
- public Shape head = `28864978`
- lag = `242198` blocks
- `eth_syncing = false`
- `op-node safe_l2.number = 28622780`
- `op-node unsafe_l2.number = 28622780`
- `op-node finalized_l2.number = 28622367`

Interpretation:
- the stack is alive
- RPC works
- `op-node` is wired to `op-reth`
- this is still a parallel migration track, not a cutover-ready declaration

## What is different from the public Shape run-a-node page

## 1. Public docs are centered on `op-geth`; our working parallel track is `op-reth`

The public page describes the standard pair:
- execution client: `op-geth`
- consensus client: `op-node`

Our current working track is instead:
- execution client: `op-reth`
- consensus client: `op-node`

That changes the practical recipe materially.

## 2. Public docs show source builds; we are using pinned Docker images

Public page:
- clone `op-geth`
- clone `optimism`
- build binaries locally

Our current working track uses pinned images directly:
- `op-reth:v2.2.2`
- `op-node:v1.18.0`

For this VPS, the pinned-image path is the path that actually produced a live parallel Reth stack.

## 3. Public docs use generic home-directory paths; we are using isolated root-owned canonical paths

Public page examples use:
- `$HOME/.shape`
- `$HOME/.shape-config`

Our current track uses explicit isolated paths:
- `/root/shape-mainnet-op-reth-upload`
- `/root/shape-mainnet-op-reth-data`
- `/root/shape-mainnet-op-node-reth-data`
- `/root/.shape-mainnet-op-reth-config`

Reason:
- this track is running beside a preserved geth rollback stack
- isolation matters more than pretty defaults

## 4. Public docs assume default P2P ports; our parallel deployment must avoid the live geth stack

Public page lists:
- `30303` for execution client P2P
- `9222` for `op-node` P2P

Our current parallel Reth runtime uses:
- `op-reth --port=31303`
- `op-node --p2p.listen.tcp=19222`
- `op-node --p2p.listen.udp=19222`

Reason:
- the live geth fallback stack is still running
- default ports would collide

This already caused a real startup failure earlier when `30303` was left in use.

## 5. Public docs show `op-node --syncmode=execution-layer`; our working Reth track is using `consensus-layer`

Public Shape mainnet page shows:
- `--syncmode=execution-layer`
- `--l2.enginekind=geth`

Our current working parallel Reth runtime uses:
- `--syncmode=consensus-layer`
- `--l2.enginekind=reth`

This is one of the biggest practical differences between the public op-geth path and the working parallel Reth path on this VPS.

## 6. Public docs show CLI hardfork overrides on op-node; our current runtime relies on explicit runtime config files

Public Shape mainnet page shows explicit op-node flags:
- `--override.granite=1727370000`
- `--override.holocene=1739880000`
- `--override.isthmus=1774530000`

Our current running command line does **not** pass those override flags directly.
Instead it uses:
- `/config/genesis-l2.runtime.json`
- `/config/rollup.runtime.json`

That does not prove the public docs are wrong.
It does mean our currently working path depends on explicit runtime config artifacts rather than only the public CLI example.

## 7. Public docs show `--rollup.sequencerhttp`; our Reth runtime also uses historical RPC

Public page op-geth example uses:
- `--rollup.sequencerhttp=https://mainnet.shape.network`

Our current `op-reth` runtime uses:
- `--rollup.sequencer=https://mainnet.shape.network`
- `--rollup.historicalrpc=https://mainnet.shape.network`
- `--rollup.disable-tx-pool-gossip`

That is a meaningful difference in the execution-client recipe.

## 8. Public docs present generic P2P expectations; our current Shape reality is that EL peering is not the main health signal

The public page still documents normal P2P exposure for the execution client.
Our live Shape-specific guidance and observations for the current migration track are different:

- execution-layer peering is intentionally not the main success metric right now
- there are currently no EL bootnodes to chase
- zero EL peers does not by itself disqualify the Reth plan
- execution head movement and shrinking lag matter more than peer count

That is why the current `op-reth` runtime is running with:
- `--disable-discovery`

And why the repo keeps stressing:
- do not use peer count alone as the health discriminator for current Shape mainnet

## 9. Public docs describe the snapshot path as fast; our real Reth track has been slower and less smooth

Public page says snapshot sync should be close to tip within minutes.

Our real first parallel Reth attempt did **not** look like a clean near-tip handoff.
Observed behavior included:
- an initially flat execution head
- `op-node` derivation continuing while execution lagged
- a later transition into slow forward convergence

So the snapshot path is still useful, but on this Reth track it did **not** translate into instant healthy convergence.

## 10. Public docs say to pass bootnodes directly when needed; our current working track does exactly that

This is one area where the public page and our working setup align.

Current `op-node` runtime includes explicit Shape bootnodes:
- `34.228.226.23:30301`
- `3.85.27.162:30301`

So this repo should not frame the public docs as wholly mismatched.
Some parts were directly useful.

## Practical operator conclusion

For this VPS and this migration phase, the public Shape page is best treated as:
- a baseline reference
- a source of network constants and standard client relationships
- not the full exact recipe for the currently working parallel Reth setup

The most important operator differences are:
- use `op-reth`, not the documented `op-geth` path
- preserve geth fallback and avoid default port collisions
- prefer explicit runtime config files over blind trust in built-in network shortcuts
- judge success by execution-head movement and lag shrinkage, not by peer count alone
- expect the imported Reth path to converge more awkwardly than the public snapshot wording implies

## Recommended repo posture

Keep both statements true at once:

1. the public Shape docs are still the canonical public starting point
2. this repo records the stricter reality that actually worked on our VPS

That is the only sane way to avoid re-learning the same Reth migration lessons later.
