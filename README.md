# vercel-deploy

A coding-agent skill for querying Vercel deployments — check status, diagnose failures, diff env vars, and trigger deploys. Driven by a per-project `.vercel-environments.json` config file so the agent never re-discovers topology.

## Quick start

1. Install the skill in your agent's skill directory
2. Add a `.vercel-environments.json` to any Vercel project:

```json
{
  "environments": {
    "production": {
      "branch": "main",
      "domain": "www.example.com",
      "vercelTarget": "production"
    },
    "staging": {
      "branch": "staging",
      "domain": null,
      "vercelTarget": "preview"
    }
  }
}
```

The skill auto-discovers and writes this file on first encounter if it doesn't exist.

## Intents

| Intent | Example trigger | What it does |
|---|---|---|
| **Check status** | "check staging" | `vercel list` → 3-line summary |
| **Diagnose failure** | "why did staging fail?" | Build logs + env var diff |
| **Check env vars** | "diff env vars" | Compact comparison table |
| **Trigger deploy** | "deploy staging" | Confirms, then `vercel deploy` |
| **Merge guardrail** | "merge to main" | Requires explicit confirmation |

## Docs

- [SKILL.md](skills/vercel-deploy/SKILL.md) — the skill body
- [ADR-0001](docs/adr/0001-vercel-environments-config.md) — why `.vercel-environments.json` exists
- [Vercel CLI reference](docs/vercel-cli-reference.md) — maintainer notes on the CLI surface
