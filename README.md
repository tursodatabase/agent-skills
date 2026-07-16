# Turso agent skills

Agent skills that help AI coding agents work with Turso.

## Skills

### [turso-db](skills/turso-db)

Work with Turso, an in-process SQLite-compatible database written in Rust.

- **Vector search** -- vector32/64/sparse types, cosine/L2/Jaccard distance functions
- **Full-text search** -- Tantivy-powered FTS with BM25 scoring, tokenizers, highlighting
- **CDC** -- Change Data Capture for audit logs and change feeds
- **MVCC** -- Concurrent transactions with snapshot isolation
- **Encryption** -- Page-level encryption at rest (AES-GCM, AEGIS)
- **Sync** -- Push/pull replication, offline-first, WAL frame streaming
- **SDK references** -- JavaScript, Serverless, WASM, React Native, Rust, Python, Go

### [turso-cloud](skills/turso-cloud)

Work with Turso Cloud, the fully managed SQLite-compatible database platform.

- **SDK references** -- JavaScript/TypeScript (`@tursodatabase/serverless`, sync family), Python, Go, Rust
- **Auth** -- platform tokens, scoping, fine-grained per-table permissions, external auth providers via JWKS (Clerk, Auth0)
- **Integrations** -- databases provisioned through the [Vercel Marketplace](https://vercel.com/marketplace/tursocloud): provisioning, regions, env vars, safety rules

## Install

```bash
npx skills add tursodatabase/agent-skills
```

Or manually:

```bash
cp -r skills/turso-db ~/.claude/skills/turso-db
cp -r skills/turso-cloud ~/.claude/skills/turso-cloud
```

## Usage

Once installed, the skills activate automatically when you work with Turso databases. Use `/turso-db` or `/turso-cloud` to explicitly invoke them.
