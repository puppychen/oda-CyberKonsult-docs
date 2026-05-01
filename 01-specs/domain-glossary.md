# Domain Glossary

> Audience: human + AI. This document is the ubiquitous language SSoT for ODA Cyber Konsult. Code, PRD, SRS, and API docs should use these terms consistently.

## Core Terms

| Term | Definition | Context | Aliases to Avoid |
|------|------------|---------|------------------|
| ODA Cyber Konsult | Three-tier AI cybersecurity consultant system for SMB users, IT users, consultants, and data cleaning reviewers. | Product | CyberKonsult, assistant platform |
| Response Mode | Answer depth selected for chat: `beginner`, `standard`, or `expert`. | Chat RAG | tier, persona level |
| Basic User | User role limited to beginner-mode cybersecurity guidance. | Identity & Access | beginner user |
| User | General business user who can use chatbot guidance. | Identity & Access | normal user |
| IT User | Technical user who can access deeper operational guidance. | Identity & Access | MIS user, engineer user |
| Consultant | Cybersecurity consultant role with expert-mode answer needs. | Identity & Access | advisor |
| Data Cleaner | Operator who edits cleaned file content and submits tasks for review. | Cleaning Review | maker, editor |
| Data Reviewer | Operator who reviews, approves, rejects, and ingests cleaned content. | Cleaning Review | checker, approver |
| Admin | System administrator who manages users, prompts, data sources, and operations. | Identity & Access | system manager |
| Conversation | User-owned chat thread containing messages and response mode metadata. | Chat RAG | chat session |
| Message | Single user or assistant entry inside a conversation. | Chat RAG | chat item |
| Source | Evidence item returned by RAG or web search and displayed with an answer. | Chat RAG | citation, reference |
| Confidence Level | Computed answer confidence based on retrieval scores and configured thresholds. | Chat RAG | trust score |
| Prompt Template | Role/mode-specific prompt content with variables rendered before LLM generation. | Prompt Management | prompt |
| RAG Retrieve | FastAPI retrieval operation that queries indexed knowledge and returns scored chunks. | RAG Service | search |
| Web Search Fallback | SearXNG-backed enrichment used when knowledge-base retrieval is insufficient. | Chat RAG | online search |
| Golden Test | Curated question and expected evidence set used to evaluate RAG answer quality. | RAG Evaluation | benchmark QA |
| Uploaded File | Source file received for cleaning or ingestion. | Cleaning Pipeline | document |
| Cleaning Task | Batch processing unit for uploaded files, cleaning status, review status, and ingestion state. | Cleaning Pipeline | job |
| Task File | File-level item within a cleaning task, including detected entities, edited content, and review status. | Cleaning Pipeline | task item |
| PII Entity | Personally identifiable information detected in uploaded content. | Data Pipeline | sensitive entity |
| De-identification Strategy | Processing action such as mask, partial mask, pseudonymize, generalize, keep labeled, or encrypt. | Data Pipeline | anonymization rule |
| Maker-Checker | Separation-of-duty review model: submitter must differ from reviewer/approver/ingester. | Cleaning Review | four-eyes review |
| Review Requested | Approval state meaning a cleaner has submitted a task for reviewer action. | Cleaning Review | submitted |
| Approved | Approval state meaning cleaned content passed review and may be ingested. | Cleaning Review | accepted |
| Ingested | Approval state meaning approved content has been sent into the RAG knowledge base. | Cleaning Review | imported |
| Internal Token | Shared service credential carried by NestJS to FastAPI in `X-Internal-Token`. | Service Boundary | service token |
| Audit Log | Persisted operational record for security-relevant or administrative action. | Audit | operation log |
| Contract Constant | Cross-language or cross-module value stored in `contracts/*.yml`. | SSoT | shared config |

## Subdomain Classification

| Module / Area | Subdomain Type | Rhythm Override | Rationale |
|---------------|----------------|-----------------|-----------|
| Chat RAG | Core | Careful | Product differentiator; crosses NestJS, FastAPI, Qdrant, LLM, SearXNG; history includes IDOR and threshold drift. |
| Cleaning Pipeline | Core | Careful | PII processing and data quality directly affect legal and knowledge-base safety. |
| Cleaning Review / Maker-Checker | Core | Careful | Multi-role state machine with separation-of-duty invariant. |
| Identity & Access | Generic | Careful | Generic capability, but security-critical and shared by all applications. |
| Prompt Management | Supporting | Standard | Enables role/mode quality but is mostly CRUD plus renderer invariants. |
| Audit | Supporting | Careful | Required for traceability and security review. |
| Data Sources | Supporting | Standard | Google Drive ingestion supports knowledge freshness. |
| Web Search | Supporting | Standard | Search fallback improves answer coverage but is bounded by chat orchestration. |
| Observability / Operations | Supporting | Careful | Required before QG-5 production delivery. |
| Admin / Cleaner / Chatbot UI | Supporting | Standard | Workflow surfaces for the core domain; security-sensitive screens inherit Careful checks. |

## Naming Rules

- Use `Conversation` only for chat history owned by a user; do not call it `session` in code or docs unless referring to runtime auth/session.
- Use `Cleaning Task` for the batch-level workflow and `Task File` for file-level review state.
- Use `Maker-Checker` for separation of duties; do not use only `approval` when the submitter/reviewer invariant matters.
- Use `Response Mode` for beginner/standard/expert answer depth; do not call it `role`, because role is RBAC.
- Use `Contract Constant` for values in `contracts/*.yml`; do not call them environment config.
