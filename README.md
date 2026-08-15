`2026-1003-D-read-001.md`  

---

**CLASSIFICATION**: D  

**Document Reference**: `2026-1003-D-read-001`   
# Project‑Sync System 
### Project      

**Type**: read  
**Classification**: D  
**Version**: 0.1     

William Murray  
Systems Architect  
15 August 2026  

**Status**: Draft     

**Scope**: A lightweight, offline‑first synchronisation system that keeps PostgreSQL metadata and Memgraph relationships aligned with README.md files across local projects. Designed as a personal learning exercise in Bash‑based automation, graph‑database modelling, and local SLM‑powered metadata extraction.  

**Primary Model / Scheme**: README Metadata Schema v0.1 — defines title, description, tags, dependency edges, and project‑level attributes extracted by the local SLM.

---

> **TL;DR** – A lightweight system that keeps a PostgreSQL table *and* a Memgraph graph in sync with every `README.md` stored at the root of each project. It updates automatically on request, via CI, or at shutdown. A **locally hosted SLM** (Gemma‑4 e2B/e4B, Llama‑3B, etc.) extracts metadata such as tags and dependency relationships — entirely offline.

---

## Table of Contents

| Section | What you’ll find |
|---------|------------------|
| [Why This?](#why-this) | The problem we solve |
| [Architecture Overview](#architecture-overview) | How components fit together |
| [Setup Instructions](#setup-instructions) | Install, configure & run the system |
| [Usage](#usage) | CLI examples, optional API endpoints |
| [Extending & Customising](#extending--customising) | Add new edge types, change SLM prompt |
| [FAQ](#faq) | Common questions |
| [License](#license) | How you can reuse this project |

---

## Why This?

- **Single source of truth** – all project metadata lives in PostgreSQL; relationships live in Memgraph.
- **Automatic updates** – sync on CI, manual request, or graceful shutdown.
- **Local AI‑powered metadata extraction** – A small SLM reads your README and generates:
  - *title*, *description*, *tags* (array)  
  - *depends_on* project names → stored as graph edges
- **Offline‑first** – no API keys, no cloud dependencies, no rate limits.
- **Extensible** – add new relationship types or modify the SLM prompt without touching core logic.

---

## Architecture Overview

```
┌──────────────────────────┐
│   Optional FastAPI Layer │
│   └── POST /sync/<name>  │
└───────────▲──────────────┘
            │
            ▼
┌──────────────────────────┐
│   Sync‑Engine (Bash)      │
│   – reads README.md       │
│   – calls local SLM       │
│   – writes to PostgreSQL  │
│   – writes to Memgraph    │
└───────────▲──────────────┘
            │
            ▼
┌──────────────────────────┐
│   PostgreSQL (metadata)   │
└──────────────────────────┘
            ▲
            ▼
┌──────────────────────────┐
│   Memgraph (graph edges)  │
└──────────────────────────┘
```

- **Sync‑Engine** is pure Bash + curl + jq.  
- **SLM** runs locally via llama.cpp or any compatible inference server.  
- **FastAPI** is optional — the system works perfectly without it.

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

> **Prerequisites** – Bash, curl, jq, PostgreSQL ≥13, Memgraph 4.x, llama.cpp (or any local inference server).

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
| **Optional FastAPI wrapper** | `POST /sync/<project>` | Calls the Bash sync engine |
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

The sync engine sends the README to your local model:

```bash
curl -s http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "local-slm",
    "messages": [
      { "role": "system", "content": "Extract JSON metadata: title, description, tags[], depends_on[]" },
      { "role": "user", "content": "'"$README_CONTENT"'" }
    ]
  }'
```

The SLM returns JSON such as:

```json
{
  "title": "Example-Alpha",
  "description": "A simple pipeline...",
  "tags": ["ETL", "csv"],
  "depends_on": ["Beta"]
}
```

Parsed with `jq`.

---

## Extending & Customising

| Feature | How |
|---------|-----|
| **New edge type** | Add a new Cypher `MERGE (a)-[:NEW_EDGE]->(b)` in `db_memgraph.sh` |
| **Custom SLM prompt** | Edit the system message in `parse_slm.sh` |
| **Batch sync CLI** | Add a wrapper script looping through `projects/*` |
| **Graph analytics endpoint** | Add FastAPI route calling Memgraph Cypher queries |

---

## FAQ

- **Q: Do I need YAML front‑matter?**  
  **A:** No — the SLM can parse plain Markdown.

- **Q: Do I need an API key?**  
  **A:** No — everything runs locally.

- **Q: How often does the SLM run?**  
  **A:** Once per sync request.

- **Q: Can I use a different local model?**  
  **A:** Yes — any model callable via HTTP or CLI works.

- **Q: Can I remove FastAPI entirely?**  
  **A:** Yes — the Bash sync engine is fully standalone.

---

## License

MIT — feel free to copy, modify, or commercialise this project.

---

**Contributions are off**
