# AI-Native SDLC Alignment Review - 2026-04-29

> Audience: human + AI. Workspace alignment review against the current `process-ai-native-sdlc` skill.

## Scope

This review checks whether the current project workspace matches the AI-Native SDLC structure across documentation, code organization, tests, contracts, scripts, and delivery gates.

## Summary

| Area | Alignment | Evidence | Required Adjustment |
|------|-----------|----------|---------------------|
| Rhythm | Aligned | `PROJECT_CONTEXT.md` declares `Rhythm = Careful` | None |
| Outer Loop docs | Mostly aligned | PRD, SRS, ARCH, RTM, threat model exist | Added domain glossary and context map |
| Design docs | Partially aligned | `docs/02-design/README.md` existed | Added design subdirectory indexes |
| Testing docs | Aligned | `docs/02-testing/` contains strategy, guide, E2E, NFR baseline | Keep RTM/test sync current |
| Delivery docs | Partially aligned | Runbook, deployment, backup, DR, staging docs exist | Added QG-5 readiness checklist |
| Tier 2A contracts | Aligned | `contracts/*.yml`, TS package, Python loader, parity test exist | Add CI enforcement later |
| Tier 2B spec pack | Aligned for pilot | `docs/04-features/chat-rag/` has three-file pack | Added feature README |
| Code workspace | Aligned | NestJS modules, React apps, Python uv workspace, shared packages present | No code changes required |
| QG-3 defenses | Partially aligned | `scripts/mypy-pr-diff.sh`, spec-pack lint, eslint/security scripts exist | Add mypy dependency + CI wiring before relying on mypy gate |
| QG-5 | Not passed | Local operations docs exist; CI/CD and observability evidence missing | Track in `docs/03-operations/qg5-readiness.md` |

## Code Workspace Observations

| Layer | Current Structure | Alignment |
|-------|-------------------|-----------|
| Monorepo | `apps/*`, `packages/*`, pnpm workspace | Matches ARCH ADR-001 |
| NestJS API | `apps/api/src/modules/*` with controller/service/repository pattern | Matches documented backend layering |
| React apps | `apps/admin`, `apps/chatbot`, `apps/cleaner` | Matches three UI surfaces in SRS/PROJECT_CONTEXT |
| Python services | `python/rag-service`, `python/data-pipeline`, `python/shared` | Matches RAG/cleaning split |
| Contracts | `contracts/`, `packages/contracts`, `python/shared/contracts.py` | Matches Tier 2A SSoT |
| Tests | Jest/Vitest/Pytest test locations across apps and python workspace | Matches QG-4 structure; latest run was not re-executed in this review |

## Document Structure Adjustments Applied

| Gap | Adjustment |
|-----|------------|
| Missing domain modeling deliverables for Careful rhythm | Added `domain-glossary.md` and `context-map.md` |
| Existing docs subdirectories without README indexes | Added README files for architecture, guides, implementation, and chat-rag |
| `02-design` had only a root index | Added diagrams, flows, wireframes, and prototype indexes |
| QG-5 state was scattered across runbook/deployment/audit notes | Added `03-operations/qg5-readiness.md` |
| Root docs index still reflected older three-folder model | Updated `docs/README.md` to include 02-design, 04-features, contracts, and AI-Native SDLC tiers |
| Scripts README did not include SDD gates | Added `spec-pack-lint.mjs` and `mypy-pr-diff.sh` entries |

## Residual Risks

- QG-5 should remain pending until CI/CD, observability, security scan evidence, and rollback evidence are captured.
- `mypy-pr-diff.sh` is a gate wrapper; it still requires `mypy` to be declared and installed in the Python uv workspace before it can validate changed Python files.
- `docs/04-features/chat-rag/` remains a D5 pilot until the 2026-05-06 verification decision.
