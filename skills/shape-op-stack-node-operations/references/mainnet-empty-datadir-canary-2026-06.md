# Shape mainnet empty-datadir canary findings (June 2026)

Use this reference when a repaired or freshly restored mainnet snapshot still shows the classic bad pattern: `unsafe_l2` advances but canonical execution head stays pinned, `safe_l2` / `current_l1` stay zero, and a known-missing block never appears.

## Why this reference exists
A fresh June 3 snapshot plus a real offline `repair-trie` still left mainnet canonical head stuck at `29343274` while `unsafe_l2` advanced. To separate "bad snapshot" from "bad runtime/flags", an isolated empty-datadir / from-genesis mainnet canary was launched on separate ports.

## Isolation pitfall discovered first
Ports `28545/28546/28551/29545/29222/32303` were already occupied by an older long-lived **Sepolia** canary. Before launching any new mainnet canary, check live listeners and container ownership; do not assume those ports are free just because they are not part of the primary mainnet lane.

Known occupied pair during this session:
- Sepolia Reth canary held `28545`, `28546`, `28551`, `32303`
- Sepolia op-node canary held `29545`, `29222`

The isolated **mainnet** canary therefore used:
- Reth: `38545` HTTP, `38546` WS, `38551` authrpc, `33303` p2p
- op-node: `39545` RPC, `39222` p2p

## What the isolated mainnet canary proved
With an empty datadir and repo-aligned mainnet config:
- `op-reth` started cleanly from genesis (`head=0`)
- `op-node` started cleanly and tracked L1 (`current_l1` moved from `20369793` to `20369795`; `head_l1` advanced)
- But canonical L2 still stayed at genesis in the short observation window:
  - `head=0`
  - `unsafe_l2=0`
  - `safe_l2=0`
  - `finalized_l2=0`
  - execution stages stayed at `0`

## Key logs to recognize
From `op-node`:
- `requesting engine missing unsafe L2 block range ... start ...:0 end ...:29449296`
- `failed to insert unsafe payload`
- `temp: cannot prepare unsafe chain for new payload ... err: updated forkchoice, but node is syncing`

From `op-reth`:
- `Received new payload from consensus engine` for block `29449296`
- status still reported `latest_block=0`

## Operational meaning
If an empty-datadir canary reproduces "payload observed but canonical head does not advance yet," the problem is not cleanly isolated to the restored snapshot alone. At that point, the investigation has crossed from "snapshot corruption only" into a broader runtime / chain-behavior discriminator:
- either the lane needs longer observation from genesis,
- or the issue is in how payload insertion / canonical advancement is behaving under the current runtime assumptions.

## Longer-sample confirmation from the follow-up session
A later follow-up kept the canary and repaired primary lane under observation long enough to sharpen the classification:
- the repaired primary lane did show **partial** canonical movement (`29403655 -> 29404322`) after offline `repair-trie`, but then flatlined again
- `safe_l2` and `finalized_l2` only caught up to that same new head instead of continuing toward tip
- the formerly missing target block still stayed absent
- the isolated empty-datadir canary kept `head=0`, `safe_l2=0`, and `finalized_l2=0` while `current_l1` and `head_l1` continued to advance

Interpretation:
- offline `repair-trie` can yield a real-but-incomplete win; do not overcall a few hundred blocks of movement as recovery
- if the from-genesis canary also refuses to canonically advance while it clearly tracks L1, the failure is **not snapshot-only**
- classify that result as `broader canonicalization/runtime issue still present`, with the original snapshot/datadir no longer the sole suspect

## Use in future incidents
When a repaired snapshot still looks bad:
1. Launch an isolated mainnet canary on ports that are confirmed unused.
2. Keep the primary lane untouched.
3. Distinguish four outcomes:
   - **Canary head advances from 0** -> snapshot lane remains the prime suspect.
   - **Canary tracks L1 but head stays 0 while payloads arrive** -> not snapshot-only; escalate as a broader runtime/canonicalization issue.
   - **Primary lane moves only a few hundred blocks after repair and then stalls again** -> treat as partial recovery, not success.
   - **Canary cannot even initialize cleanly** -> fix isolation/config before drawing conclusions.
