# DevFlow — Technical Architecture

This document is a focused technical overview of how DevFlow works. For the full product specification, see [`product-spec.md`](product-spec.md). For the demo scenario and sample data, see [`demo-storyline.md`](demo-storyline.md).

---

## 1. System Overview

DevFlow is a multi-agent RAG (retrieval-augmented generation) system that ingests heterogeneous engineering knowledge sources, indexes them in a vector database, and serves queries through three distinct user-facing modes.

![Architecture](../diagrams/architecture.svg)

The architecture follows a clean separation:

- **Ingestion layer** pulls data from each source on a schedule or trigger, normalizes it, chunks it, and writes embeddings to the vector store.
- **Storage layer** holds per-source vector collections, allowing source-aware retrieval and citation.
- **Agent layer** orchestrates retrieval and synthesis through specialized agents, each with a focused responsibility.
- **Presentation layer** exposes three interfaces (Departure, Onboarding, Knowledge Map) that consume the agent layer through a common API.

---

## 2. Tech Stack

| Component | Technology | Rationale |
|---|---|---|
| Backend framework | Python + FastAPI | Async-native, fast iteration, strong type support |
| Agent orchestration | LangGraph | Multi-agent workflows with explicit state and routing |
| LLM | Anthropic Claude | Long context window, strong at structured analysis |
| Vector database | Chroma | Lightweight, fast setup, sufficient metadata filtering |
| Embeddings | sentence-transformers | Local, free, good quality for code + natural language |
| Git parsing | gitpython + subprocess | Stable parsing of `git log`, `git blame`, `git show` |
| Frontend | React + Tailwind CSS | Fast UI iteration, rich component ecosystem |
| Visualization | Recharts / D3 | Knowledge map heatmap and treemap |

---

## 3. Data Sources

Four heterogeneous sources, each with distinct semantics:

### 3.1 Git commit history
Authored commits, including bodies (the "why" of detailed commit messages). Indexed with metadata: author, files changed, timestamp, parent commit, co-authors.

### 3.2 Pull request descriptions
PR titles, descriptions, alternatives-considered sections, and known limitations. Often the highest signal-to-noise source for design rationale.

### 3.3 Inline code comments
Docstrings, `# WARNING`, `# HACK`, `# NOTE`, and `# TODO` blocks. Extracted with file path and line range for precise citation.

### 3.4 Captured exit interview answers
AI-conducted interview transcripts from Departure Mode, structured as Q&A pairs with metadata: departing engineer, related files/modules, date captured.

Each source is ingested into its own vector collection so the retriever can reason about source provenance and surface multi-source citations.

---

## 4. Ingestion Pipeline

The ingestion pipeline runs in four steps:

1. **Source connector** — pulls raw data from the source (git log export, GitHub PR API, AST parsing for comments, structured JSON for interviews).
2. **Normalizer** — converts source-specific format to a common `Document` schema with `text`, `source_type`, `metadata`, and `created_at`.
3. **Chunker** — splits long documents into retrieval-sized chunks. Commit messages are usually a single chunk; PR descriptions are split by section; code comment blocks are paired with surrounding signature context.
4. **Embedder + writer** — runs sentence-transformer encoding and writes to the appropriate Chroma collection with full metadata.

Re-ingestion is incremental: each connector tracks the latest processed timestamp and only ingests new records.

---

## 5. Vector Store Schema

Four Chroma collections:

| Collection | Document Type | Key Metadata |
|---|---|---|
| `commit_history` | Commit message + body | author, sha, files_changed, timestamp |
| `pr_descriptions` | PR title + body sections | author, pr_number, files_touched, status |
| `code_comments` | Comment block + nearest signature | file_path, line_start, line_end, function_name |
| `exit_interviews` | Q&A pair | engineer, related_files, capture_date, question_type |

Embeddings use a single sentence-transformer model across all collections to enable cross-collection similarity scoring during retrieval.

---

