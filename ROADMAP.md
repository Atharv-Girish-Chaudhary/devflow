# DevFlow Roadmap

## Vision
A multi-agent platform that captures engineering knowledge before it walks out the door, and delivers it on demand to the next person who needs it.

---

## Stage 1 — Design + Public Repo · May 2026

- [x] Hackathon win + product design (March 2026)
- [x] Architecture spec + demo storyline
- [x] Public repo with full design documentation
- [x] Architecture diagram
- [ ] Initial discovery (recruiter outreach, LinkedIn featured section)

## Stage 2 — Ingestion + RAG Core

- [ ] EventPulse demo repo built (fake company backend, 80–120 commits across 3 authors)
- [ ] Git history ingestion pipeline (commits, authors, file-level ownership)
- [ ] PR description ingestion
- [ ] Code comment extraction
- [ ] Embedding pipeline (sentence-transformers) + Chroma vector store
- [ ] First retrieval test: semantic search over commit history

## Stage 3 — Onboarding Assistant

- [ ] Multi-agent retrieval orchestration (LangGraph)
- [ ] Citation-grounded answer synthesis (Anthropic Claude)
- [ ] FastAPI backend with streaming responses
- [ ] React chat UI with source attribution
- [ ] First end-to-end demo: query → grounded multi-source answer

## Stage 4 — Departure Mode

- [ ] Footprint analyzer (git ownership, code authorship density, PR review patterns)
- [ ] Targeted question generation agent (questions only their work history could produce)
- [ ] Exit interview UI with structured Q&A flow
- [ ] Captured answers ingested back into the vector store

## Stage 5 — Knowledge Map

- [ ] Knowledge-density heuristic per file/module (commits + comments + interview coverage)
- [ ] Bus-factor calculation
- [ ] React visualization dashboard with red-zone highlighting

## Stage 6 — Polish

- [ ] Demo video walkthrough
- [ ] Hosted preview deployment
- [ ] README screenshots for each mode
- [ ] Blog post / case study

---

## Sequencing Rationale

Onboarding Assistant ships before Departure Mode despite the lifecycle order being reversed — because the core RAG infrastructure built for Onboarding is reused by both Departure and Knowledge Map. Departure Mode adds the question-generation agent on top of existing retrieval. Knowledge Map is purely a visualization over data the prior stages have populated.

## Scoping Note

This is solo work being done alongside a full-time GenAI internship at Tavant Technologies. Timelines past Stage 1 are intentionally undated — work progresses as bandwidth allows. The repo doubles as portfolio artifact and active build target.
