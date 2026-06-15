# Alchemy key rotation via local L1 multiplexer (2026-06)

Use this note when the live Shape mainnet lane must switch away from a near-exhausted Alchemy key without teaching `op-node` another direct remote URL.

## Proven live pattern

Keep `op-node` pointed at stable loopback URLs:
- execution: `http://127.0.0.1:18580`
- beacon: `http://127.0.0.1:18581`

Serve those from a local systemd unit:
- `shape-mainnet-l1-multiplexer.service`

During the live cutover in this session, the older direct failover units were disabled so there was only one active provider-control plane:
- `shape-mainnet-l1-rpc-failover.service` -> disabled
- `shape-mainnet-l1-beacon-failover.service` -> disabled

## Key-rotation rule

If the user says the current Alchemy key is close to quota exhaustion, do not leave it first in rotation just because it still works.

Preferred operator move:
1. put the two fresh Alchemy keys at the front of the multiplexer env
2. remove the near-exhausted old key from the active live multiplexer env
3. keep any public tertiary fallbacks behind the fresh paid keys
4. restart the multiplexer first
5. only then recreate/restart `op-node` if needed to move it from direct remote URLs to the stable loopback URLs

## Verification checklist

Verify the provider path separately from node recovery.

Provider-path proof:
- multiplexer service is `active`
- direct proxied execution request returns `200`
- direct proxied beacon request returns `200`
- proxy health output shows labeled upstreams, for example `alchemy-fresh-a` and `alchemy-fresh-b`
- at least one direct request confirms the currently selected upstream label is one of the fresh keys

Node-recovery proof:
- repeated decimal local `eth_blockNumber` samples
- `optimism_syncStatus`
- local-vs-public Shape head comparison
- recent `op-node` logs after restart/recreate

## Reporting rule

After a key-rotation cutover, keep the report split into two layers:
- `L1 provider path switched successfully`
- `node healthy again` or `node still settling/unhealthy`

Do not flatten those into one claim. A successful fresh-key cutover can still leave `op-node` in a rewind/reset window where `current_l1` starts repopulating before the local Shape head resumes moving.
