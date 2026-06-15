# Shape OP Stack reseed: confirmation gate + SIGPIPE pitfall

Use this when a Shape OP Stack node has already crossed from routine sync debugging into destructive recovery (reseed, datadir replacement, snapshot extraction, stop/move/restart of live services).

## Durable lessons

### 1) Put an explicit confirmation gate immediately before destructive reseed steps
For live node recovery, do discovery and staging first:
- locate/download the official snapshot
- verify archive size / headers
- preserve the current datadirs into a timestamped backup directory
- confirm the live datadir paths and service names

Then stop and ask for an explicit go-ahead right before the irreversible part:
- replacing the live datadir contents
- extracting the snapshot into the live path
- restarting the production services against the new data

Reason: destructive remote actions may be blocked by the safety guard if the user has not explicitly consented in the current turn. Treat this as a workflow requirement, not an optional courtesy.

### 2) Avoid `set -euo pipefail` + `... | head` archive preview traps
A preview command like:

```bash
tar -tzf "$ARCHIVE" | head -n 20 | tee "$NOTES/archive-head.txt"
```

can abort the whole orchestration with exit 141 (SIGPIPE) under `set -euo pipefail`, because `head` exits early while `tar` is still writing.

Safer patterns:
- use `tar -tzf "$ARCHIVE" | sed -n '1,20p' | tee ... || true`
- or temporarily relax `pipefail` for the preview segment
- or write the full listing to a file first, then inspect the first lines separately

Do not put a fragile preview pipeline ahead of extraction/restart in an all-or-nothing recovery script.

### 3) Post-reseed verification must prove canonical recovery, not just container uptime
After restart, verify with repeated samples, not a single ready check:
- local `eth_blockNumber` advances over time
- local head surpasses the previously stuck height
- formerly missing blocks in the bad range are now queryable via `eth_getBlockByNumber`
- `optimism_syncStatus` fields such as `safe_l1` / `finalized_l1` are no longer pinned abnormally
- public-vs-local gap narrows or at least changes in the expected direction during catch-up

For Shape work, prefer reporting block numbers in decimal in operator summaries.
