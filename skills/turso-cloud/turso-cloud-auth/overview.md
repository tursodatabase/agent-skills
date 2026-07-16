# Authentication & Authorization

Turso Cloud uses scoped, JWT-based tokens to control access to databases. Every token can be restricted by database, permission level, and expiration.

## Database URL

If the database was provisioned through an integration or marketplace, its URL (and auth token) is usually already injected into the project's environment — prefer those variables over constructing anything by hand. Otherwise, find the database URL with the Turso CLI (`turso db show <database-name> --url`) or the Platform API. It looks like:

```bash
# libSQL protocol (SDK default)
libsql://[DB-NAME]-[ORG-NAME].turso.io

# HTTPS
https://[DB-NAME]-[ORG-NAME].turso.io
```

WebSockets generally perform better when keeping a socket open for multiple queries; HTTP is typically more efficient for single queries — benchmark both when it matters.

## Auth tokens

SDKs require an auth token unless working against a local database. Tokens are passed as `authToken` when creating a client:

```javascript
import { connect } from "@tursodatabase/serverless";

const conn = connect({
  url: "<your-database-url>",
  authToken: "<your-token>",
});
```

## Scoping levels

Tokens are scoped at multiple levels, from broad to narrow, and the levels can be combined:

| Level | Scope | How to create |
| --- | --- | --- |
| **Group** | Access all databases in a group | `turso group tokens create <group>` |
| **Database** | Access a single database | `turso db tokens create <database>` |
| **Read-only** | Queries only, no writes | Add `--read-only` flag |
| **Table + Action** | Specific tables and operations | Add `-p <table>:<actions>` flag |
| **Time-limited** | Auto-expires after a duration | Add `--expiration 7d` flag |

```bash
# Read-only token scoped to a single database, expiring in 7 days
turso db tokens create mydb --read-only --expiration 7d

# Token that only allows reading all tables and inserting into `comments`
turso db tokens create mydb \
  -p all:data_read \
  -p comments:data_add
```

## Platform tokens (CLI / API)

```bash
# Token for a single database
turso db tokens create <database-name>

# Token for all databases in a group
turso group tokens create <group-name>
```

| Flag | Description |
| --- | --- |
| `-r`, `--read-only` | Create a read-only token (queries only, no writes) |
| `-p`, `--permissions` | Set fine-grained permissions per table and action |
| `-e`, `--expiration` | Set token expiration (`never`, `7d`, `30d`, etc.) |

Tokens can also be created via the Platform API (`https://docs.turso.tech/api-reference/databases/create-token.md`).

## Fine-grained permissions

Permissions follow the format `<table-name|all>:<action1>,<action2>` and work with both CLI-generated and JWKS-issued tokens:

```bash
# Read-only access to all tables
turso db tokens create mydb -p all:data_read

# Specific actions on a specific table
turso db tokens create mydb -p users:data_read,data_update

# Multiple permission rules
turso db tokens create mydb \
  -p all:data_read \
  -p comments:data_add,data_update \
  -p posts:data_add,data_update,data_delete
```

Available actions:

| Action | Description |
| --- | --- |
| `data_read` | Read data from tables |
| `data_add` | Insert new rows |
| `data_update` | Update existing rows |
| `data_delete` | Delete rows |
| `schema_add` | Create new tables |
| `schema_update` | Modify table schemas |
| `schema_delete` | Drop tables |

> **Note:** `data_read` is allowed on SQLite system tables (e.g., `sqlite_master`, `sqlite_schema`) by default, so users can query database metadata.

Role-based access example — grant different permissions per user role (with JWKS, permissions are embedded in the JWT based on user metadata from the auth provider):

| Role | Permissions |
| --- | --- |
| Admin | `all:data_read,data_add,data_update,data_delete,schema_add,schema_update,schema_delete` |
| Moderator | `all:data_read,data_update` |
| User | `all:data_read` |

## External auth providers (JWKS)

Instead of managing tokens manually, let an authentication provider issue JWT tokens using JWKS. During the Turso beta, only Clerk and Auth0 are supported as OIDC providers.

1. **Generate a JWT template** and copy it into the auth provider's JWT configuration:

   ```bash
   # Full access to a database
   turso org jwks template --database <database-name> --scope full-access

   # Read-only access to a group
   turso org jwks template --group <group-name> --scope read-only

   # Fine-grained permissions
   turso org jwks template \
     --database <database-name> \
     --permissions all:data_read \
     --permissions comments:data_add
   ```

2. **Register the provider's JWKS endpoint** with the Turso organization (also possible from the Turso dashboard under organization settings):

   ```bash
   turso org jwks save <name> <url>
   # e.g. with Clerk:
   turso org jwks save clerk https://your-app.clerk.accounts.dev/.well-known/jwks.json
   ```

3. **Use the provider-issued JWT** as the `authToken`:

   ```javascript
   import { connect } from "@tursodatabase/serverless";

   const authToken = await getAuthToken(); // e.g., from Clerk, Auth0

   const conn = connect({
     url: "<your-database-url>",
     authToken,
   });
   ```

> **Warning:** Without a JWT template with specific permissions, generated tokens have access to **all databases in all groups** by default.

Manage registered endpoints:

```bash
turso org jwks list
turso org jwks remove <name>
```

## Invalidating tokens

Invalidating rotates the signing keys — **all** previously issued tokens for the target stop working, so every consumer must receive a new token:

```bash
# Invalidate all database-scoped tokens for a database
turso db tokens invalidate <database-name>

# Invalidate all group-scoped tokens for a group
turso group tokens invalidate <group-name>
```

Group tokens and database tokens are invalidated **independently**: invalidating a group's tokens does not invalidate database-scoped tokens for databases in that group, and invalidating a database's tokens does not invalidate group tokens (which keep working against that database).

## Docs

| Topic | URL |
|-------|-----|
| Authentication | `https://docs.turso.tech/sdk/authentication.md` |
| Authorization overview | `https://docs.turso.tech/sdk/authorization.md` |
| Platform tokens | `https://docs.turso.tech/sdk/authorization/tokens.md` |
| Fine-grained permissions | `https://docs.turso.tech/sdk/authorization/fine-grained-permissions.md` |
| External auth providers (JWKS) | `https://docs.turso.tech/sdk/authorization/jwks.md` |
