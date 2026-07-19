---
audience: both
purpose: reference
status: approved
owner: ODA Cyber Konsult
---

# 測試文件（Middle Loop — QG-4）

> ODA Cyber Konsult — 測試策略、測試指南、E2E 測試架構

本目錄涵蓋系統品質驗證的所有文件，對應 AI-Native SDLC 的 Middle Loop 階段（Quality Gate 4）。

---

## 文件清單

| 文件 | 說明 | 更新日期 |
|------|------|---------|
| [test-strategy.md](./test-strategy.md) | 測試策略 — 測試金字塔、工具鏈（Jest/pytest）、涵蓋率目標、安全性檢核對照表 | 2026-03-01 |
| [test-guide.md](./test-guide.md) | 測試指南 — 服務啟動、來源狀態與 PostgreSQL rollback 整合測試、角色權限矩陣 | 2026-07-17 |
| [api-e2e-testing.md](./api-e2e-testing.md) | E2E 測試 — NestJS E2E 架構、Mock 策略（Prisma/RAG/LLM）、測試範圍 | 2026-02-11 |
| [nfr-baseline-plan.md](./nfr-baseline-plan.md) | NFR 基準測試規劃 — P95 latency / RAG 召回率 / PII 偵測準確率（k6 壓測待跑）| 2026-04-20 |

---

## 文件鏈

```
test-strategy（策略與目標）
       ↓
test-guide（手動執行步驟）
       ↓
api-e2e-testing（自動化 E2E 實作）
```

---

## 相關文件

| 文件 | 位置 | 說明 |
|------|------|------|
| SRS_TECHNICAL.md §11 | [../01-specs/SRS_TECHNICAL.md](../01-specs/SRS_TECHNICAL.md) | 驗收標準（功能/效能/安全） |
| RTM.md §3 | [../01-specs/RTM.md](../01-specs/RTM.md) | 驗收測試追溯 |
| PRD.md | [../01-specs/PRD.md](../01-specs/PRD.md) | User Story 與 AC |
| ARCH.md ADR-011 | [../01-specs/architecture/ARCH.md](../01-specs/architecture/ARCH.md) | Contracts as Code 跨層常數 SSoT — 含反漂移驗證機制（contract test pattern）|

---

## 本批新增驗證工具（2026-04-29）

| 工具 | 用途 | 參考 |
|------|------|------|
| `apps/api/src/modules/chat/services/chat.service.contract.spec.ts` | Anti-drift 行為測試（jest.mock 注入極端值，驗證 service 真的從 contracts 載入而非硬編碼）| ADR-011 反漂移驗證機制 §1 |
| `python/tests/test_contract_parity.py` | 跨語言 schema parity（TS/Python 載入結果 deep-equal）| ADR-011 §2 |
| `scripts/docs-lint.mjs` | 文件 lint（broken-link 檢查 + Tier 1 long-line copy 偵測）| `pnpm lint:docs` |
| `scripts/mypy-pr-diff.sh` | QG-3 mypy patch-strict（PR diff 觸及的 Python 模組嚴格型別檢查）| QG-3 三道防線 §2 |
