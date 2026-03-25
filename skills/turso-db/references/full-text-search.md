# Full-Text Search (FTS)

Turso provides FTS powered by the [Tantivy](https://github.com/quickwit-oss/tantivy) search engine. Requires the `fts` feature at compile time.

**Status:** Experimental — requires `--experimental-index-methods` flag when starting `tursodb`.

## Creating an FTS Index

```sql
CREATE INDEX idx_articles ON articles USING fts (title, body);
```

Index multiple columns in one FTS index. The index automatically tracks inserts, updates, and deletes.

## Tokenizer Configuration

Configure tokenization with the `WITH` clause:

```sql
CREATE INDEX idx_products ON products USING fts (name) WITH (tokenizer = 'ngram');
CREATE INDEX idx_tags ON articles USING fts (tag) WITH (tokenizer = 'raw');
```

### Available Tokenizers

| Tokenizer | Description | Use Case |
|-----------|-------------|----------|
| `default` | Lowercase, punctuation/whitespace split, drops tokens longer than 40 chars | General English text |
| `raw` | No tokenization — exact match only | IDs, UUIDs, tags |
| `simple` | Basic whitespace/punctuation split | Text without lowercasing |
| `whitespace` | Split on whitespace only | Space-separated tokens |
| `ngram` | 2-3 character n-grams | Autocomplete, substring matching |

### Tokenizer Examples

```
default:    "Hello World"  → ["hello", "world"]
raw:        "user-123"     → ["user-123"]
ngram:      "iPhone"       → ["iP", "iPh", "Ph", "Pho", "ho", "hon", "on", "one", "ne"]
```

## Field Weights

Configure relative importance for BM25 scoring:

```sql
-- Title matches are 2x more important than body
CREATE INDEX idx_articles ON articles USING fts (title, body)
WITH (weights = 'title=2.0,body=1.0');

-- Combined with tokenizer
CREATE INDEX idx_docs ON docs USING fts (name, description)
WITH (tokenizer = 'simple', weights = 'name=3.0,description=1.0');
```

Default weight is `1.0`. Weights must be positive numbers.

## Query Functions

### `fts_match(col1, col2, ..., 'query')`

Returns boolean — use in WHERE clauses to filter matching rows:

```sql
SELECT id, title FROM articles WHERE fts_match(title, body, 'database');
```

### `fts_score(col1, col2, ..., 'query')`

Returns BM25 relevance score for ranking:

```sql
SELECT fts_score(title, body, 'database') AS score, id, title
FROM articles
WHERE fts_match(title, body, 'database')
ORDER BY score DESC
LIMIT 10;
```

### `fts_highlight(col1, col2, ..., before_tag, after_tag, 'query')`

Returns text with matching terms wrapped in tags:

```sql
SELECT fts_highlight(body, '<mark>', '</mark>', 'database') AS highlighted
FROM articles
WHERE fts_match(title, body, 'database');
-- Returns: "Learn about <mark>database</mark> optimization"
```

Notes on `fts_highlight`:
- Supports multiple text columns (concatenated with spaces)
- Case-insensitive matching
- Returns original text if no matches found
- Returns NULL if query/before_tag/after_tag is NULL
- NULL text columns are skipped

## Query Syntax (Tantivy)

| Syntax | Example | Description |
|--------|---------|-------------|
| Single term | `database` | Match "database" |
| Multiple terms (OR) | `database sql` | Match "database" OR "sql" |
| AND | `database AND sql` | Match both terms |
| NOT | `database NOT nosql` | Exclude "nosql" |
| Phrase | `"full text search"` | Exact phrase match |
| Prefix | `data*` | Terms starting with "data" |
| Column filter | `title:database` | Match only in title field |
| Boosting | `title:database^2` | Boost title matches 2x |

## Complex Queries

FTS functions work alongside regular WHERE conditions:

```sql
SELECT id, title, fts_score(title, body, 'Rust') AS score
FROM articles
WHERE fts_match(title, body, 'Rust')
  AND category = 'tech'
  AND published = 1
ORDER BY score DESC;
```

## Index Maintenance

Merge Tantivy segments for better query performance:

```sql
-- Optimize a specific FTS index
OPTIMIZE INDEX idx_articles;

-- Optimize all FTS indexes
OPTIMIZE INDEX;
```

Run after bulk inserts or when performance degrades.

## Complete Example

```sql
-- Create table
CREATE TABLE documents (
    id INTEGER PRIMARY KEY,
    title TEXT,
    content TEXT,
    category TEXT
);

-- Create FTS index with weighted fields
CREATE INDEX fts_docs ON documents USING fts (title, content)
WITH (weights = 'title=2.0,content=1.0');

-- Insert data
INSERT INTO documents VALUES
    (1, 'Introduction to SQL', 'Learn SQL basics and queries', 'tutorial'),
    (2, 'Advanced SQL Techniques', 'Complex joins and optimization', 'tutorial'),
    (3, 'Database Design', 'Schema design best practices', 'architecture');

-- Search with scoring and highlighting
SELECT
    id,
    title,
    fts_score(title, content, 'SQL') AS score,
    fts_highlight(content, '<b>', '</b>', 'SQL') AS snippet
FROM documents
WHERE fts_match(title, content, 'SQL')
ORDER BY score DESC;
```

## Limitations

| Limitation | Description |
|------------|-------------|
| No read-your-writes in transactions | FTS changes visible only after COMMIT |
| No `snippet()` function | Use `fts_highlight()` instead |
| No automatic segment merging | Use `OPTIMIZE INDEX` for manual merging |
| Requires `fts` compile-time feature | Not available in all builds |
