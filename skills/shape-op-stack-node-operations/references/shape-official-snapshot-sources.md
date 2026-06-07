# Shape official snapshot sources

Use this when a runbook, repo, or answer needs to tell an operator where to get the current Shape snapshot.

## Stable reference pattern

Prefer linking these stable entry points instead of hardcoding a direct archive URL:
- Shape docs: https://docs.shape.network/technical-details/run-a-node
- Shape docs source: https://github.com/shape-network/docs/blob/main/content/technical-details/run-a-node.mdx
- Current provider landing page used by the docs: https://www.alchemy.com/snapshots/shape

## Why

Direct snapshot archive URLs can rotate.

The safer operator pattern is:
1. link the official docs page as the canonical entry point
2. link the docs source if you want a durable place to verify the current recommendation
3. link the provider landing page the docs currently point to

## Repo-writing guidance

If you add a snapshot section to docs or a runbook:
- avoid pasting a one-off direct `.tar.lz4` URL unless you revalidated it very recently
- say explicitly that the direct file URL may rotate
- point readers to the stable docs/source/provider trio above
