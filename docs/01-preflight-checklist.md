# Preflight Checklist

Run this before changing anything on the VPS.

## Safety and scope

- [ ] Confirm the current geth stack is preserved as rollback anchor
- [ ] Confirm `/root/Upload` is not being repurposed casually
- [ ] Confirm Reth work will use separate paths where possible
- [ ] Confirm documentation is being updated as decisions are made

## Storage

- [ ] Check root free space
- [ ] Check Docker usage
- [ ] Decide where extracted Reth data will live
- [ ] Verify enough room for both staging and runtime if needed

## Current live state

- [ ] Record current container status
- [ ] Record current local execution head
- [ ] Record public Shape head
- [ ] Record lag
- [ ] Record current geth rollback paths

## Reth artifacts

- [ ] Decide whether using uploaded unpacked data or other source
- [ ] Verify archive integrity if any archive is involved
- [ ] Verify extracted data looks complete before startup
- [ ] Avoid blind redownload if file already matches expected size and checksum/integrity tests pass

## Shape-specific config

- [ ] Decide whether built-in chain name is enough or custom chain spec is required
- [ ] Decide explicit fork timestamps, especially `Jovian`
- [ ] Decide `op-node` sync mode to test first
- [ ] Decide port isolation plan
- [ ] Decide exact data and config directories

## Reporting discipline

- [ ] Report execution head in decimal
- [ ] Distinguish unsafe movement from real execution movement
- [ ] Avoid calling zero EL peers a failure by default
