# Failure Patterns

## Pattern A: fake movement

Symptoms:
- `unsafe_l2` rises
- execution `eth_blockNumber` does not

Interpretation:
- not healthy
- likely engine/forkchoice/canonical-head issue

## Pattern B: chain-spec drift

Symptoms:
- payload parsing trouble
- unsupported fork style errors
- behavior changes after explicit hardfork overrides

Interpretation:
- runtime chain understanding is stale or mismatched

## Pattern C: snapshot optimism

Symptoms:
- re-extracting same snapshot reproduces the same stuck execution head

Interpretation:
- stop assuming local corruption was the only issue

## Pattern D: wrong health metric obsession

Symptoms:
- lots of focus on `net_peerCount`
- not enough focus on execution head movement

Interpretation:
- wrong diagnosis frame for current Shape mainnet
