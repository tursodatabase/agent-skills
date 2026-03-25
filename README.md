# turso-db skill

An agent skill that helps AI coding agents work with Turso, an in-process SQLite-compatible database written in Rust.

## What's Included

- **Vector search** -- vector32/64/sparse types, cosine/L2/Jaccard distance functions
- **Full-text search** -- Tantivy-powered FTS with BM25 scoring, tokenizers, highlighting
- **CDC** -- Change Data Capture for audit logs and change feeds
- **MVCC** -- Concurrent transactions with snapshot isolation
- **Encryption** -- Page-level encryption at rest (AES-GCM, AEGIS)
- **Sync** -- Push/pull replication, offline-first, WAL frame streaming
- **SDK references** -- JavaScript, Serverless, WASM, React Native, Rust, Python, Go

## Install

```bash
npx skills add sivukhin/turso-skill
```

Or manually:

```bash
cp -r skills/turso-db ~/.claude/skills/turso-db
```

## Usage

Once installed, the skill activates automatically when you work with Turso/libSQL databases. Use `/turso-db` to explicitly invoke it.
