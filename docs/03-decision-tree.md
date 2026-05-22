# Decision Tree

## Is the task rescue or migration?

### If the current priority is keeping service up
Use the geth recovery repo first.

### If the current priority is preparing the next client
Continue here.

## Does Reth have a valid data source?

### If no
Stop and solve data source first.

### If yes
Proceed to isolated bring-up.

## Does execution `eth_blockNumber` move?

### If yes
Keep sampling and compare against public Shape RPC.

### If no
Go deeper.

## Does only `unsafe_l2` move?

### If yes
Interpretation:
- not healthy
- likely engine/forkchoice/canonical-head or chain-spec issue

### If no movement at all
Interpretation:
- more fundamental startup/config issue

## Are logs pointing at fork support / payload parsing?

### If yes
Think:
- chain-spec
- hardfork timestamps
- Reth/Node version mismatch
- `Jovian` handling

## Is someone tempted to throw away geth fallback?

### If yes
Refuse until Reth proves healthy.
