---
name: vercel-deploy
description: Query Vercel deployment status, diagnose failures, and check env vars. Activates on "deployment", "deploy", "vercel", "server error". Reads .vercel-environments.json from the project root for branch→environment mapping.
always-load: true
---

# Vercel Deploy Skill

This skill eliminates token waste and confusion around Vercel deployments. Instead of re-discovering project topology every session, it reads a single canonical config file (`.vercel-environments.json`) and runs precise `vercel` CLI commands.

## Activation

This skill is always loaded. It activates on any user message containing:
- `deployment`, `deploy`, `vercel`
- `server error` (triggers diagnose flow)
- Direct environment names: `staging`, `production`, `dev`, `commissioning`

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
- **domain**: Custom domain if one exists, `null` for auto-gen Vercel URLs
- **vercelTarget**: `"production"` or `"preview"` — passed to `vercel list --environment=`

## Init Flow — First Encounter

If `.vercel-environments.json` doesn't exist in the project root:

1. **Run discovery commands** (all at once, in parallel where possible):
   ```
   vercel project ls
   vercel env ls
   vercel domains
   git branch -a
   ```
2. **Infer the config** by matching env var scopes to branches, and domains where available
3. **Show the inferred config** to the user and ask: "Does this look right?"
4. **Write `.vercel-environments.json`** after confirmation
5. **Proceed** with the original query

Do NOT re-discover on subsequent sessions. Read the file.

## Intent: Check Status

**Trigger**: "check staging", "deployment status", "is dev ready?", etc.

**If the user doesn't specify an environment**, present the choices from `.vercel-environments.json`:
> Which environment? production · staging · dev · commissioning

**When the environment is known**:

1. Look up the environment in `.vercel-environments.json`
2. Run:
   ```
   vercel list --environment=<vercelTarget> --format=json --limit=5
   ```
3. Filter results by `meta.githubCommitRef === "<branch>"`
4. Pick the first (latest) entry

**Report format** — exactly 3 lines, no filler:

```
<env>  <emoji> <state>  (<relative time>)
<url>
Deployed from branch <branch> · <commit sha short> · <build duration>
```

Where `<emoji>` is:
- ✅ Ready / READY
- 🔴 Error / ERROR
- 🏗️ Building / BUILDING
- ⏳ Queued / QUEUED

If the domain is set in the config, append it: `<url> (→ <domain>)`

**Example output**:
```
staging  ✅ Ready  (2m ago)
https://aquabio-website-git-staging-aquabio-ltd.vercel.app
Deployed from branch staging · abc1234 · 42s build
```

## Intent: Diagnose Failure

**Trigger**: "why did staging fail?", "what's wrong with dev?", "server error"

**If triggered by "server error"** with no environment specified, ask which environment.

1. First run the **Check Status** flow above to get the latest deployment URL and state
2. If state is `ERROR`, run:
   ```
   vercel inspect <url> --logs
   ```
3. If state is `READY` but user reports errors, explain the deployment itself succeeded — the error is runtime, not build
4. After presenting build logs, run an env var check:
   ```
   vercel env ls <vercelTarget>
   ```
   Compare against the shared env vars (Production, Preview, Development) and environment-specific overrides. Flag any that are missing compared to other environments

**Report format**: build logs first, then env var summary. Do NOT go on a debugging expedition beyond this unless the user explicitly asks.

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
   Show at most the comparison the user asked for — don't dump all 40 vars unless asked.

## Intent: Trigger Deploy (rare)

**Trigger**: "deploy staging", "push to production"

1. Confirm the target environment and branch with the user
2. Ensure the correct branch is checked out or that the user intends to deploy the current branch
3. Run:
   ```
   vercel deploy --target=<vercelTarget>
   ```
   or for production:
   ```
   vercel deploy --prod
   ```
4. Report the resulting URL

**Never deploy production without explicit confirmation.** For staging/dev/commissioning, confirm once and proceed.

## Principles

- **One command first.** Don't run discovery commands when the config file already has the answer
- **Filter in code, not with flags.** The CLI doesn't have a `--branch` filter, so use `--format=json` and filter by `meta.githubCommitRef`
- **Short output.** The user wants the headline, not the firehose
- **Don't guess the environment.** If ambiguous, ask
- **Don't go on debugging expeditions.** Diagnose means build logs + env vars. Stop there
