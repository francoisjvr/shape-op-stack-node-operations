# Bring-up Plan

This is the disciplined order for the upcoming Shape mainnet `op-reth` work.

## 1. Freeze the rollback picture

Record:
- current geth health
- current local/public heads
- where rollback data lives
- current free disk

## 2. Prepare isolated Reth paths

Use dedicated paths for:
- upload/staging
- runtime data
- config

Current canonical base paths are:
- `/root/shape-mainnet-op-reth-upload`
- `/root/shape-mainnet-op-reth-data`
- `/root/shape-mainnet-op-node-reth-data`
- `/root/.shape-mainnet-op-reth-config`

Do not revive old `reth-fresh` paths just because they existed before.

## 3. Validate source data

Before blaming the runtime later:
- verify the uploaded/extracted data is what you think it is
- if using an archive, test integrity before extraction
- if using unpacked upload, verify directory structure and size sanity

## 4. Decide runtime recipe

Choose and document:
- exact `op-reth` image/version
- exact `op-node` image/version
- exact chain spec source
- explicit fork timestamps if needed
- first sync mode to try

## 5. Bring up execution first enough to be meaningful

Do not over-interpret a process merely listening on a port.

## 6. Bring up op-node against the execution engine

Verify the wiring is correct.

## 7. Sample health repeatedly

Track at minimum:
- local execution head in decimal
- public Shape head in decimal
- lag
- `eth_syncing`
- `optimism_syncStatus`
- recent logs

## 8. Classify outcome honestly

Possible outcomes:
- healthy and converging
- alive but not converging
- fake movement only
- startup/config failure
- storage/data-source failure
