# Elasticsearch Python Client 9.x

## Install

```bash
pip install "elasticsearch>=9.0.0" python-dotenv
```

Latest stable: **9.1.3** (PyPI, Feb 2026). 9.3.0 released 2026-02-03 — check PyPI for availability.

## Connect

```python
from elasticsearch import Elasticsearch
from dotenv import load_dotenv
import os

load_dotenv()

es = Elasticsearch(
    os.getenv("ELASTICSEARCH_URL"),
    api_key=os.getenv("ELASTICSEARCH_API_KEY")
)

info = es.info()
print(info["version"]["number"])  # "9.3.0"
```

## ⚠️ Breaking Change: No `body=` Parameter

The `body=` parameter was deprecated in 8.x and **removed in 9.x**.
Use keyword arguments directly instead.

```python
# ❌ Old (8.x) — raises TypeError in 9.x
es.search(index="my-index", body={"query": {"match_all": {}}})
es.indices.create(index="my-index", body={"mappings": {...}})

# ✅ New (9.x) — keyword args
es.search(index="my-index", query={"match_all": {}})
es.indices.create(index="my-index", mappings={...}, settings={...})
```

## Common Operations — 9.x Syntax

### Create index
```python
es.indices.create(
    index="fresh-produce",
    settings={"number_of_replicas": 0, "refresh_interval": "1s"},
    mappings={
        "properties": {
            "name":     {"type": "text"},
            "category": {"type": "keyword"},
            "price":    {"type": "float"},
            "semantic_text": {
                "type": "semantic_text",
                "inference_id": "jina-embeddings-v3"
            }
        }
    }
)
```

### Search
```python
results = es.search(
    index="fresh-produce",
    query={"semantic": {"field": "semantic_text", "query": "healthy greens"}},
    size=10,
    source=["name", "category", "price"]
)

for hit in results["hits"]["hits"]:
    print(hit["_source"])
```

### Inference endpoint
```python
# Check if exists, create if not
from elasticsearch import NotFoundError

try:
    es.inference.get(inference_id="jina-embeddings-v3")
except NotFoundError:
    es.inference.put(
        task_type="text_embedding",
        inference_id="jina-embeddings-v3",
        inference_config={
            "service": "jinaai",
            "service_settings": {
                "api_key": os.getenv("JINA_API_KEY"),
                "model_id": "jina-embeddings-v3"
            }
        }
    )
```

## helpers.bulk — Recommended for Bulk Indexing

Use `helpers.bulk` over raw `es.bulk` — handles chunking, retries, error reporting.

```python
from elasticsearch import helpers, NotFoundError

def generate_docs(documents, index_name):
    for doc in documents:
        yield {
            "_index": index_name,
            "_id": str(doc["id"]),
            **doc,
            "semantic_text": f"{doc['name']} {doc['description']} {doc['category']}"
        }

success, errors = helpers.bulk(
    es,
    generate_docs(documents, "fresh-produce"),
    refresh=True,
    raise_on_error=False   # collect errors instead of raising
)

if errors:
    for error in errors:
        print(f"Error: {error}")
```

### helpers.bulk vs es.bulk

| | `helpers.bulk` | `es.bulk` |
|---|---|---|
| Auto-chunking | ✅ (500 docs / chunk) | ❌ manual |
| Retry on failure | ✅ configurable | ❌ |
| Error reporting | ✅ returns (success, errors) | partial — check response |
| Recommended | ✅ | Only for low-level control |

## Type Hints (9.x)

```python
from elasticsearch import Elasticsearch

def connect() -> Elasticsearch:
    return Elasticsearch(ES_URL, api_key=API_KEY)

def create_index(es: Elasticsearch, index_name: str) -> None:
    es.indices.create(index=index_name, mappings={...})
```

## Error Handling

```python
from elasticsearch import NotFoundError, BadRequestError, AuthenticationException

try:
    es.indices.get(index="my-index")
except NotFoundError:
    print("Index doesn't exist")
except AuthenticationException:
    print("Invalid API key")
except BadRequestError as e:
    print(f"Bad request: {e.error}")
```

## Environment Pattern (.env)

```bash
# .env
ELASTICSEARCH_URL=http://localhost:9200
ELASTICSEARCH_API_KEY=base64encodedkey==
JINA_API_KEY=jina_xxxxxxxxxx
```

```python
from dotenv import load_dotenv
import os

load_dotenv()

ES_URL    = os.getenv("ELASTICSEARCH_URL")
ES_APIKEY = os.getenv("ELASTICSEARCH_API_KEY")
JINA_KEY  = os.getenv("JINA_API_KEY")

# Validate before using
missing = [k for k, v in {"ES_URL": ES_URL, "ES_APIKEY": ES_APIKEY}.items() if not v]
if missing:
    raise ValueError(f"Missing env vars: {missing}")
```

Always add `.env` to `.gitignore`.

## Suppress SSL Warning (LibreSSL on macOS)

```python
import warnings
import urllib3
warnings.filterwarnings("ignore", category=urllib3.exceptions.NotOpenSSLWarning)
```

Common on macOS with system Python 3.9 — harmless but noisy.
