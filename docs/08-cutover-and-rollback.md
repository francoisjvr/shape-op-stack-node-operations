# Cutover and Rollback

## Cutover should require proof

Do not cut over because Reth merely starts.

Require evidence:
- execution head advances in decimal
- lag converges acceptably
- repeated checks stay healthy
- restart behavior is sane
- logs are boring in the good way

## Rollback should be easy

Until cutover criteria are met:
- keep geth data intact
- keep geth docs intact
- keep the old recovery branch understandable

## If Reth stalls badly

Do not burn the fallback in frustration.

Rollback means:
- stop or isolate Reth attempt
- keep documentation of exact failure mode
- preserve artifacts needed for next iteration
