# Shape mainnet fee floor and RPC notes (2026-06)

## Why this matters
Recent Shape troubleshooting showed two easy-to-miss realities:

1. Public Shape RPC may reject or throttle some requests while the private RPC is healthy.
2. Post-Isthmus/Jovian Shape now behaves like it has a real minimum L2 gas price because a minimum base fee is active.

## RPC findings
- Private RPC used successfully for reads and writes: `http://178.18.241.113:18545`
- `eth_chainId` returned `360`
- Public `https://mainnet.shape.network/` returned `HTTP 403` during probing in this session

Operational takeaway:
- For troubleshooting fees, estimating costs, or sending transactions, prefer the private RPC.
- If public and private RPC disagree, trust the private RPC that is actually serving current chain data and accepting traffic.

## Live fee findings from private RPC
Observed on Shape mainnet during this session:
- `baseFeePerGas = 10,000,000 wei = 0.01 gwei`
- `eth_gasPrice = 11,000,000 wei = 0.011 gwei`
- `eth_maxPriorityFeePerGas = 1,000,000 wei = 0.001 gwei`

Block `extraData` decoded to:
- version `1`
- denominator `250`
- elasticity `2`
- `minBaseFee = 10,000,000 wei = 0.01 gwei`

Interpretation:
- Shape has an active minimum base fee floor of about `0.01 gwei`.
- Transactions priced materially below that are now underpriced even on a quiet network.
- A practical send target during this session was about `0.011 gwei` total with `0.001 gwei` priority fee.

## Operator fee check
Live predeploy reads showed:
- `operatorFeeScalar = 0`
- `operatorFeeConstant = 0`

Operational takeaway:
- The observed increase in minimum acceptable gas price was explained by the minimum base fee floor, not by a newly nonzero operator fee surcharge.

## Cost interpretation reminder
This fee-floor change explains why txs now need a real minimum gas price.
It does **not** explain absurdly large uploader estimates by itself. If an estimate jumps to something like `0.5 ETH`, investigate:
- token/file selection scope
- total planned gas
- payload size / calldata size
- report math

Do not blame batch vs incremental mode alone unless the gas-plan numbers support it.
