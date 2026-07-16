# Vercel Marketplace

Working with Turso Cloud databases provisioned through the [Vercel Marketplace](https://vercel.com/marketplace/tursocloud) (integration slug: `tursocloud`). This file contains exact commands, safety rules, and the mental model an agent needs to operate autonomously.

If the user has **already provisioned** a Turso database and connected it to their project, go straight to [Using an already-provisioned database](#using-an-already-provisioned-database).

## Security: never read secrets

> **Warning:** **NEVER read `.env` files or any file that may contain Turso credentials.** This includes direct reads (`cat .env.local`), wildcards (`cat *`), and any command that could print directory contents where a `.env*` file exists. Use `vercel env ls` to see variable **names** (values are redacted) — that is all an agent needs; the SDK reads the values from `process.env` at runtime.

## Using an already-provisioned database

Every Vercel project connected to a Turso resource has two environment variables injected by the integration:

- `TURSO_DATABASE_URL` — the database connection URL
- `TURSO_AUTH_TOKEN` — the token used to authenticate requests

A `--prefix` chosen at connect time may have altered the names (e.g. `ANALYTICS_TURSO_DATABASE_URL`) — **always verify with `vercel env ls`** before writing connection code.

```bash
# 1. Link the local checkout to the Vercel project (if not already linked)
vercel link --yes
# 2. Verify the variable names (values stay redacted)
vercel env ls
# 3. Make the variables available for local development — do NOT read the file
vercel env pull .env.local
```

Then connect with the recommended SDK — `@tursodatabase/serverless` (fetch-only, works in Vercel serverless and edge functions). See [turso-cloud-js](../turso-cloud-js/overview.md) for install, query, batch, and transaction examples.

### In-Function SQLite — `@tursodatabase/vercel-experimental` (BETA)

Vercel-specific alternative for read-heavy workloads: database pages are replicated into the serverless function, so reads execute locally with zero network round-trips while writes go directly to the remote database. Databases are created on demand (`createDb("user-" + userId)`), which makes database-per-tenant patterns trivial. Configured with `TURSO_API_TOKEN`, `TURSO_ORG`, and `TURSO_GROUP` (a Turso platform API token — **not** the marketplace-injected variable pair). See `https://docs.turso.tech/integrations/vercel.md` for setup and the full comparison.

## Inspecting state (safe, autonomous)

These commands are read-only and can be run freely:

```bash
# Turso resources for the linked project (--all for the whole team)
vercel integration list --integration tursocloud --format=json

# Environment variable names for the linked project (values redacted)
vercel env ls

# Open the Turso dashboard via SSO (opens a browser)
vercel integration open tursocloud
```

> **Note:** Available `vercel integration` subcommands vary between CLI versions — run `vercel integration --help` to see what the installed version supports before relying on a subcommand.

## Provisioning a new database

Provisioning is additive and non-destructive, but whether an agent may do it autonomously depends on the installation's billing plan:

- On **paid plans**, Turso does not limit the number of databases — an agent may provision without confirmation.
- On the free **`starter` plan**, the installation is limited to **100 databases** — get the user's approval before provisioning so an autonomous agent doesn't silently consume the quota. If the plan is unknown, assume `starter` and ask.

```bash
vercel integration add tursocloud --name my-db --metadata region=iad1 --plan starter
```

- `region` is the only metadata key. It is **required at creation and cannot be changed afterwards**.
- After provisioning, the CLI connects the resource to the linked project and runs `vercel env pull` automatically (`--no-connect` / `--no-env-pull` to skip).
- When connecting a second Turso database to the same project, avoid variable-name collisions with `--prefix` (prepended as-is, e.g. `--prefix ANALYTICS_` → `ANALYTICS_TURSO_DATABASE_URL`).
- Environment variable changes only apply to **new** deployments — redeploy the project afterwards.
- If the integration is not installed yet, first-time installation requires accepting marketplace terms — **never accept terms on the user's behalf**; direct the user to complete that step in the Vercel dashboard.

### Regions

The `region` metadata key takes a Vercel region code, mapped to a Turso location:

| Vercel region code | Turso location | Region |
| --- | --- | --- |
| `iad1` | `aws-us-east-1` | US East (Virginia) |
| `cle1` | `aws-us-east-2` | US East 2 (Ohio) |
| `pdx1` | `aws-us-west-2` | US West (Oregon) |
| `dub1` | `aws-eu-west-1` | EU West (Ireland) |
| `bom1` | `aws-ap-south-1` | AP South (Mumbai) |
| `hnd1` | `aws-ap-northeast-1` | AP NorthEast (Tokyo) |

Pick the region where the project's **Vercel Functions** run — the database's client is the deployed function, not the developer's machine, so never choose a region based on the local environment. To determine the function region:

1. Check `vercel.json` for a `regions` field (e.g. `"regions": ["dub1"]`) — it overrides the project setting.
2. Otherwise use the project's **Function Region** setting (Vercel dashboard → Project Settings → Functions). If it was never changed, Vercel's default is `iad1`.
3. If the function region is not in the table above, pick the geographically closest supported region (e.g. `fra1` → `dub1`); if functions run in multiple regions, pick the one closest to where most writes originate.

## Dangerous operations — require explicit user consent

> **Warning:** **ANY DESTRUCTIVE ACTION MUST BE DONE THROUGH EXPLICIT USER CONSENT.** An agent must NEVER autonomously delete databases, disconnect resources from projects, uninstall the integration, change billing plans, or rotate credentials. All of these can cause data loss, service disruption, or billing changes.

- **Deleting a database resource** permanently destroys its data. **Disconnecting** a resource removes its environment variables and breaks deployments that use it. Both are done from the Vercel dashboard (or newer CLI versions' `vercel integration resource` subcommands).
- **Billing plan is installation-scoped** — the plan (`starter` free tier; paid tiers with `_overages` variants for pay-as-you-go beyond quotas) applies to ALL Turso databases under the installation, and billing is unified through Vercel. Always present pricing and get confirmation before any plan change.
- **Uninstalling the integration** (`vercel integration remove tursocloud`) requires all resources to be removed first.
- **Rotating credentials** (from the resource's settings in the Vercel dashboard) hard-invalidates previously issued tokens for that database — every consumer of the old token loses access once it expires. After rotation, redeploy connected projects and re-run `vercel env pull`.

## Docs

| Topic | URL |
|-------|-----|
| Turso + Vercel integration (SQL over HTTP, In-Function SQLite) | `https://docs.turso.tech/integrations/vercel.md` |
| Vercel CLI `integration` command | `https://vercel.com/docs/cli/integration` |
