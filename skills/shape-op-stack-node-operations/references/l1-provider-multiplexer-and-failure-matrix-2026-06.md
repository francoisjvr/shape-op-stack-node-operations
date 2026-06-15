# L1 provider multiplexer and failure matrix (2026-06)

Use this when Morpheus asks how to rotate multiple Ethereum mainnet beacon or execution API keys **without** another disruptive `op-node` reconfiguration/restart cycle.

## Core operator pattern

Do **not** keep teaching `op-node` about individual provider keys.

Instead:
- give `op-node` one stable local loopback URL for beacon, e.g. `http://127.0.0.1:3500`
- optionally give it one stable local loopback URL for execution too, e.g. `http://127.0.0.1:8547`
- rotate real upstream keys behind that local proxy/multiplexer

This makes key rotation a proxy-only change after the initial cutover.

## Important truth

There is still **one initial controlled `op-node` restart** when cutting over from direct remote URLs to local loopback URLs.

After that:
- future beacon key rotation should only require proxy restart/reload
- future execution key rotation should only require proxy restart/reload if execution is proxied too
- changing keys should **not** imply datadir wipe or starting from zero

## Why `l1.beacon-fallbacks` is not the main fix

OP docs describe `--l1.beacon-fallbacks` / `OP_NODE_L1_BEACON_FALLBACKS` as fallback endpoints for blob sidecars not available at the primary beacon endpoint.

Treat it as:
- useful extra safety
- **not** the primary general key-rotation architecture

## Recommended cutover shapes

### Beacon-only cutover

Change only:
- `--l1.beacon=http://127.0.0.1:3500`

Keep:
- direct `--l1=${L1_RPC_URL}`
- current conservative L1 throttles if they are still justified

Use when:
- the immediate pain is beacon key churn
- you want the smallest maintenance change first

Tradeoff:
- future execution provider rotation can still require `op-node` config change/restart

### Full L1 cutover

Change to:
- `--l1=http://127.0.0.1:8547`
- `--l1.rpckind=basic`
- `--l1.beacon=http://127.0.0.1:3500`

Use when:
- you want to avoid a second future maintenance event just to proxy execution later

Why `basic`:
- once `op-node` talks to a generic local HTTP proxy rather than directly to Alchemy, the safest provider kind assumption is standard JSON-RPC

## Beacon health-check nuance

For Alchemy-backed beacon verification, prefer:
- `GET /eth/v1/beacon/headers/head`
- optionally `GET /eth/v1/node/version`
- optionally `GET /eth/v1/config/spec`

Do **not** rely on:
- `GET /eth/v1/node/health`

Observed behavior in this environment: Alchemy beacon returned `400 Unsupported method` there.

## Failure matrix summary

### Single beacon key fails (`429`, timeout, `5xx`)
- proxy should cool down that upstream and retry another
- `op-node` keeps the same local URL
- no restart-from-zero implication

### All beacon upstreams fail
- beacon path stalls until one upstream recovers or a new key is added
- no automatic datadir loss
- operator action is upstream restoration, not wipe-first panic

### Proxy process dies
- restart the proxy only
- if `restart: unless-stopped` is configured, Docker should attempt this automatically
- `op-node` config does not need to change

### VPS power cut with intact storage
- expected outcome is reboot + service restart + resume from persisted datadirs
- possible short replay/rewind is normal
- this is **not** the same as starting all over

### Disk loss or severe datadir corruption
- proxy architecture does not save you here
- repair or fresh snapshot/reseed may still be required

## Reporting rule for this user

When Morpheus asks 'if the VPS power-cuts, do we start all over anyway?', answer in two layers:
- provider resilience: proxy helps avoid provider-path restarts
- storage resilience: persistent datadirs determine whether reboot resumes or a rebuild is needed

Preferred phrasing:
- power cut with intact storage -> reboot and resume
- storage loss/corruption -> repair or reseed may be required

## Draft artifacts prepared in session

A local non-live draft set was produced under the Shape runbook repo at:
- `drafts/l1-multiplexer/`

Contents included:
- Python multiplexer draft
- env example with labeled upstreams
- draft compose service
- beacon-only and full-L1 cutover patches
- explicit failure matrix

Current draft convenience pattern:
- the local draft script can build both beacon and execution Alchemy URLs from one shared `ALCHEMY_ACCOUNT_KEYS=label=key,...` list
- this is the simplest fit when the operator has 2-3 separate Alchemy accounts and wants to avoid duplicating the same keys in both beacon and execution env blocks
- full manual `BEACON_UPSTREAMS` / `EXECUTION_UPSTREAMS` entries are still supported when exact URLs must be pinned

These were intentionally **not** implemented on the VPS during the session.
