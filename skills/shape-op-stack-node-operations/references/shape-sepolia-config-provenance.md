# Shape Sepolia config provenance

Use this note when the live Shape Sepolia lane appears to use fork timings or rollup fields that do not match public docs or the public Superchain Registry.

## Verified provenance chain from the session

Observed on the VPS:

- `/root/.shape-sepolia-op-reth-config/rollup.snapshot.json`
- `/root/.shape-sepolia-op-reth-config/genesis-l2.snapshot.json`
- `/root/.shape-sepolia-op-reth-config/reth.snapshot.toml`

These were verified byte-identical to the extracted runtime datadir copies:

- `/root/shape-sepolia-op-reth-data/rollup.json`
- `/root/shape-sepolia-op-reth-data/genesis-l2.json`
- `/root/shape-sepolia-op-reth-data/reth.toml`

Practical takeaway:
- the live lane was using snapshot-bundled config copied back out into the explicit config directory
- for provenance questions, treat the snapshot contents as the immediate source of truth before comparing against public docs

## Public Shape docs artifacts at the time

Shape docs `run-a-node` page pointed to:

- Shape Sepolia rollup: `https://arweave.net/gSb3hOzLaBIBy-AyWA2O2_03sfXJ-IwPqmV6Mhstmj8`
- Shape Sepolia genesis: `https://arweave.net/nw6opum2ALT9T39TSAAK_Wq0Z75aZJWd6J2RmiWHm_s`

Observed doc-artifact mismatch versus the live rollup:

Public docs rollup:
- includes `holocene_time = 1739880000`
- omits `chain_op_config`
- omits `isthmus_time`
- omits `jovian_time`

Live snapshot-backed rollup:
- includes `holocene_time = 1739880000`
- includes `isthmus_time = 1763650800`
- includes `jovian_time = 1777552200`
- includes `chain_op_config`

Observed public genesis mismatch versus the live genesis config section:
- live genesis config included `isthmusTime`
- live genesis config included `jovianTime`
- live genesis config included `pragueTime`
- current public Arweave genesis asset did not include those keys

## Public Superchain Registry comparison at the time

Public files checked:

- `superchain/configs/sepolia/shape.toml`
- `superchain/configs/sepolia/superchain.toml`
- `docs/hardfork-activation-inheritance.md`

Relevant findings:
- `shape.toml` only carried explicit Shape Sepolia hardfork values through `granite_time`
- `shape.toml` did not expose `superchain_time`
- registry inheritance spec says superchain-wide hardfork inheritance requires `superchain_time`
- Sepolia-wide defaults in `superchain.toml` were:
  - `holocene_time = 1732633200`
  - `isthmus_time = 1744905600`
  - `jovian_time = 1763568001`

Practical takeaway:
- the public registry did not validate the later live Shape Sepolia schedule in this session
- if a live lane shows `jovian_time = 1777552200`, do not claim that public upstream confirms it unless the registry or Shape docs are updated to match

## Reporting pattern

When summarizing this class of mismatch:
1. say whether the live files are identical between config dir and datadir
2. name the public docs artifacts checked
3. name the registry files checked
4. separate `live source of truth` from `public authoritative source`
5. avoid overclaiming that the public upstream confirms the live values when it does not
