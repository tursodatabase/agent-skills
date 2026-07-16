---
name: turso-cloud
description: >
  Turso Cloud — fully managed SQLite-compatible database platform, accessed over the network.
  Use when connecting an application to a Turso Cloud database, creating or scoping auth tokens
  (JWT, fine-grained permissions, JWKS/external auth providers), or provisioning and managing
  cloud databases. Covers per-language SDKs (JavaScript/TypeScript, Python, Go, Rust),
  authentication & authorization, and marketplace integrations (Vercel).
---

# Turso Cloud Skills

This skill combines documentation for working with Turso Cloud, the fully managed SQLite-compatible database platform. Pick the relevant sub-skill below.

## [turso-cloud-js](turso-cloud-js/overview.md)

Work with the `@tursodatabase/serverless` TypeScript/JavaScript SDK for remote Turso Cloud databases — queries, positional/named parameters, batches, and interactive transactions — plus the local-first sync family (`@tursodatabase/sync`, `-wasm`, `-react-native`). Use when connecting to Turso Cloud from Node.js, serverless/edge functions, browsers, or React Native.

## [turso-cloud-py](turso-cloud-py/overview.md)

Work with Turso Cloud from Python: `pyturso` for local-first databases that sync with the cloud, and `libsql` for remote-only access over the libsql protocol. Use when connecting Python applications to Turso Cloud.

## [turso-cloud-go](turso-cloud-go/overview.md)

Work with Turso Cloud from Go: `tursogo` for local-first databases that sync with the cloud, and `libsql-client-go` for remote-only access over the libsql protocol. Use when connecting Go applications to Turso Cloud.

## [turso-cloud-rust](turso-cloud-rust/overview.md)

Work with Turso Cloud from Rust: the `turso` crate (`sync` feature) for local-first databases that sync with the cloud, and the `libsql` crate for remote-only access over the libsql protocol. Use when connecting Rust applications to Turso Cloud.

## [turso-cloud-auth](turso-cloud-auth/overview.md)

Authenticate and authorize access to Turso Cloud databases: database URLs, platform tokens via the Turso CLI, scoping (group/database/read-only/time-limited), fine-grained per-table permissions, external auth providers via JWKS (Clerk, Auth0), and token invalidation. Use when issuing credentials, restricting what a client can do, or wiring an existing auth provider to Turso.

## [turso-cloud-vercel](turso-cloud-vercel/overview.md)

Use and manage Turso Cloud databases provisioned through the Vercel Marketplace (integration slug `tursocloud`): injected environment variables, inspecting and provisioning resources with the Vercel CLI, region selection, and safety rules for credentials and destructive operations. Use when a Vercel project has — or needs — a Turso database.

## Turso Cloud features

Turso Cloud databases support these features out of the box:

- **Vector search** — native similarity search for AI/RAG workflows, no extensions needed. Supports cosine, L2, and L1 distance functions with DiskANN indexing.
- **Full-text search** — keyword search with BM25 ranking via built-in FTS support.
- **AI & embeddings** — native vector types for storing and querying embeddings.
- **Branching** — create isolated copy-on-write database branches for testing/staging.
- **Point-in-time recovery** — restore databases to any previous point in time (retention depends on plan).
- **Embedded replicas** — local replicas that sync with Turso Cloud for fast reads and offline support.
- **SQLite extensions** — JSON, FTS5, R*Tree, SQLean, UUID, regexp.

## Further reading

**Turso online docs** — `https://docs.turso.tech` (Mintlify — append `.md` to any URL path for raw markdown, e.g. `https://docs.turso.tech/sdk/ts/quickstart.md`). Each sub-skill carries its own docs links; general references:

| Topic | URL |
|-------|-----|
| AI & embeddings | `https://docs.turso.tech/features/ai-and-embeddings.md` |
| Embedded replicas | `https://docs.turso.tech/features/embedded-replicas/introduction.md` |
| Cloud limitations | `https://docs.turso.tech/cloud/limitations.md` |
| SQLite extensions | `https://docs.turso.tech/features/sqlite-extensions.md` |
| Full docs index (for LLMs) | `https://docs.turso.tech/llms.txt` |
