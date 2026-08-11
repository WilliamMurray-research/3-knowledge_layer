# Normalisation in a Metadata‑Driven Multi‑Layer Architecture  

## Abstract  
This whitepaper examines the role of normalisation in a three‑layer research‑corpus indexing system consisting of (1) directory‑level Markdown summaries, (2) a relational metadata layer, and (3) a graph‑based relationship layer. Unlike traditional systems that store raw documents or semi‑structured content directly in a database, this architecture stores only metadata in PostgreSQL. Normalisation therefore becomes a core efficiency strategy rather than a theoretical exercise. This paper explains why metadata‑only storage changes the normalisation calculus, how it improves performance, and how it enables predictable behaviour across the entire pipeline.

---

## 1. Introduction  
Normalisation is often taught as a method for reducing redundancy in relational databases. In practice, its value depends heavily on what type of data is being stored. In systems that ingest raw documents, logs, or semi‑structured content, normalisation may be impractical or unnecessary. However, in systems that store metadata, normalisation becomes a powerful tool for:

- improving efficiency  
- reducing write amplification  
- stabilising query performance  
- simplifying indexing  
- maintaining long‑term consistency  

This whitepaper describes how normalisation functions within a metadata‑only relational layer and why this design is particularly effective for a multi‑layer architecture.

---

## 2. System Overview  
The system consists of three distinct layers, each optimised for a specific role:

### 2.1 Source Layer — Directory‑Level Markdown  
Each project directory contains a README.md file. This file is the canonical human‑authored summary of the project. It is not stored in any database. Instead, it is read directly from the filesystem by the sync engine.

### 2.2 Metadata Layer — PostgreSQL  
PostgreSQL stores only structured metadata extracted from the README:

- project name  
- description  
- tags  
- timestamps  

This layer is fully normalised and acts as the authoritative source of project identity.

### 2.3 Relationship Layer — Memgraph  
Memgraph stores inter‑project relationships extracted by the local SLM:

- dependency edges  
- semantic relationships  
- future edge types  

This layer contains no metadata — only graph edges referencing canonical project nodes.

For deeper exploration, see metadata schema design or graph‑layer design.

---

## 3. Why Metadata‑Only Storage Changes the Normalisation Strategy  
Normalisation is most effective when applied to small, structured, frequently accessed data. Metadata fits this profile perfectly:

- It is small and atomic.  
- It is updated frequently via the sync engine.  
- It is queried often for indexing, filtering, and analytics.  
- It benefits from strict constraints and predictable structure.  

Storing raw Markdown in PostgreSQL would:

- increase row width  
- reduce cache locality  
- increase WAL volume  
- complicate indexing  
- introduce redundancy  
- blur the separation of concerns  

By keeping raw documents in the filesystem, the relational layer remains lean and efficient.

---

## 4. Normalisation Principles Applied to Metadata  

### 4.1 First Normal Form (1NF)  
Metadata values are atomic:

- tags are stored individually  
- descriptions are single text fields  
- project names are unique  

This ensures efficient indexing and predictable query planning.

### 4.2 Second Normal Form (2NF)  
Non‑key attributes depend solely on the project identifier:

- tags depend on project_id  
- descriptions depend on project_id  

This eliminates partial dependencies and reduces redundancy.

### 4.3 Third Normal Form (3NF)
No transitive dependencies exist:

- relationships are stored in Memgraph  
- metadata is stored in PostgreSQL  
- raw documents remain in the filesystem  

This architectural separation is a form of domain‑driven normalisation.

For deeper analysis, see normalisation for efficiency.

---

## 5. Efficiency Gains from Metadata Normalisation  

### 5.1 Reduced Write Amplification  
Metadata updates affect only the relevant atomic fields.  
No cascading updates occur.  
The sync engine performs minimal writes.

### 5.2 Improved Index Selectivity
Atomic metadata fields produce highly selective indexes:

- tags  
- project names  
- timestamps  

This improves query performance and planner accuracy.

### 5.3 Smaller Row Width  
Narrow tables:

- fit more rows per page  
- reduce cache misses  
- reduce I/O  
- improve sequential scan performance  

### 5.4 Lower WAL and Vacuum Load
Normalised metadata produces:

- fewer tuple rewrites  
- smaller WAL entries  
- less vacuum churn  

This stabilises long‑term performance.

### 5.5 Predictable Query Behaviour
Metadata queries remain stable because:

- structure does not drift  
- row width does not inflate  
- redundant fields do not accumulate  

This is essential for systems that evolve over time.

---

## 6. Why Relationships Are Not Stored in PostgreSQL  
Storing relationships in PostgreSQL would violate 3NF and reduce efficiency:

- relationships are not intrinsic metadata  
- they change more frequently than metadata  
- they require multi‑hop traversal  
- they benefit from graph algorithms  

Memgraph is optimised for:

- dependency chains  
- lineage analysis  
- semantic clustering  
- variable‑length path queries  

Normalisation at the architectural level dictates:

> Metadata → PostgreSQL  
> Relationships → Memgraph  
> Raw documents → Filesystem

For more detail, see relationship extraction.

---

## 7. The Role of the Sync Engine  
The sync engine enforces consistency across layers:

1. Reads README.md  
2. Extracts metadata via SLM  
3. Writes metadata to PostgreSQL  
4. Writes edges to Memgraph  

Normalisation reduces the sync engine’s workload:

- fewer fields to update  
- fewer redundant writes  
- simpler conflict resolution  
- predictable schema evolution  

See sync‑engine internals for more.

---

## 8. Long‑Term Benefits of Metadata Normalisation  

### 8.1 Schema Stability  
Normalised metadata schemas resist drift and bloat.

### 8.2 Predictable Performance  
Query plans remain stable as the corpus grows.

### 8.3 Extensibility  
New metadata fields can be added without restructuring existing tables.

### 8.4 Clean Layer Boundaries  
Each layer remains responsible for its own domain:

- filesystem → raw content  
- PostgreSQL → metadata  
- Memgraph → relationships  

This prevents cross‑layer contamination.

---

## 9. Conclusion  
Normalisation is not merely a theoretical construct in this system. It is a practical efficiency strategy that enables predictable behaviour, stable performance, and clean architectural separation. By storing only metadata in PostgreSQL, the system avoids redundancy, reduces write cost, and maintains clarity across layers. The result is a lightweight, scalable, and maintainable architecture that can support thousands of projects without degradation.

---

