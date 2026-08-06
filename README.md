# Project‑Sync System  
*(PostgreSQL + Memgraph + SLM Integration)*  

> **TL;DR** – A lightweight service that keeps a PostgreSQL table *and* a Memgraph graph in sync with every `README.md` you keep at the root of each project.  It updates automatically on request, on CI runs, or right before shutdown, and it can ask GPT‑4 to extract tags & relationships from the README.

---

## Table of Contents

| Section | What you’ll find |
|---------|-----------------|
| [Why this?](#why-this) | The problem we solve |
| [Architecture Overview](#architecture-overview) | How components fit together |
| [Setup Instructions](#setup-instructions) | Install, configure & run the service |
| [Usage](#usage) | API endpoints, CLI examples, shutdown behavior |
| [Extending & Customising](#extending--customising) | Add new edge types, change GPT prompt, etc. |
| [FAQ](#faq) | Common questions answered |
| [License](#license) | How you can reuse this project |

---

## Why This?

- **Single source of truth** – all projects live in one PostgreSQL table; the graph lives in Memgraph.
- **Automatic updates** – sync on CI, manual request or at graceful shutdown.  
  No need to remember `pg_dump` or `mg-admin dump`.
- **AI‑powered metadata extraction** – GPT‑4 reads your README and automatically generates:
  - *Title*, *description*, *tags* (array)  
  - *depends_on* project names (which become edges in the graph)
- **Extensible** – add new relationship types or GPT prompts without touching the core logic.

---

## Architecture Overview

```
┌───────────────────────┐
│   FastAPI Service       │
│   └──/sync/<project>   │
├───────────┬─────────────┤
│          │             │
│ 1. Request /sync/…    │
│ 2. CI job (post‑push)│
│ 3. Shutdown hook     │
└───────▲────────────────┘

            ▲
            │
            ▼

┌───────────────────────┐
│   Sync‑Engine          │
│   – reads README.md    │
│   – calls GPT‑4         │
│   – writes to Postgres  │
│   – writes to Memgraph  │
└───────────▲─────────────┘

            ▲
            │
            ▼

┌───────────────────────┐
│  PostgreSQL            │
│  projects, tags        │
└───────────────────────┘

            ▲
            │
            ▼

┌───────────────────────┐
│  Memgraph              │
│  nodes & relationships │
└───────────────────────┘
```

- **FastAPI** gives us a clean HTTP API + lifecycle events.  
- **Sync‑Engine** is pure Python; it can be called from any script or CI job.  
- **PostgreSQL** stores static data: `project_id`, `name`, `description`, tags, timestamps.  
- **Memgraph** stores *directed* edges (`DEPENDS_ON`, …) that let you traverse the project network.

---

## Setup Instructions

> **Prerequisites** – Python 3.10+, PostgreSQL ≥13, Memgraph 4.x, OpenAI API key (for GPT‑4).  

### 1️⃣ Create a virtual environment

```bash
python -m venv .venv
source .venv/bin/activate      # Linux / macOS
.\.venv\Scripts\Activate.ps1   # PowerShell on Windows
```

### 2️⃣ Install dependencies

```bash
pip install fastapi uvicorn psycopg2-binary memgraph-driver python-frontmatter openai
```

> `python-frontmatter` is optional (if you keep YAML front‑matter).  
> For GPT‑4 you’ll need the official OpenAI SDK.

### 3️⃣ PostgreSQL

```sql
CREATE DATABASE projects;
\c projects

-- create extensions ------------------------------------------------
CREATE EXTENSION IF NOT EXISTS "pgcrypto";   -- for gen_random_uuid()

-- tables ---------------------------------------------------------
CREATE TABLE projects (
    project_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL UNIQUE,
    description     TEXT,
    created_at      TIMESTAMP WITH TIME ZONE DEFAULT now(),
    updated_at      TIMESTAMP WITH TIME ZONE DEFAULT now()
);

CREATE TABLE project_tags (
    id          BIGSERIAL PRIMARY KEY,
    project_id  UUID REFERENCES projects(project_id) ON DELETE CASCADE,
    tag         TEXT
);
```

> **Tip** – Keep the DB credentials in a `.env` file:

```dotenv
PG_DSN=postgresql://postgres:secret@localhost:5432/projects
MG_URL=http://localhost:7687
OPENAI_API_KEY=sk-...
```

### 4️⃣ Memgraph

Download & install from the official site.  Start it with default settings or a custom config.

```bash
# Example (Linux)
./memgraph -c /etc/memgraph/config.json
```

> The default HTTP port is **7687** – used by `memgraph-driver`.

### 5️⃣ Put your README

Create `README.md` at the root of any project you want to track.  
Example:

```markdown
---
title: Example‑Alpha
description: A simple data pipeline that pulls CSVs into a Postgres table.
tags:
  - ETL
  - csv
depends_on:
  - Beta   # will create DEPENDS_ON edge to the project named “Beta”
---

# Example‑Alpha

This is a minimal example … 
```

> **If you don’t have YAML front‑matter** – GPT will still parse it (see section *GPT Extraction* below).

### 6️⃣ Run the service

```bash
uvicorn fastapi_app:app --host 0.0.0.0 --port 8000
```

The app listens on `http://localhost:8000`.

---

## Usage

| Action | Endpoint / Command | What it does |
|--------|-------------------|--------------|
| **Health check** | `GET /health` | Returns `{"status":"ok"}` |
| **Sync a single project** | `POST /sync/<project_name>` | Finds `<project_name>/README.md`, extracts metadata via GPT, writes to both DBs.  Response: `{ "uuid": "<UUID>" }` |
| **Manual sync of all README files** | Run `python -m fastapi_app.sync_all` (custom script) | Loops through every README and calls the engine – handy for local dev |
| **Graceful shutdown** | Send SIGINT (`Ctrl+C`) or kill the process | FastAPI’s `shutdown` event runs a final sync of all README files in the current working directory before exiting. |

### Example curl

```bash
curl -X POST http://localhost:8000/sync/Example-Alpha
```

Response:

```json
{
  "uuid": "3e8d5b23-2a9f-4c6d-b1bd-6ecb7e2b1ff7"
}
```

You can now query Postgres or Memgraph directly.

---

## GPT‑4 Extraction (Optional)

If you prefer deterministic parsing instead of GPT, replace the `extract_metadata()` function in `sync.py` with your own logic:

```python
def extract_metadata(content: str) -> dict:
    # Use frontmatter.load_file or regex to pull fields.
```

The rest of the sync engine stays unchanged.

---

## Extending & Customising

| Feature | How |
|---------|-----|
| **New edge type** | Add a new `MERGE (a)-[:NEW_EDGE]->(b)` line in `sync_from_path()`; add a check for `"new_edge"` key in GPT JSON. |
| **Custom GPT prompt** | Edit the string passed to `openai.ChatCompletion.create` – keep it short, but include your own instructions. |
| **Batch sync CLI** | Add a new FastAPI endpoint `/batch-sync` that accepts a list of project names. |
| **Graph analytics endpoint** | Create `/graph/<uuid>/downstream` returning all nodes reachable via `DEPENDS_ON`. |

---

## FAQ

- **Q: Do I need to keep the README in YAML format?**  
  **A:** No – GPT will parse plain Markdown.  If you use YAML front‑matter it’s faster and deterministic.

- **Q: What if two projects have the same name?**  
  **A:** PostgreSQL enforces `name` uniqueness.  Rename or add a prefix to avoid collision.

- **Q: How often does GPT run?**  
  **A:** Once per sync request (API call, CI job, shutdown).  You can cache results in a separate table if you worry about rate‑limits.

- **Q: Can I use another LLM provider?**  
  **A:** Absolutely.  Replace the `openai` import with your SDK and adjust the API key env var.

---

## License

MIT – feel free to copy, modify, or commercialise this project.  

> See the [LICENSE](./LICENSE) file for details.

--- 

