# Shape mainnet geth recovery reference, May 2026

Use this note as a concrete companion to the umbrella skill `shape-op-stack-node-operations`.

## Scenario

A previously working Shape mainnet Docker stack failed to restart cleanly. The user had uploaded a large data folder and explicitly warned not to delete it because re-uploading would take a long time.

## Critical discoveries

- The folder that looked like a generic upload directory was actually the live `op-geth` datadir.
- The immediate restart blocker was root disk exhaustion, not protocol mismatch.
- After safe cleanup, `op-geth` still failed because the existing container itself was broken and exited with code `127`.
- The successful fix was to recreate the broken geth container against the same preserved datadir, then restart `op-node` after Engine API health returned.

## Proven-safe recovery sequence

1. Inspect container status and mounts before cleanup.
2. Confirm whether the uploaded folder is mounted as the live datadir.
3. Check root disk usage before restart attempts.
4. Free space only by deleting unused experiment or staging directories that are not mounted live.
5. Back up `docker inspect` output for `op-geth` before removing the container.
6. Recreate `op-geth` with the same image, mounts, host network mode, restart policy, and command flags.
7. Restart `op-node` only after geth is serving Engine API again.
8. Verify by comparing local and public Shape block numbers, checking `eth_syncing`, checking `optimism_syncStatus`, and reading recent logs.

## Shape-specific operational notes

- Public Shape block comparisons should be reported in decimal for this user.
- `net_peerCount = 0` was expected on this Shape deployment and did not indicate failure.
- `safe_l2` and `finalized_l2` can trail `unsafe_l2` without implying the earlier outage persists.
- Official docs were useful for public RPC and chain metadata, but insufficient for this recovery. The decisive truth came from mounts, inspect output, logs, and live RPC behavior.

## Docs-vs-reality findings worth remembering

- Superchain registry presents distinct public and sequencer RPC endpoints for Shape.
- The recovered working stack used `https://mainnet.shape.network` for both `--rollup.sequencerhttp` and `--rollup.historicalrpc`.
- Explicit fork overrides were part of the successful path and were safer than relying on embedded defaults after stale-config suspicion.
- A healthy local pairing was observed with `op-geth v1.101603.4` and `op-node v1.18.0`, even though upstream release notes may point to a different recommended partner version.

## Runbook-authoring lesson

When turning a recovery into documentation, include:
- exact mounts
- exact flags
- exact restart order
- safe-versus-unsafe cleanup guidance
- differences from official docs
- a warning when a misleading directory name hides production chain state
