A Dual‑Layer Metadata Schema for Research Corpus Indexing: Design Principles, Structure, and Normalisation

Abstract
This paper presents a schema architecture for a dual‑layer metadata system that synchronises structured relational data in PostgreSQL with graph‑based relationship data in Memgraph. The system indexes and relates a large research corpus (~8,500 files) using a lightweight Bash‑driven sync engine and a local small‑language‑model (SLM) for metadata extraction. We outline the schema design, justify normalisation choices, and explain how relational and graph layers complement each other to form a unified knowledge layer.

---

1. Introduction
Large research repositories often contain thousands of project folders, each with its own README.md describing purpose, dependencies, and conceptual relationships. Traditional relational databases excel at storing structured metadata, while graph databases excel at representing relationships. The Project‑Sync System integrates both:

- PostgreSQL stores canonical project metadata.
- Memgraph stores inter‑project relationships extracted by a local SLM.
- A Bash sync engine ensures consistency between layers.

This paper focuses on the schema design underpinning this architecture, with special attention to normalisation, relationship modelling, and cross‑system consistency.

---

2. Schema Overview

2.1 Relational Layer (PostgreSQL)
The relational schema stores project‑level metadata extracted from README.md files.

Tables
- projects
  - project_id (UUID, PK)  
  - name (TEXT, UNIQUE)  
  - description (TEXT)  
  - created_at (TIMESTAMPTZ)  
  - updated_at (TIMESTAMPTZ)

- project_tags
  - id (BIGSERIAL, PK)  
  - project_id (UUID, FK → projects)  
  - tag (TEXT)

This schema is intentionally minimal. It stores only intrinsic metadata: attributes that describe a project itself, not its relationships.

---

2.2 Graph Layer (Memgraph)
The graph schema stores inter‑project relationships, such as:

- (:Project)-[:DEPENDS_ON]->(:Project)

Future extensions may include:

- [:RELATED_TO]
- [:USESDATAFROM]
- [:CONTAINS_MODULE]

Graph databases excel at representing variable‑length, multi‑hop relationships, making them ideal for dependency analysis, lineage tracking, and semantic clustering.

---

3. Normalisation Principles

3.1 Why Normalisation Matters
Normalisation ensures:

- elimination of redundancy  
- consistency across updates  
- efficient querying  
- clear separation of concerns  

In a dual‑layer system, normalisation also prevents cross‑system duplication. Metadata belongs in PostgreSQL; relationships belong in Memgraph.

---

4. Normal Forms Applied

4.1 First Normal Form (1NF)
All tables contain atomic values:

- Tags are stored as individual rows in project_tags.
- No arrays or comma‑separated lists inside columns.

This ensures clean indexing and simple joins.

---

4.2 Second Normal Form (2NF)
Every non‑key attribute depends on the whole key:

- In projecttags, the tag depends on projectid, not on the surrogate key id.

This avoids partial dependencies.

---

4.3 Third Normal Form (3NF)
No transitive dependencies:

- Project descriptions depend only on the project.
- Tags depend only on the project.
- Relationships are stored externally in Memgraph, not in PostgreSQL.

This separation is crucial: relationships are not intrinsic metadata.

---

5. Why Relationships Are Not Stored in PostgreSQL
Relational databases can store relationships, but they do not excel at:

- multi‑hop traversal  
- dependency chain analysis  
- graph analytics  
- community detection  
- semantic clustering  

Graph databases provide:

- constant‑time edge traversal  
- native Cypher queries  
- built‑in algorithms (PageRank, BFS, DFS)

Thus, normalisation at the architectural level dictates:

> Metadata → PostgreSQL  
> Relationships → Memgraph

This is a form of domain‑driven normalisation.

---

6. Cross‑Layer Consistency
The sync engine ensures:

- PostgreSQL is the single source of truth for project identity.  
- Memgraph stores only edges referencing canonical project nodes.  
- Updates propagate atomically:  
  - Read README → Extract metadata → Write to PostgreSQL → Write edges to Memgraph.

This avoids duplication and maintains referential integrity across systems.

---

7. Extended Schema Considerations

7.1 Additional Metadata Tables
Future expansions may include:

- project_dependencies  
- project_authors  
- project_versions  
- project_modules  

Each should follow 3NF principles.

---

7.2 Additional Graph Edge Types
Examples:

- [:SIMILAR_TO]  
- [:DERIVED_FROM]  
- [:CONTAINS]  
- [:USES_LIBRARY]  

These edges should remain exclusively in Memgraph.

---

8. Benefits of This Schema Design

8.1 Scalability
Normalised relational tables scale to tens of thousands of projects.  
Graph edges scale to millions of relationships.

8.2 Flexibility
New edge types can be added without modifying relational schema.

8.3 Offline‑First AI Integration
The SLM extracts metadata without cloud dependencies.

8.4 Clean Separation of Concerns
Metadata and relationships remain distinct, reducing complexity.

---

9. Conclusion
The dual‑layer schema described here provides a robust foundation for indexing and relating a large research corpus. Normalisation ensures consistency and scalability, while the graph layer enables rich relationship analysis. Together, PostgreSQL and Memgraph form a unified knowledge layer that is lightweight, extensible, and fully offline‑capable.

---

