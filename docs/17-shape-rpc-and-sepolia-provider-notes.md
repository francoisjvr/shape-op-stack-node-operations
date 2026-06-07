# Shape RPC and Sepolia Provider Notes

This repo is mainly about **Shape mainnet op-reth + op-node bring-up**, but the current Shape docs now make one operator lesson explicit enough that it is worth recording here too:

> when you need steady throughput or production-grade behavior, do not lean too hard on the public RPC.

## Official Shape docs worth remembering

### 1. Network Information
Shape docs page:
- `https://docs.shape.network/technical-details/network-information`

Current published Shape Sepolia details on that page include:
- network name: `Shape Sepolia`
- chain ID: `11011`
- public RPC endpoint: `https://sepolia.shape.network`
- block explorer: `https://sepolia.shapescan.xyz`

That same page also says:
- for production systems or high-throughput testing, use a node from **Alchemy**, one of Shape's node providers

### 2. Node Providers
Shape docs page:
- `https://docs.shape.network/tools/node-providers`

Current published guidance there includes:
- mainnet public RPC: `https://mainnet.shape.network`
- mainnet Alchemy pattern: `https://shape-mainnet.g.alchemy.com/v2/YOUR_KEY`
- Sepolia public RPC: `https://sepolia.shape.network`
- Sepolia Alchemy pattern: `https://shape-sepolia.g.alchemy.com/v2/YOUR_KEY`

The key operational wording from that page is basically:
- the public RPC is rate-limited
- it is not suitable for production
- use Alchemy / a node provider instead

## What that means for this repo

Even though this repo is a **mainnet Reth** operations repo, the same lesson matters any time we:
- compare local head vs a public Shape endpoint
- run repeated sync checks
- automate RPC polling
- use Sepolia for testing or smoke checks

## Blunt operator translation

Use the public endpoint for:
- quick one-off manual checks
- lightweight sanity checks
- confirming that a response exists at all

Prefer a provider endpoint for:
- repeated health sampling
- automation
- scripts that poll frequently
- higher-throughput testing
- anything where you do not want rate-limit noise muddying the conclusion

## Practical note from this repo's point of view

Bluntly:
- a paid API/provider endpoint seems to work better than the public endpoint

That does **not** mean the public endpoint is useless.
It means:
- public is fine for occasional checks
- paid/provider access is the safer baseline when you need steadier behavior

## Recommended wording to preserve in future docs

If we mention Shape RPC selection again, keep it simple:
- public RPC is okay for light manual checks
- for production, high-throughput, or repeated automated checks, prefer a node provider
- if behavior matters, paid/provider access tends to work better than public RPC
