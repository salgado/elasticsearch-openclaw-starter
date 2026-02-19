---
name: elasticsearch-openclaw
description: >
  Complete Elasticsearch 9.x reference for AI-orchestrated applications. Covers modern
  and classic patterns, all updated for ES 9.x. Use when working with Elasticsearch for:
  (1) Semantic search with JINA embeddings or ELSER via Elastic Inference API,
  (2) semantic_text field type with automatic embedding generation at index and query time,
  (3) kNN vector search, dense_vector mappings, bbq_hnsw index options,
  (4) Hybrid search combining BM25 + kNN with RRF (Reciprocal Rank Fusion),
  (5) Elasticsearch Python client 9.x — no body= parameter, keyword args, helpers.bulk,
  (6) Classic ES patterns updated for 9.x: mappings, text/keyword, analyzers,
      query vs filter context, aggregations, pagination, bulk indexing,
  (7) API key creation with least-privilege, read-only scoping per index,
  (8) Local development with Docker via elastic.co/start-local,
  (9) Common errors and performance gotchas.
metadata:
  openclaw:
    emoji: 🔍
    requires:
      anyBins: ["curl", "python3"]
    os: ["linux", "darwin", "win32"]
    env:
      ELASTICSEARCH_URL: "Elasticsearch cluster URL (required)"
      ELASTICSEARCH_API_KEY: "Base64-encoded API key (required, secret)"
---

# Elasticsearch OpenClaw 🔍

Modern Elasticsearch 9.x patterns for AI-orchestrated applications.

## Quick Start — Local Dev

```bash
# Spin up ES 9.x + Kibana with one command
curl -fsSL https://elastic.co/start-local | sh
# Elasticsearch: http://localhost:9200
# Kibana:        http://localhost:5601
```

Credentials are auto-generated in `elastic-start-local/.env`.

## Auth — Always Use API Keys

```bash
# Test connection
curl -s "$ELASTICSEARCH_URL" -H "Authorization: ApiKey $ELASTICSEARCH_API_KEY"

# Python client 9.x
from elasticsearch import Elasticsearch
es = Elasticsearch(ES_URL, api_key=API_KEY)
```

## Reference Files

Load these only when needed — do not load all at once:

| File | Load when... |
|------|-------------|
| `references/semantic-search.md` | Setting up JINA, ELSER, `semantic_text`, inference endpoint |
| `references/vector-search.md` | kNN queries, `dense_vector` mapping, hybrid search with RRF |
| `references/classic-patterns.md` | Mapping design, aggregations, pagination, bulk indexing |
| `references/python-client-9.md` | Python `elasticsearch` 9.x — no `body=`, `helpers.bulk`, type hints |

## When to Use Each Pattern

```
User asks about meaning / intent / "find products like X"
  → semantic_text + semantic query  →  references/semantic-search.md

User needs exact match + semantic combined
  → hybrid search (RRF)            →  references/vector-search.md

User asks about mapping, field types, analyzers, aggregations
  → classic patterns                →  references/classic-patterns.md

User uses Python elasticsearch library
  → always check                    →  references/python-client-9.md
```

## Security Best Practices

- Always use API keys over username/password
- Scope API keys to specific indices and minimal privileges
- For read-only OpenClaw access: `privileges: ["read", "view_index_metadata"]`
- Store credentials in `.env`, never hardcode in scripts
- `.env` always in `.gitignore`

```json
POST /_security/api_key
{
  "name": "openclaw-readonly",
  "role_descriptors": {
    "reader": {
      "indices": [{ "names": ["my-index"], "privileges": ["read"] }]
    }
  }
}
```

Save the `encoded` field immediately — it cannot be retrieved later.
