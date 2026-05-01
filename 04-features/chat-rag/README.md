# chat-rag Spec Pack

> Audience: AI-primary. Feature-level working memory for Chat RAG changes.

This spec pack exists because Chat RAG crosses NestJS, FastAPI, Qdrant, SearXNG, LLM providers, contracts, and user-owned conversation data. It is security-sensitive and has historical incidents around IDOR, prompt rendering, and threshold drift.

| File | Purpose | Update Trigger |
|------|---------|----------------|
| [`requirements.md`](./requirements.md) | Pointers to PRD AC plus chat-rag-specific authorization and threshold rules | PRD AC, security rule, or pitfall changes |
| [`design.md`](./design.md) | Feature call chain, key decisions, trade-offs, and external dependencies | Architecture or dependency changes |
| [`tasks.md`](./tasks.md) | Completed/pending/future work and maintenance discipline | Every chat-rag fix or feature commit |

## Authoritative Upstream

| Source | Purpose |
|--------|---------|
| [`../../01-specs/PRD.md`](../../01-specs/PRD.md) | User stories and acceptance criteria |
| [`../../01-specs/SRS_TECHNICAL.md`](../../01-specs/SRS_TECHNICAL.md) | Functional and non-functional requirements |
| [`../../01-specs/RTM.md`](../../01-specs/RTM.md) | Traceability to implementation and tests |
| [`../../../contracts/README.md`](../../../contracts/README.md) | RAG thresholds and user role SSoT boundary |

## D0 Red Line

Do not copy PRD/SRS/RTM long sections into this directory. Use pointers plus feature-specific rules, design decisions, and pitfall context only.
