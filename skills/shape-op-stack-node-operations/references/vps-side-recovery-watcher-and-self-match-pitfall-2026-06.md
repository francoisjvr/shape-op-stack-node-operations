# VPS-side recovery watcher and self-match pitfall (June 2026)

Use this when a long-running Shape mainnet offline maintenance step must outlive the local Hermes wrapper or SSH session.

## Durable findings

- A local Hermes/background watcher can fail even when the real remote repair is still fine.
- The safest control plane for long offline maintenance is a **VPS-side Python watcher** that runs on the node host itself, not a local wrapper that polls over SSH.
- Before branching recovery logic, verify the real maintenance truth from:
  - the remote Python watcher PID
  - the actual `docker run ... repair-trie` parent
  - the inner `/usr/local/bin/op-reth db ...` PID
  - MDBX lock ownership on the datadir
- For this lane, seeing the inner `op-reth db repair-trie --dry-run` at high CPU with the MDBX lock held was decisive evidence that the scan was still alive even when log output had paused.

## Specific shell pitfall

A naive `pgrep -af 'shape-mainnet-remote-recovery-vps.py'` check can match the shell wrapper command that is *performing the check*, not the real long-lived Python process.

Safer patterns:
- verify with `ps -eo pid,ppid,etime,cmd | grep -F 'python3 /root/shape-mainnet-reth-runbook-live/backups/shape-mainnet-remote-recovery-vps.py' | grep -v grep`
- or launch first, then confirm the specific PID with `ps -p <pid> -o pid,ppid,etime,cmd`
- avoid deciding `already running` from a broad `pgrep -af` hit alone

## Recommended operator pattern

1. Keep the node down for offline maintenance.
2. Start a VPS-side watcher script from the host.
3. Let that watcher run the dry-run directly and stream stdout to a VPS-local log file.
4. Branch only from the watcher's direct result:
   - if dry-run shows `StorageMissing(...)`, `Inconsistency found:`, or `inconsistent_nodes > 0`, run live `repair-trie`
   - if dry-run exits clean without fresh inconsistency and `MerkleChangeSets` is still the isolated lagging checkpoint, pivot to offline Merkle rebuild
5. Restart `op-reth`, wait for RPC, then restart `op-node`.
6. Keep a **separate** lightweight local notifier if you want a completion ping back in Hermes, but do not let that notifier own the recovery logic.

## Why this belongs in the class skill

This is not just a one-off error transcript. It changes the preferred orchestration architecture for any future Shape preserved-datadir maintenance that may outlive the local session.