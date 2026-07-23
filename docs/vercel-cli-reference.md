# Vercel CLI Reference

Maintainer notes on the Vercel CLI surface used by this skill.

- `vercel list` — all deployments; `--environment=preview|production` scopes
- `meta.githubCommitRef` — the branch name in JSON output
- `vercel inspect <url> --logs` — build logs
- `vercel env ls <target>` — env vars scoped to `production`, `preview`, or `development`
- Custom environments in the Vercel dashboard don't appear as distinct CLI targets — they show as Preview with a branch specifier
- Vercel has no built-in `.firebaserc` equivalent (see [ADR-0001](adr/0001-vercel-environments-config.md))
