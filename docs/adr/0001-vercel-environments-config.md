# .vercel-environments.json as canonical deployment config

Vercel has no built-in equivalent of `.firebaserc` — no standard file declaring which branches deploy to which environments. Without one, the agent re-discovers deployment topology every session by running multiple CLI lookups, often getting confused about branch→environment mappings.

We decided to define a single canonical config file, `.vercel-environments.json`, that declares every environment's branch, domain, and Vercel target. The skill reads this file instead of re-discovering.

## Considered Options

- **Embed in AGENTS.md as prose.** Fragile, not machine-readable, easy to miss when editing. Rejected.
- **Re-discover every session.** Wastes tokens and produces inconsistent results. Rejected.
- **Vercel API / dashboard scraping.** No stable CLI surface for branch→environment mappings. Rejected.
