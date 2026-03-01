# 測試文件（Middle Loop — QG-4）

> ODA Cyber Konsult — 測試策略、測試指南、E2E 測試架構

本目錄涵蓋系統品質驗證的所有文件，對應 AI-Native SDLC 的 Middle Loop 階段（Quality Gate 4）。

---

## 文件清單

| 文件 | 說明 | 更新日期 |
|------|------|---------|
| [test-strategy.md](./test-strategy.md) | 測試策略 — 測試金字塔、工具鏈（Jest/pytest）、涵蓋率目標、安全性檢核對照表 | 2026-03-01 |
| [test-guide.md](./test-guide.md) | 測試指南 — 服務啟動步驟、手動測試案例、角色權限矩陣、預設帳號 | 2026-02-24 |
| [api-e2e-testing.md](./api-e2e-testing.md) | E2E 測試 — NestJS E2E 架構、Mock 策略（Prisma/RAG/LLM）、測試範圍 | 2026-02-11 |

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
