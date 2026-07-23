# Add envOverride field to .vercel-environments.json

Agents can't understand the `.env` → `.env.local` override chain, and `vercel env ls` can't retrieve secret values from Vercel. We decided to add an optional `envOverride` field that points to a single env file whose keys overlay on top of `.env` (the implicit baseline). The config stays safe to commit — only file paths, no secrets.

## Considered Options

- **Store DATABASE_URL directly in the config.** Puts credentials in a committed file. Rejected.
- **Array of env files loaded in order.** Overkill — the pattern is always two layers (`.env` + one override). Rejected in favour of simplicity.
- **`dbKeys` subset extraction.** The user didn't want to maintain a separate list of which keys are database-related. Rejected.
