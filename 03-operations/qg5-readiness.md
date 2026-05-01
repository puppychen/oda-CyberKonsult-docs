# QG-5 Readiness

> Audience: human + AI. Delivery Loop checklist for deciding whether ODA Cyber Konsult is ready for deployment handoff.

## Current Decision

| Gate | Status | Decision |
|------|--------|----------|
| QG-5 Deployment → Delivery | Not passed | Local development and functional validation are documented, but production delivery still needs CI/CD, deployment diagram promotion, observability, and final security/rollback evidence. |

## Checklist

| QG-5 Item | Current Evidence | Status | Gap / Next Action |
|-----------|------------------|--------|-------------------|
| Deploy config reviewed | [`deployment.md`](./deployment.md), [`staging-setup.md`](./staging-setup.md) | Partial | Confirm production resource limits, secrets handling, and rollback command path. |
| Monitoring + alerting configured | [`runbook.md`](./runbook.md) health checks | Partial | Add SLI/SLO, alert thresholds, dashboards, and runbook-linked alerts. |
| Security hardening verified | [`../01-specs/threat-model.md`](../01-specs/threat-model.md), `pnpm security:all` script | Partial | Record dependency scan outputs and production header/TLS/secrets verification. |
| Runbook covers startup/errors/rollback/scaling | [`runbook.md`](./runbook.md) | Partial | Add explicit rollback and scaling decision table. |
| Deployment diagram current | [`../02-design/diagrams/README.md`](../02-design/diagrams/README.md) points to existing architecture diagrams | Partial | Promote or create `../02-design/diagrams/deployment.drawio` before QG-5 pass. |
| Knowledge transfer complete | Existing docs index and guides | Pending | Prepare handoff checklist and owner matrix. |
| RTM integrity checked | [`../01-specs/RTM.md`](../01-specs/RTM.md) | Partial | Re-run RTM gap/orphan check after CI evidence is available. |

## Required Evidence Before PASS

| Evidence | Storage Location |
|----------|------------------|
| CI/CD workflow status and commands | `.github/workflows/` plus this document |
| Latest full test run summary | `../02-testing/test-guide.md` or a dated test report |
| Dependency scan summary | `../03-operations/` dated security evidence |
| Backup restore verification | [`backup-strategy.md`](./backup-strategy.md), [`dr-checklist.md`](./dr-checklist.md) |
| SLO/alert configuration | Future `observability.md` or runbook section |
| Deployment/rollback diagram | `../02-design/diagrams/deployment.drawio` |

## Current Non-Blocking Strengths

- Dev Environment and service registry are captured in `PROJECT_CONTEXT.md`.
- Runbook, deployment, backup, DR checklist, and staging setup documents exist.
- Health endpoints and operational scripts exist for local verification.
- Spec Pack and contracts lint gates are available for SDD-specific drift prevention.

## Blocking Gaps

1. CI/CD automation is not yet present in the workspace.
2. Observability is health-check oriented; SLO/SLI and alerting evidence are not yet documented.
3. Deployment diagram is indexed but not yet promoted as `docs/02-design/diagrams/deployment.drawio`.
4. QG-5 evidence is not yet tied to a dated full verification run.