## 6. Agent Layer

Three specialized agents, orchestrated through LangGraph:

### 6.1 Footprint Analyzer
**Used by:** Departure Mode.
**Input:** repository identifier + departing engineer username.
**Output:** structured footprint report — owned modules, bus-factor risks, key decisions (commits with detailed messages), co-author relationships.

The analyzer pulls git data via the connector, runs author-specific queries, and uses Claude to summarize ownership patterns into a JSON footprint.

### 6.2 Interview Generator
**Used by:** Departure Mode.
**Input:** footprint report + corresponding commits, PRs, and code comments.
**Output:** 8–10 targeted exit interview questions.

Generated questions are intentionally specific — referencing actual files, decisions, and commits — so the engineer is prompted to recall context rather than answering generic questions like "what should the next person know?"

### 6.3 Knowledge Retriever
**Used by:** Onboarding Assistant.
**Input:** natural-language question.
**Output:** grounded answer with multi-source citations.

The retriever performs parallel similarity searches across all four collections, ranks results by relevance and source authority (interviews and detailed commits ranked higher than terse comments), and synthesizes a citation-grounded answer through Claude.

---

## 7. API Endpoints

```
POST   /api/departure/analyze            # Run footprint analysis
POST   /api/departure/generate-interview # Generate exit interview questions
POST   /api/departure/save-answers       # Store and ingest captured answers

POST   /api/onboarding/query             # Onboarding RAG query

GET    /api/knowledge/map                # Knowledge coverage data
GET    /api/knowledge/gaps               # Undocumented modules and bus-factor risks

POST   /api/ingest/refresh               # Trigger re-ingestion of a source
GET    /api/sources/status               # Per-source ingestion state and counts
```

---

## 8. Frontend

React single-page application with three primary views:

- **Departure dashboard** — engineer selector, footprint visualization, interactive interview flow.
- **Onboarding chat** — single-pane chat UI with citations rendered as clickable source pills.
- **Knowledge map** — treemap or heatmap of the codebase, colored by knowledge density (computed from commit verbosity, comment coverage, and interview attribution).

All three views call the FastAPI backend through a thin client layer; no business logic in the frontend.

---

## 9. Key Design Decisions

**Source-aware retrieval.** Storing each source in its own collection — rather than merging into a single index — preserves provenance. A user asking "why does this exist?" gets answers cross-referenced from PR descriptions and exit interviews; a user asking "how is this implemented?" gets answers leaning on code comments. The retriever weights collections differently depending on the question type.

**LangGraph over a chain.** Departure Mode requires multi-step reasoning: analyze footprint, then generate questions conditioned on the analysis, then iterate as the engineer answers. LangGraph's explicit state model handles this more cleanly than a linear LangChain pipeline.

**Local embeddings.** sentence-transformers runs locally rather than calling a paid embedding API. This keeps the cost profile flat (only Claude API calls scale with usage) and removes a network dependency from the ingestion path.

**Closed feedback loop.** Captured exit interview answers become a new source — feeding back into the same retriever that serves the next hire. Each engineer's departure makes the next onboarding faster.

---

## 10. Open Questions

These are unresolved as of the design phase and will be revisited during implementation:

- **Embedding model selection.** sentence-transformers has many model options. The right balance of speed, quality, and code-token handling needs empirical evaluation.
- **Chunking strategy for long PR descriptions.** Section-based chunking works for well-structured PRs; falls back to fixed-window for unstructured ones. May need a hybrid.
- **Knowledge density heuristic.** The Knowledge Map's red-zone calculation needs a concrete formula. Candidate inputs: commit message length, comment-to-code ratio, interview coverage, file-level co-author count.
- **Re-ingestion frequency.** Push (webhook on PR merge) vs pull (scheduled cron) vs hybrid. The right answer probably depends on team size.

---

For demo scenarios and sample queries that exercise this architecture, see [`demo-storyline.md`](demo-storyline.md).
