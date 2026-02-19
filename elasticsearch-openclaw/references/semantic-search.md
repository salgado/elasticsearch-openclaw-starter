# Semantic Search — JINA, ELSER, semantic_text

## semantic_text Field Type (ES 8.11+, recommended)

The simplest way to add semantic search. ES handles embedding generation automatically
at index time and query time via the Inference API — no manual vector management needed.

```json
PUT my-index
{
  "mappings": {
    "properties": {
      "title":   { "type": "text" },
      "content": { "type": "text" },
      "semantic_content": {
        "type": "semantic_text",
        "inference_id": "my-inference-endpoint"
      }
    }
  }
}
```

Index a document — embedding generated automatically:
```json
POST my-index/_doc
{
  "title": "Fresh Broccoli",
  "content": "Nutritious green broccoli, rich in vitamins.",
  "semantic_content": "Fresh Broccoli Nutritious green broccoli, rich in vitamins. vegetables"
}
```

Query:
```json
POST my-index/_search
{
  "query": {
    "semantic": {
      "field": "semantic_content",
      "query": "healthy leafy greens"
    }
  }
}
```

## Inference Endpoints — Setup

### JINA (jinaai service) — dense vectors, multilingual

```json
PUT _inference/text_embedding/jina-embeddings-v3
{
  "service": "jinaai",
  "service_settings": {
    "api_key": "jina_xxxxxx",
    "model_id": "jina-embeddings-v3"
  }
}
```

- Model: `jina-embeddings-v3` — 1024 dims, multilingual, best as of 2026
- Requires JINA API key: https://jina.ai/embeddings (free tier available)
- ⚠️ External inference: every query and indexing call reaches api.jina.ai
- Best for: multilingual content, new projects, open-source preference

### ELSER (Elastic-native) — sparse vectors, English

```json
PUT _inference/sparse_embedding/elser
{
  "service": "elasticsearch",
  "service_settings": {
    "model_id": ".elser_model_2",
    "adaptive_allocations": { "enabled": true }
  }
}
```

- Runs locally inside Elasticsearch — no external API calls
- Best for: English-only content, air-gapped environments, no external dependencies
- `sparse_embedding` task type → use with `semantic_text` or sparse_vector queries

### Check existing inference endpoints

```bash
GET _inference
```

## JINA vs ELSER — Decision Guide

| | JINA | ELSER |
|---|---|---|
| Vector type | Dense (1024 dims) | Sparse (tokens with weights) |
| Languages | 100+ | English only |
| Runs where | External API (jina.ai) | Inside Elasticsearch |
| Internet required | Yes (query + index) | No |
| Interpretable | No | Yes (token weights visible) |
| API cost | Yes (tokens) | Elastic license |
| Best for | Multilingual, new projects | English enterprise, air-gapped |

## semantic_text vs manual dense_vector

| | semantic_text | dense_vector (manual) |
|---|---|---|
| Embedding generation | Automatic | Manual (you call the API) |
| Complexity | Low | High |
| Flexibility | Less | More |
| Recommended | ✅ Yes, default choice | Only if you need custom control |

## Troubleshooting

- `inference_not_found` → inference endpoint doesn't exist, run PUT _inference first
- `model_not_deployed` → ELSER model not loaded, wait for deployment
- Slow first query after idle → adaptive allocations scaling up from 0, normal
- Embeddings not updating → re-index document, semantic_text does not auto-update on inference model change
