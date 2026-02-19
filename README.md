# Elasticsearch + OpenClaw Starter 🦞🔍

Starter kit for exploring Elasticsearch 9.x with an AI agent via OpenClaw.
Includes semantic search with JINA, observability logs, and a ready-to-use workspace.

---

## Prerequisites

- **Docker Desktop** installed and running
- **JINA API Key** (free): https://jina.ai/embeddings
- **OpenClaw** installed: https://openclaw.ai

---

## Quick Start

### 1. Start Elasticsearch locally

```bash
curl -fsSL https://elastic.co/start-local | sh
```

Elasticsearch at `http://localhost:9200`, Kibana at `http://localhost:5601`.
Credentials auto-generated in `elastic-start-local/.env`.

---

### 2. Set up indices in Kibana DevTools

Go to `http://localhost:5601` → **Dev Tools** and run the files below **in order**:

| File | What it does |
|---|---|
| `devtools_fresh_produce_jina.md` | ← **Run first** — fresh produce index + API Key |
| `devtools_app_logs_synthetic.md` | ← Run after — observability logs index |

> ⚠️ In Step 1 of the first file, replace `YOUR_JINA_API_KEY` with your JINA key.
> ⚠️ In Step 5, save the `encoded` field from the API Key response — it cannot be retrieved later.

---

### 3. Set up the OpenClaw workspace

```bash
# Fill in your credentials
cp openclaw-workspace-elastic-blog/.env.example openclaw-workspace-elastic-blog/.env
# Edit .env with your ELASTICSEARCH_URL and ELASTICSEARCH_API_KEY

# Install the skill
cp -R elasticsearch-openclaw ~/.openclaw/skills/
# Or once published on ClawHub:
# clawhub install elasticsearch-openclaw

# Create the agent
openclaw agents add elasticsearch-agent \
  --workspace ~/path/to/openclaw-workspace-elastic-blog \
  --non-interactive

openclaw gateway restart
```

---

### 4. Test it

```bash
# Exploration
openclaw agent --agent elasticsearch-agent --message "What indices exist in the cluster?"

# Semantic search
openclaw agent --agent elasticsearch-agent --message "Refreshing fruits for hot days"

# Observability
openclaw agent --agent elasticsearch-agent --message "How many 500 errors occurred in the last 2 hours?"

# Surprise use case 🎁
openclaw agent --agent elasticsearch-agent --message "Generate a beautiful HTML report with products on sale that match today's weather, using the image_url field from each product. Save to ~/Desktop/produce-report.html and open in the browser."
```

---

## Repository structure

```
elasticsearch-openclaw-starter/
├── devtools_fresh_produce_jina.md     ← Part 1: fresh produce index + API Key
├── devtools_app_logs_synthetic.md     ← Part 2: observability logs index
├── openclaw-workspace-elastic-blog/
│   ├── .env.example                   ← credentials template
│   └── AGENTS.md                      ← agent briefing
└── elasticsearch-openclaw/            ← ES 9.x skill
    ├── SKILL.md
    └── references/
        ├── semantic-search.md
        ├── vector-search.md
        ├── classic-patterns.md
        └── python-client-9.md
```

---

## OpenClaw architecture

```
OpenClaw
├── Agents         ← isolated assistant instances
│   └── each agent has its own workspace
├── Workspace      ← defines who the agent is
│   ├── AGENTS.md  ← permanent briefing
│   ├── .env       ← credentials
│   └── skills/    ← agent-specific skills (optional)
└── Skills         ← reusable technical knowledge
    ├── ~/.openclaw/skills/  ← global (all agents)
    └── bundled              ← shipped with OpenClaw
```

The agent reads `AGENTS.md`, loads credentials from `.env`, and consults skills automatically.
No extra servers needed — just OpenClaw + Elasticsearch.

---

## Complementary skills

After setup, explore other skills that work well with Elasticsearch:

| Skill | What it does |
|---|---|
| `weather` (already installed) | Combines current weather with semantic search |
| `coding-agent` (already installed) | Writes and runs Python scripts for ES 9.x |
| `summarize` | Summarizes Elastic docs and blog posts |

---

## License

MIT
