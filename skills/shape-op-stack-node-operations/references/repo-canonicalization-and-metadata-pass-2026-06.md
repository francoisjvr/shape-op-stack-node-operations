# Repo canonicalization and metadata pass (2026-06)

Use this note when the Shape ops skill and its companion GitHub repo have drifted in naming, document numbering, or entrypoint metadata.

## What mattered

- After promoting `shape-op-stack-node-operations` to the sole canonical skill/repo identity, a second cleanup pass still found worthwhile polish work.
- The repo had a decent body of docs, but the public entrypoints were weaker than they should have been:
  - README intro still sounded transitional instead of canonical
  - the practical-files section omitted useful operator assets already present in the repo
  - the GitHub repo description/homepage/topics were either weak or empty
  - doc numbering had a duplicate `16-*` pair, which is easy to miss during renames

## Good cleanup pattern

1. Tighten the README intro so it clearly states the canonical purpose of the repo.
2. Audit the README "start here" list for stale links and duplicate numbering.
3. Rename docs when numbering drift creates ambiguous entrypoints.
4. Update every in-repo reference after a doc rename.
5. Surface practical operator assets already in the repo, not just prose docs.
6. Update GitHub metadata to match the canonical positioning:
   - description
   - homepage
   - topics
7. Verify the homepage URL actually returns HTTP 200.
8. Re-run stale-name searches after the polish pass, not just after the initial rename.

## Concrete results from this pass

- Renamed `docs/16-shape-rpc-and-sepolia-provider-notes.md` to `docs/17-shape-rpc-and-sepolia-provider-notes.md` so doc numbering no longer had two `16-*` entries.
- README now points operators to:
  - `examples/.env.example`
  - `examples/docker-compose.recommended.yml`
  - `config/docker-compose.reference.yml`
  - `templates/experiment-log-template.md`
- README companion repo mention was upgraded from plain text to a real GitHub link.
- GitHub repo metadata was improved to a Reth-first operations framing with Shape- and OP-Stack-relevant topics.

## Durable lesson

After any canonical rename or consolidation, do a second pass that treats the repo like a public product surface, not just a pile of internal notes. The follow-up pass should check naming consistency, doc numbering, entrypoint links, practical-file discoverability, and GitHub metadata.