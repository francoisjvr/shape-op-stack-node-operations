# Live merkle/unwind/watchdog notes (2026-06)

Use this note when the preserved Shape mainnet Reth datadir is still the right target and you need the next least-destructive repair step after a clean `repair-trie --dry-run`.

## When this branch is justified
- canonical `eth_blockNumber` is flat across repeated samples
- `head + 1` is still missing (`eth_getBlockByNumber` returns `null`)
- `eth_syncing` / stage evidence shows `Execution` and `Finish` near tip but `MerkleChangeSets` far behind
- `repair-trie --dry-run` reports `0 inconsistencies`

## Correct command-shape findings for this image/build

### Merkle stage run
Use:
- `op-reth stage run --chain shape --config /root/.shape-mainnet-op-reth/reth.runtime.toml --datadir /root/shape-mainnet-op-reth-data merkle --commit`

Do **not** use:
- `op-reth db ... stage run merkle`

### Stage CLI lives at image entrypoint
Inside the current `us-docker.pkg.dev/oplabs-tools-artifacts/images/op-reth:v2.2.2` image, invoke the binary subcommands directly.

Use:
- `docker run --rm ... <image> stage drop ...`
- `docker run --rm ... <image> stage unwind to-block <TARGET> ...`

Do **not** prepend another `op-reth` token inside `docker run`, or you will get an unrecognized-subcommand failure.

### Unwind syntax
For this build, the unwind target is positional under `to-block`.

Use the help path first if rusty:
- `stage unwind to-block --help`

Operationally, expect the real form to be equivalent to:
- `... stage unwind to-block <TARGET> --chain shape --config ... --datadir ...`

## Storage-settings warning interpretation
During offline stage work on the preserved datadir, this warning may appear:
- `Storage settings mismatch detected. Using the stored settings from the existing database. stored=StorageSettings { storage_v2: false } requested=StorageSettings { storage_v2: true }`

Interpretation for this branch:
- treat it as an observed compatibility warning first, not an automatic stop condition
- continue reading the stage command's real exit/result before escalating
- only treat it as the blocker if the stage operation itself fails or refuses to proceed

## Monitoring / orchestration pattern that held up better
For unattended or overnight live-lane work:
- keep the branching repair orchestration in one SSH-driven script
- keep health reporting in a separate watchdog script
- deploy the watchdog through Hermes cron from `~/.hermes/scripts/`, not `~/.hermes/runtime/`
- prefer on-host SSH probes because the live rollup + reth RPCs are loopback-bound

Live mainnet loopback ports used here:
- `op-reth` HTTP RPC: `127.0.0.1:18545`
- `op-node` rollup RPC: `127.0.0.1:19545`

## Reporting rule
Split outcomes into:
- `repair action launched / completed`
- `canonical head actually advancing`
- `overnight watchdog armed`

Do not collapse those into one success statement until repeated decimal head samples prove real advancement.
