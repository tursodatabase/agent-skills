# Vector Search

Turso supports vector search for semantic search, recommendation systems, and similarity matching.

## Vector Types

### Dense Vectors

Store a value for every dimension:

- **`vector32`** — 32-bit float, 4 bytes/dimension. Use for most ML embeddings (OpenAI, sentence transformers).
- **`vector64`** — 64-bit float, 8 bytes/dimension. Use when higher precision is needed.
- **`vector8`** — 8-bit integer, 1 byte/dimension. Use for quantized embeddings where memory/storage is critical.
- **`vector1bit`** — 1-bit binary, 1 bit/dimension. Use for binary quantization (e.g., Matryoshka embeddings).

### Sparse Vectors

Store only non-zero values with their indices:

- **`vector32_sparse`** — 32-bit float sparse. Use for TF-IDF, bag-of-words, high-dimensional sparse data.

## Creating Vectors

```sql
-- Dense 32-bit
SELECT vector32('[1.0, 2.0, 3.0]');

-- Dense 64-bit
SELECT vector64('[1.0, 2.0, 3.0]');

-- Sparse 32-bit (zeros are not stored)
SELECT vector32_sparse('[0.0, 1.5, 0.0, 2.3, 0.0]');

-- Extract vector as readable text
SELECT vector_extract(embedding) FROM documents;
```

## Distance Functions

**IMPORTANT: Lower distance = more similar. Always ORDER BY distance ASC.**

### `vector_distance_cos(v1, v2)` — Cosine Distance

Returns 0 (identical direction) to 2 (opposite direction). Computed as `1 - cosine_similarity`.

Best for: text embeddings, document similarity, cases where magnitude doesn't matter.

```sql
SELECT name, vector_distance_cos(embedding, vector32('[0.1, 0.5, 0.3]')) AS distance
FROM documents
ORDER BY distance
LIMIT 10;
```

### `vector_distance_l2(v1, v2)` — Euclidean (L2) Distance

Returns straight-line distance in n-dimensional space.

Best for: image embeddings, spatial data, unnormalized embeddings.

```sql
SELECT name, vector_distance_l2(embedding, vector32('[0.1, 0.5, 0.3]')) AS distance
FROM documents
ORDER BY distance
LIMIT 10;
```

### `vector_distance_jaccard(v1, v2)` — Weighted Jaccard Distance

Measures dissimilarity based on min/max ratio across dimensions. Different from ordinary (binary) Jaccard distance.

Best for: sparse vectors, set-like comparisons, TF-IDF representations.

```sql
SELECT name, vector_distance_jaccard(sparse_emb, vector32_sparse('[0.0, 1.0, 0.0, 2.0]')) AS distance
FROM documents
ORDER BY distance
LIMIT 10;
```

## Utility Functions

### `vector_concat(v1, v2)`

Concatenates two vectors. Result has dimensions = dim(v1) + dim(v2).

```sql
SELECT vector_concat(vector32('[1.0, 2.0]'), vector32('[3.0, 4.0]'));
-- Result: [1.0, 2.0, 3.0, 4.0]
```

### `vector_slice(vector, start, end)`

Extracts a slice from `start` to `end` (exclusive, 0-indexed).

```sql
SELECT vector_slice(vector32('[1.0, 2.0, 3.0, 4.0, 5.0]'), 1, 4);
-- Result: [2.0, 3.0, 4.0]
```

## Complete Example: Semantic Search

```sql
-- Create table with embedding column
CREATE TABLE documents (
    id INTEGER PRIMARY KEY,
    name TEXT,
    content TEXT,
    embedding BLOB
);

-- Insert documents with precomputed embeddings
INSERT INTO documents (name, content, embedding) VALUES
    ('Doc 1', 'Machine learning basics', vector32('[0.2, 0.5, 0.1, 0.8]')),
    ('Doc 2', 'Database fundamentals', vector32('[0.1, 0.3, 0.9, 0.2]')),
    ('Doc 3', 'Neural networks guide', vector32('[0.3, 0.6, 0.2, 0.7]'));

-- Find most similar documents to a query embedding
SELECT
    name,
    content,
    vector_distance_cos(embedding, vector32('[0.25, 0.55, 0.15, 0.75]')) AS distance
FROM documents
ORDER BY distance
LIMIT 5;
```

## Distance Function Comparison

| Function | Range | Best For |
|----------|-------|----------|
| `vector_distance_cos` | 0 to 2 | Text embeddings, normalized vectors |
| `vector_distance_l2` | 0 to infinity | Image embeddings, spatial data |
| `vector_distance_jaccard` | 0 to 1 | Sparse vectors, TF-IDF |
