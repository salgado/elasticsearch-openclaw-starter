# Classic Patterns — Updated for ES 9.x

## Mapping Design

### Field Types
- `text` — full-text search (tokenized, analyzed)
- `keyword` — exact match, aggregations, sorting
- `semantic_text` — AI semantic search via inference endpoint (ES 8.11+)
- `dense_vector` — manual vector storage for kNN
- `integer`, `float`, `double` — numeric
- `boolean`, `date`, `ip` — as named

### Multi-field pattern (text + keyword)
```json
"title": {
  "type": "text",
  "fields": {
    "raw": { "type": "keyword" }
  }
}
```
Search on `title`, sort/aggregate on `title.raw`.

### Strict mapping (recommended for production)
```json
PUT my-index
{
  "mappings": {
    "dynamic": "strict",
    "properties": { ... }
  }
}
```
Rejects unknown fields — prevents mapping explosions and typos.

### ⚠️ Cannot change field type after indexing
Must reindex to a new index with correct mapping. Use aliases for zero-downtime reindex.

## Query vs Filter Context

```json
{
  "query": {
    "bool": {
      "must": [
        { "match": { "title": "broccoli" } }      ← scores (relevance)
      ],
      "filter": [
        { "term": { "on_sale": true } },           ← cached, no score
        { "range": { "price": { "lte": 20 } } }   ← cached, no score
      ]
    }
  }
}
```

- `must` / `should` → scoring context, expensive
- `filter` / `must_not` → yes/no, **cached**, use for exact conditions

## Analyzers

```bash
# Test before indexing
POST _analyze
{
  "analyzer": "english",
  "text": "Fresh running vegetables"
}
```

| Analyzer | Behavior | Use for |
|---|---|---|
| `standard` | lowercase, removes punctuation | General text |
| `keyword` | exact string | Codes, emails, IDs |
| `english` | stems words (running→run) | English prose |
| Custom | configurable | Special requirements |

## Aggregations

```json
POST my-index/_search
{
  "size": 0,
  "aggs": {
    "by_category": {
      "terms": {
        "field": "category",   ← must be keyword type
        "size": 20             ← default is 10, increase if needed
      },
      "aggs": {
        "avg_price": { "avg": { "field": "price" } }
      }
    },
    "price_stats": { "stats": { "field": "price" } }
  }
}
```

- `terms` agg requires `keyword` field — text fields fail or give garbage
- Default `size: 10` — always set explicitly
- Cardinality is approximate (HyperLogLog) — exact requires full scan
- Nested aggs require `nested` query wrapper when field is `nested` type

## Pagination

```json
// ✅ Standard — up to 10,000 hits
{ "from": 0, "size": 20 }

// ✅ Deep pagination — search_after
POST my-index/_search
{
  "size": 20,
  "sort": [{ "updated_at": "desc" }, { "_id": "asc" }],
  "search_after": ["2026-02-08T10:00:00Z", "10"]   ← from last result
}

// ✅ Bulk export — Point in Time
POST my-index/_pit?keep_alive=5m
→ { "id": "pit-id..." }

POST _search
{
  "size": 1000,
  "pit": { "id": "pit-id...", "keep_alive": "5m" },
  "sort": [{ "_shard_doc": "asc" }]
}
```

- `from + size` limit: **10,000 hits** — fails beyond that
- `search_after` → for user-facing deep pagination
- PIT + `search_after` → for bulk export, consistent snapshot
- Scroll API → deprecated for user pagination, still ok for one-time exports

## Bulk Indexing

```json
POST _bulk
{ "index": { "_index": "my-index", "_id": "1" } }
{ "title": "Cherry Tomatoes", "price": 12.5 }
{ "index": { "_index": "my-index", "_id": "2" } }
{ "title": "Organic Spinach", "price": 11.2 }
```

Best practices:
- Batch size: 5–15 MB (not number of docs)
- Set `refresh=false` during bulk load, refresh manually after
- Always check response for partial failures — bulk can return 200 with individual errors
- Use `helpers.bulk` in Python client (see python-client-9.md)

## Index Management

### Templates
```json
PUT _index_template/my-template
{
  "index_patterns": ["my-index-*"],
  "template": {
    "settings": { "number_of_replicas": 1 },
    "mappings": { ... }
  }
}
```

### Zero-downtime reindex with alias
```json
POST _aliases
{
  "actions": [
    { "remove": { "index": "my-index-v1", "alias": "my-index" } },
    { "add":    { "index": "my-index-v2", "alias": "my-index" } }
  ]
}
```

### ILM for time-series (logs, metrics)
```json
PUT _ilm/policy/my-policy
{
  "policy": {
    "phases": {
      "hot":    { "actions": { "rollover": { "max_size": "50gb" } } },
      "warm":   { "min_age": "7d", "actions": { "shrink": { "number_of_shards": 1 } } },
      "delete": { "min_age": "30d", "actions": { "delete": {} } }
    }
  }
}
```

## Sharding

- Optimal shard size: **10–50 GB**
- Number of shards is fixed at creation — plan ahead
- Single-node local dev: `number_of_replicas: 0`
- Over-sharding kills performance — start with 1 shard for small indices

## Common Errors

| Error | Cause | Fix |
|---|---|---|
| `cluster_block_exception` | Disk > 85%, cluster read-only | Free disk, reset with `PUT _cluster/settings {"persistent":{"cluster.blocks.read_only_allow_delete": null}}` |
| `version_conflict_engine_exception` | Concurrent update | Use `retry_on_conflict: 3` or optimistic locking with `if_seq_no` |
| `circuit_breaker_exception` | Query uses too much memory | Reduce agg scope, add filters first |
| `index_not_found_exception` | Index doesn't exist | Create index or check name |
| `illegal_argument_exception: mapper [x] cannot be changed` | Changing field type after indexing | Reindex to new index |
| Mapping explosion | Dynamic mapping creating too many fields | Set `dynamic: "strict"` and explicit mappings |

## Performance Tips

- `_source: false` + `stored_fields` if you don't need full document
- `"profile": true` in query to see slow clauses
- Avoid leading wildcards `*term` — forces full scan, use `reverse` field instead
- Filter before scoring: `bool.filter` is cached, `bool.must` is not
- Disable `refresh_interval` during heavy indexing, re-enable after
