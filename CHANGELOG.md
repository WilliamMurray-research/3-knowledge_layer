# Changelog

| **Version** | **Date** | **Key Changes** |
| --- | --- | --- |
| **v0.0.1** | **2026‑08‑06** | **Initial architecture and implementation outline for a dual‑database project‑sync system.** Establishes a Bash‑driven sync engine that reads project READMEs, invokes a local *[SLM extractor](ca://s?q=Explain_local_SLM_metadata_extraction)*, and writes metadata to PostgreSQL while storing dependency edges in Memgraph. Defines directory layout, setup steps, and usage patterns. Introduces optional *[FastAPI wrapper](ca://s?q=Explain_FastAPI_wrapper_design)*, offline‑first design, and extensibility via custom edge types and prompt modifications. Marks the beginning of a unified metadata + graph sync pipeline across all local projects. |
