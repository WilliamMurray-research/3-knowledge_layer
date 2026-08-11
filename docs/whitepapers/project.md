
# Knowledge Layer: A Dual‑Layer Semantic Indexing and Provenance System for Research Ecosystems

## Abstract  
The Knowledge Layer is a dual‑layer indexing and relationship‑mapping system designed to unify, structure, and relate the entire research corpus (~8,500+ files). It synchronizes a PostgreSQL metadata table with a Memgraph relationship graph, using a lightweight Bash‑based sync engine and a locally hosted small‑language‑model (SLM) for offline metadata extraction. This whitepaper presents the system architecture, metadata model, synchronization protocol, provenance semantics, and extensibility mechanisms that define the Knowledge Layer as a foundational substrate for governed research ecosystems.

---

## 1. Introduction 
Large research ecosystems accumulate thousands of artefacts—papers, frameworks, proofs, algorithms, DSLs, specifications, and project READMEs. Without a governed indexing system, these artefacts become fragmented, untraceable, and semantically disconnected.

The Knowledge Layer solves this problem by providing:

- a single source of truth for project metadata (PostgreSQL),  
- a relationship graph for dependency edges (Memgraph), and  
- an offline‑first metadata extraction pipeline using a local SLM.

The attached document states:

> “A dual‑layer system indexing and relating the entire research corpus (~8,500 files).”

and:

> “Single source of truth – all project metadata lives in PostgreSQL; relationships live in Memgraph.”

This system forms the semantic substrate required for programme‑level governance and cross‑domain coherence.

---

## 2. Motivation  
Research ecosystems require:

- semantic indexing  
- dependency mapping  
- provenance tracking  
- offline metadata extraction  
- automatic synchronization  

Traditional tooling (GitHub APIs, cloud services, manual tagging) fails to meet these constraints.

The attached document explains the motivation clearly:

> “Automatic updates – sync on CI, manual request, or graceful shutdown.”  
> “Offline‑first – no API keys, no cloud dependencies, no rate limits.”

The Knowledge Layer is designed to be:

- deterministic  
- governed  
- offline  
- extensible  
- metadata‑rich  

It is the semantic backbone of the entire research programme.

---

## 3. System Overview  
The Knowledge Layer consists of four components:

### 3.1 Sync Engine (Bash + curl + jq)  
Responsible for:

- reading project README.md files  
- calling the local SLM  
- writing metadata to PostgreSQL  
- writing edges to Memgraph  

The document states:

> “Sync‑Engine is pure Bash + curl + jq.”

### 3.2 Local SLM (llama.cpp or compatible)  
Extracts:

- title  
- description  
- tags[]  
- depends_on[]  

As described:

> “A small SLM reads your README and generates: title, description, tags (array), depends_on project names.”

### 3.3 PostgreSQL (metadata layer)  
Stores:

- project names  
- descriptions  
- timestamps  
- tags  

### 3.4 Memgraph (relationship layer)  
Stores:

- dependency edges  
- custom relationship types  
- graph analytics  

---

## 4. Architecture  
`
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
`

The architecture is explicitly described in the attached document:

> “FastAPI is optional — the system works perfectly without it.”

---

## 5. Directory Layout  
`
project-sync/
├── sync.sh
├── parse_slm.sh
├── db_postgres.sh
├── db_memgraph.sh
└── projects/
    ├── Alpha/README.md
    ├── Beta/README.md
    └── Gamma/README.md
`

This minimal layout ensures deterministic behaviour and easy extension.

---

## 6. Metadata Model  

### 6.1 PostgreSQL Schema
The attached document provides the exact schema:

> *“CREATE TABLE projects (  
>     project_id UUID PRIMARY KEY,  
>     name TEXT NOT NULL UNIQUE,  
>     description TEXT,  
>     created_at TIMESTAMP,  
>     updated_at TIMESTAMP  
> );”*

Tags are stored separately:

> *“CREATE TABLE project_tags (  
>     id BIGSERIAL PRIMARY KEY,  
>     project_id UUID REFERENCES projects,  
>     tag TEXT  
> );”*

### 6.2 Memgraph Schema  
Dependency edges are stored as:

`
(a)-[:DEPENDS_ON]->(b)
`

Additional edge types can be added by modifying db_memgraph.sh.

---

## 7. Synchronization Protocol  

### 7.1 Step 1 — Read README.md  
Sync engine loads the project’s README.

### 7.2 Step 2 — Call Local SLM  
The attached document shows the exact call:

> “Extract JSON metadata: title, description, tags[], depends_on[]”

### 7.3 Step 3 — Write Metadata to PostgreSQL  
Metadata is inserted or updated.

### 7.4 Step 4 — Write Edges to Memgraph  
Dependency edges are created or updated.

### 7.5 Step 5 — Optional FastAPI Trigger  
FastAPI provides:

- /sync/<project> endpoint  
- remote triggering  
- CI integration  

---

## 8. Local SLM Extraction  
The SLM is called via an OpenAI‑style API:

> “curl -s http://localhost:8080/v1/chat/completions …”

The SLM returns structured JSON:

> “{ "title": "Example-Alpha", "tags": ["ETL", "csv"], "depends_on": ["Beta"] }”

This ensures deterministic metadata extraction.

---

## 9. Workflow Semantics  

### 9.1 Sync a Single Project  
`
./sync.sh Alpha
`

### 9.2 Sync All Projects  
`
for p in projects/*; do ./sync.sh "$(basename "$p")"; done
`

### 9.3 Shutdown Sync  
A systemd service triggers sync on shutdown.

### 9.4 CI Sync  
CI pipelines can call FastAPI or the Bash engine directly.

---

## 10. Extensibility

| Feature | How to Extend |
|--------|----------------|
| New edge type | Add new Cypher MERGE in db_memgraph.sh |
| Custom SLM prompt | Edit system message in parse_slm.sh |
| Batch sync | Add wrapper script |
| Graph analytics | Add FastAPI route calling Memgraph |

The attached document states:

> “Extensible – add new relationship types or modify the SLM prompt without touching core logic.”

---

## 11. Design Principles  

| Principle | Meaning |
|----------|---------|
| Offline‑First | No API keys, no cloud dependencies |
| Deterministic | Sync produces stable metadata and edges |
| Minimal | Bash + curl + jq; optional FastAPI |
| Governed | Metadata and relationships are canonical |
| Extensible | New edge types and prompts are trivial |

---

## 12. Applications  

The Knowledge Layer is ideal for:

- large research ecosystems  
- provenance tracking  
- dependency mapping  
- semantic indexing  
- project governance  
- structural invariants  
- programme‑level coherence  

It is the semantic substrate for the Unified Asset Registry and the Project Template Framework.

---

## 13. Conclusion  
The Knowledge Layer provides a governed, deterministic, offline‑first indexing system for research ecosystems. By combining PostgreSQL metadata, Memgraph relationships, Bash‑based synchronization, and local SLM extraction, it forms the semantic backbone required for programme‑level governance, provenance tracking, and cross‑domain coherence.

---

