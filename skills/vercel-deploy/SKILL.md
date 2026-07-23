---
name: vercel-deploy
description: Query Vercel deployment status, diagnose failures, and check env vars. Use when the user mentions "deployment", "deploy", "vercel", "server error", "merge", or environment names like "staging", "production", "dev". Use when another skill needs Vercel deployment information.
---

# Vercel Deploy Skill

## Activation

Activates on: `deployment`, `deploy`, `vercel`, `server error`, `staging`, `production`, `dev`, `commissioning`, `merge`.

## Merge Guardrail

When the user says "merge to main": confirm explicitly — merging to main triggers a production deployment.

When the user says "merge" without a target branch: ask which branch. Never assume `main`.

## Configuration File

Every project using this skill MUST have a `.vercel-environments.json` in its root:

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
    },
    "dev": {
      "branch": "dev",
      "domain": "dev.example.com",
      "vercelTarget": "preview"
    }
  }
}
```

- **branch**: Git branch that triggers this deployment
- **domain**: Custom domain, or `null` for auto-gen Vercel URLs
- **vercelTarget**: `"production"` or `"preview"` — passed to `vercel list --environment=`

## Init Flow — First Encounter

If `.vercel-environments.json` doesn't exist in the project root:

1. **Run discovery commands** in parallel:
   ```
   vercel project ls
   vercel env ls
   vercel domains
   git branch -a
   ```
2. **Infer the config** by matching env var scopes to branches and domains
3. **Show the inferred config** and ask: "Does this look right?"
4. **Write `.vercel-environments.json`** after confirmation
5. **Proceed** with the original query

On subsequent sessions, read the file — don't re-discover.

## Intent: Check Status

**Trigger**: "check staging", "deployment status", "is dev ready?"

If the user doesn't specify an environment, present the choices from `.vercel-environments.json`:
> Which environment? production · staging · dev · commissioning

When the environment is known:

1. Look up the environment in `.vercel-environments.json`
2. Run:
   ```
   vercel list --environment=<vercelTarget> --format=json --limit=5
   ```
3. If `branch` is `"*"` (catch-all): take the first entry. Otherwise filter by `meta.githubCommitRef === "<branch>"`
4. Take the first (latest) entry

**Report** — exactly 3 lines:

```
<env>  <emoji> <state>  (<relative time>)
<url>
Deployed from branch <branch> · <commit sha short> · <build duration>
```

Emojis: ✅ Ready · 🔴 Error · 🏗️ Building · ⏳ Queued

If the domain is set in the config, append it: `<url> (→ <domain>)`

**Example**:
```
staging  ✅ Ready  (2m ago)
https://aquabio-website-git-staging-aquabio-ltd.vercel.app
Deployed from branch staging · abc1234 · 42s build
```

## Intent: Diagnose Failure

**Trigger**: "why did staging fail?", "what's wrong with dev?", "server error"

If triggered by "server error" with no environment, ask which environment.

1. Run the **Check Status** flow to get the latest deployment URL and state
2. If state is `ERROR`, run:
   ```
   vercel inspect <url> --logs
   ```
3. If state is `READY` but the user reports errors: the deployment succeeded — the error is runtime, not build
4. After build logs, check env vars:
   ```
   vercel env ls <vercelTarget>
   ```
   Compare against shared env vars (Production, Preview, Development) and environment-specific overrides. Flag any missing compared to other environments

**Report**: build logs first, then env var summary. Stop after build logs + env vars — only go further if the user explicitly asks.

## Intent: Check Env Vars

**Trigger**: "what env vars are set on staging?", "diff env vars"

1. Look up the environment
2. Run:
   ```
   vercel env ls <vercelTarget>
   ```
3. Report in a compact table:
   ```
   Key          Staging    Production    Dev
   DATABASE_URL    ✅          ✅         ✅
   NOTIFY_API_KEY  ✅          ✅         ❌
   ```
   Show only the comparison the user asked for — don't dump all vars unless asked.

## Intent: Trigger Deploy

**Trigger**: "deploy staging", "push to production"

1. Confirm the target environment and branch
2. Ensure the correct branch is checked out, or confirm the user intends to deploy the current branch
3. Run:
   ```
   vercel deploy --target=<vercelTarget>
   ```
   or for production:
   ```
   vercel deploy --prod
   ```
4. Report the resulting URL

Confirm staging/dev/commissioning deploys once. Confirm production deploys explicitly — never deploy to production without it.
