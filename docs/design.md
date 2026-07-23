# Design: Vercel Deploy Skill

**Date**: 2026-07-20
**Session**: Grilling with domain-modeling

## Problem

The coding agent wastes tokens and time re-discovering Vercel deployment topology every session. It runs multiple `vercel` lookups, `curl` commands, and `dash` scripts — often getting confused about which branch maps to which environment. The user wants to say "check staging" and get an instant, correct answer.

## Root Causes

1. **No canonical config file.** AGENTS.md has a partial branch→environment table but it's prose, not machine-readable, and was missing `staging`
2. **No CLI recipe.** The agent knows what branches exist but not which `vercel` commands to run for each intent
3. **No caching.** The agent re-discovers the same static facts every session
4. **No `.firebaserc` equivalent.** Vercel has no standard file declaring project environments

## Solution

### `.vercel-environments.json`

A machine-readable, project-scoped config file (inspired by `.firebaserc`) that declares every environment:

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

- **branch**: The git branch that triggers deployment
- **domain**: Custom domain or `null` for auto-gen Vercel URLs
- **vercelTarget**: `"production"` or `"preview"` — maps to `--environment=` flag

### Global Skill

Always-loaded skill that activates on keywords: `deployment`, `deploy`, `vercel`, `server error`.

Three core intents:

| Intent | Trigger | CLI |
|---|---|---|
| Status | "check staging" | `vercel list --environment=X --format=json` → filter by `meta.githubCommitRef` |
| Diagnose | "why did staging fail?" | `vercel inspect <url> --logs` + `vercel env ls <target>` |
| Env vars | "what env vars on staging?" | `vercel env ls <target>` |
| Init | First encounter | Autodiscover → confirm → write config |

### Init Flow

On first encounter in a project (no `.vercel-environments.json`):

1. Run `vercel project ls`, `vercel env ls`, `vercel domains`, `git branch -a`
2. Infer config by matching env var scopes to branches
3. Show inferred config, ask "does this look right?"
4. Write file after confirmation

### Status Report Format

Three-line summary, no filler:

```
staging  ✅ Ready  (2m ago)
https://aquabio-website-git-staging-aquabio-ltd.vercel.app
Deployed from branch staging · abc1234 · 42s build
```

### Key Design Decisions

1. **Always-loaded, not slash-command.** The user wants it ready when they talk about deployments naturally, not something they have to remember to invoke
2. **One command first.** Read the config file, run one `vercel` command. Don't re-discover
3. **JSON filter, not CLI flag.** Vercel CLI has no `--branch` filter. Use `--format=json` and filter by `meta.githubCommitRef` in code
4. **Diagnose is bounded.** Build logs + env var diff only. No debugging expeditions unless explicitly asked
5. **Global, not project-specific.** Same skill works across all Vercel projects by reading each project's `.vercel-environments.json`

## Vercel CLI Facts (discovered during session)

- `vercel list` shows all deployments; `--environment=preview|production` filters
- `meta.githubCommitRef` in JSON output gives the branch name
- `vercel inspect <url> --logs` surfaces build logs
- `vercel env ls <target>` shows env vars scoped to that environment
- Custom environments in Vercel dashboard don't show in CLI's `env ls` targets — they appear as Preview with a branch specifier
- Vercel has no built-in `.firebaserc` equivalent

## Aquabio-Specific Topology

| Branch | Vercel Target | Domain |
|---|---|---|
| `main` | production | `www.aquabio.uk` |
| `dev` | preview | `dev.aquabio.uk` |
| `staging` | preview (custom env vars) | auto-gen only |
| `commissioning` | preview | `commissioning.aquabio.uk` |
| other | preview | auto-gen |

## Remaining Tensions

- **AGENTS.md is still the authority for non-deployment project rules.** The `.vercel-environments.json` only covers deployment topology. The skill bridges the two
- **`staging` has custom env vars** scoped to `staging` (not just `Preview`). The `vercel env ls` command needs to target the right scope
- **"Server error" ambiguity**: the skill activates but must ask which environment if not specified
- **Trigger deploy is rare** and intentionally manual — never automate production deploys without explicit confirmation
