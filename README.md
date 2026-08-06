# Project‑Sync System  
(*Bash + curl, PostgreSQL + Memgraph + Local SLM Integration*)  

> **TL;DR** – A lightweight service that keeps a PostgreSQL table *and* a Memgraph graph in sync with every `README.md` you keep at the root of each project. It updates automatically on request, on CI runs, or right before shutdown, and it can ask a **locally hosted SLM** (Gemma‑4 e2B/e4B, Llama‑3B, etc.) to extract tags & relationships from the README.

---

## Table of Contents

| Section | What you’ll find |
|---------|-----------------|
| [Why this?](#why-this) | The problem we solve |
| [Architecture Overview](#architecture-overview) | How components fit together |
| [Setup Instructions](#setup-instructions) | Install, configure & run the service |
| [Usage](#usage) | API endpoints, CLI examples, shutdown behavior |
| [Extending & Customising](#extending--customising) | Add new edge types, change SLM prompt, etc. |
| [FAQ](#faq) | Common questions answered |
| [License](#license) | How you can reuse this project |

---

## Why This?

- **Single source of truth** – all projects live in one PostgreSQL table; the graph lives in Memgraph.
- **Automatic updates** – sync on CI, manual request or at graceful shutdown.  
- **Local AI‑powered metadata extraction** – A small local SLM reads your README and generates:
  - *title*, *description*, *tags* (array)  
  - *depends_on* project names (which become edges in the graph)
- **Extensible** – add new relationship types or change the SLM prompt without touching core logic.
- **No cloud dependencies** – runs entirely offline using llama.cpp or any local inference server.

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
│   Sync‑Engine (Bash)   │
│   – reads README.md    │
│   – calls local SLM    │
│   – writes to Postgres │
│   – writes to Memgraph │
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

- **FastAPI** is optional — you can run syncs entirely via Bash.  
- **Sync‑Engine** is pure Bash + curl + jq.  
- **PostgreSQL** stores structured metadata.  
- **Memgraph** stores directed edges (`DEPENDS_ON`, etc.).

---

## Minimal Directory Layout

```
project-sync/
├── sync.sh
├── parse_slm.sh
├── db_postgres.sh
├── db_memgraph.sh
└── projects/
    ├── Alpha/README.md
    ├── Beta/README.md
    └── Gamma/README.md
```

---

## Setup Instructions

> **Prerequisites** – Bash + curl, PostgreSQL ≥13, Memgraph 4.x, llama.cpp (or any local inference server).

### 1️⃣ Install PostgreSQL

```sql
CREATE DATABASE projects;
\c projects

CREATE EXTENSION IF NOT EXISTS "pgcrypto";

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

### 2️⃣ Install Memgraph

```bash
./memgraph -c /etc/memgraph/config.json
```

Default HTTP port: **7687**

### 3️⃣ Start your local SLM

Example (llama.cpp server):

```bash
./server -m models/gemma-4-e2b.gguf -c 4096
```

This exposes an OpenAI‑style API at:

```
http://localhost:8080/v1/chat/completions
```

### 4️⃣ Add your README files

Each project must have:

```
projects/<ProjectName>/README.md
```

YAML front‑matter is optional.

---

## Usage

| Action | Command / Endpoint | What it does |
|--------|--------------------|--------------|
| **Sync a single project** | `./sync.sh Alpha` | Reads README, calls SLM, writes to both DBs |
| **Sync all projects** | `for p in projects/*; do ./sync.sh "$(basename "$p")"; done` | Batch sync |
| **FastAPI endpoint (optional)** | `POST /sync/<project>` | Wraps the Bash sync engine |
| **Shutdown sync** | systemd service | Runs sync on shutdown |

### Example

```bash
./sync.sh Example-Alpha
```

Output:

```json
{ "uuid": "3e8d5b23-2a9f-4c6d-b1bd-6ecb7e2b1ff7" }
```

---

## Local SLM Extraction

Replace GPT‑4 with your local model:

```bash
curl -s http://localhost:8080/v1/chat/completions \
  -d '{
    "model": "local-slm",
    "messages": [
      { "role": "system", "content": "Extract JSON metadata: title, description, tags[], depends_on[]" },
      { "role": "user", "content": "'"$README_CONTENT"'" }
    ]
  }'
```

Parse with `jq`.

---

## Extending & Customising

| Feature | How |
|---------|-----|
| **New edge type** | Add a new Cypher `MERGE (a)-[:NEW_EDGE]->(b)` in `db_memgraph.sh` |
| **Custom SLM prompt** | Edit the system message in `parse_slm.sh` |
| **Batch sync CLI** | Add a wrapper script that loops through `projects/*` |
| **Graph analytics endpoint** | Add FastAPI route calling Memgraph Cypher queries |

---

## FAQ

- **Q: Do I need YAML front‑matter?**  
  **A:** No — the SLM can parse plain Markdown.

- **Q: Do I need an API key?**  
  **A:** No — local SLMs require no authentication.

- **Q: How often does the SLM run?**  
  **A:** Once per sync request.

- **Q: Can I use a different local model?**  
  **A:** Yes — anything callable via HTTP or CLI works.

---

## License

MIT — feel free to copy, modify, or commercialise this project.

---
