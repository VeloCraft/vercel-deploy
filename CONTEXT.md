# Vercel Deploy Skill

Domain model for the vercel-deploy agent skill.

## Language

**Environment**:
A named deployment slot (e.g. "production", "staging", "dev"). Each environment maps to a Git branch, a Vercel target, and optionally a domain and env override.
_Avoid_: Setup, config, deployment target

**Vercel Target**:
The Vercel-level environment type — `production` or `preview`. Passed to `vercel list --environment=` and `vercel deploy --target=`. A project environment maps to one Vercel target.
_Avoid_: Vercel environment, scope

**Env Override**:
An optional single env file whose keys override `.env` for a given environment. `.env` is always the implicit baseline loaded first. If absent, only `.env` is used. Contains only a file path — no secret values.
_Avoid_: Env chain, env array, env file
