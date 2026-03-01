# ODA Cyber Konsult - 文件導覽

三層式 AI 資安顧問系統（新手/一般/顧問模式），基於 RAG 技術搭配在地化法規知識庫，服務中小企業。

本文件庫以 **AI-Native SDLC 三迴圈**組織，對應系統生命週期各階段。

```
┌──────────────────────────────────────────────────────────────┐
│                    AI-Native SDLC 迴圈                        │
│                                                              │
│   Outer Loop          Middle Loop          Delivery Loop     │
│   ┌──────────┐       ┌──────────┐        ┌──────────┐       │
│   │ 01-specs  │ ───→  │02-testing│  ───→  │03-operations│    │
│   │ 規格與設計 │       │ 測試驗證  │        │ 運維與部署   │    │
│   └──────────┘       └──────────┘        └──────────┘       │
│        ↑                                       │             │
│        └───────────── 回饋迭代 ─────────────────┘             │
└──────────────────────────────────────────────────────────────┘
```

---

## 目錄結構

### 01-specs/ — 規格與設計（Outer Loop）

需求定義、架構決策、API 規格、實作說明。

| 文件 | 說明 | 更新日期 |
|------|------|---------|
| [PRD.md](./01-specs/PRD.md) | 產品需求文件 — 41 個 User Story + 85 AC | 2026-03-01 |
| [SRS_BUSINESS.md](./01-specs/SRS_BUSINESS.md) | 商業需求規格書 — 使用情境、角色、KPI | 2026-02-17 |
| [SRS_TECHNICAL.md](./01-specs/SRS_TECHNICAL.md) | 技術需求規格書 — FR-01~21、驗收標準 | 2026-02-13 |
| [RTM.md](./01-specs/RTM.md) | 需求追溯矩陣 — US → FR → API → 模組 → 測試 | 2026-03-01 |
| [threat-model.md](./01-specs/threat-model.md) | 威脅模型 — STRIDE 分析、攻擊面、風險矩陣 | 2026-03-01 |
| [architecture/](./01-specs/architecture/) | 架構文件 — ADR、系統架構、RAG 藍圖 | 2026-03-01 |
| [api/](./01-specs/api/) | API 規格文件 — 15 份端點規格 | 2026-02-24 |
| [implementation/](./01-specs/implementation/) | 實作說明 — NestJS 模組、聊天模組、前端 | 2026-02-09 |
| [guides/](./01-specs/guides/) | 操作指南 — 聊天快速上手、RAG Cookbook | 2026-02-13 |

### 02-testing/ — 測試文件（Middle Loop — QG-4）

測試策略、測試指南、E2E 測試架構。

| 文件 | 說明 | 更新日期 |
|------|------|---------|
| [test-strategy.md](./02-testing/test-strategy.md) | 測試策略 — 金字塔、工具鏈、涵蓋率、安全檢核 | 2026-03-01 |
| [test-guide.md](./02-testing/test-guide.md) | 測試指南 — 啟動方式、手動測試案例、角色權限矩陣 | 2026-02-24 |
| [api-e2e-testing.md](./02-testing/api-e2e-testing.md) | E2E 測試 — Mock 策略、測試範圍、整合架構 | 2026-02-11 |

### 03-operations/ — 運維與部署文件（Delivery Loop — QG-5）

環境建置、部署指南、故障排除、運維手冊。

| 文件 | 說明 | 更新日期 |
|------|------|---------|
| [runbook.md](./03-operations/runbook.md) | 運維手冊 — 健康檢查、故障排除、備份還原、安全事件應變 | 2026-03-01 |
| [getting-started.md](./03-operations/getting-started.md) | 快速開始 — 5 分鐘上手 | 2026-02-04 |
| [development.md](./03-operations/development.md) | 開發環境完整說明 | 2026-02-11 |
| [deployment.md](./03-operations/deployment.md) | 部署與維運指南（Docker/GCP/VM） | 2026-02-11 |
| [auth-deps.md](./03-operations/auth-deps.md) | 認證模組依賴安裝 | 2026-02-09 |
| [cleaning-proxy-setup.md](./03-operations/cleaning-proxy-setup.md) | 清洗代理層設定 | 2026-02-09 |
| [gdrive-setup.md](./03-operations/gdrive-setup.md) | Google Drive 整合設定 | 2026-02-13 |
| [linting-setup.md](./03-operations/linting-setup.md) | ESLint + Prettier + Husky 設定 | 2026-02-11 |

---

## Quick Start

| 我想要... | 前往 |
|----------|------|
| 快速建立開發環境 | [getting-started.md](./03-operations/getting-started.md) |
| 了解系統架構 | [system-overview.md](./01-specs/architecture/system-overview.md) |
| 查閱產品需求 | [PRD.md](./01-specs/PRD.md) → [RTM.md](./01-specs/RTM.md) |
| 查詢 API 規格 | [api/README.md](./01-specs/api/README.md) |
| 執行測試 | [test-guide.md](./02-testing/test-guide.md) |
| 排除系統故障 | [runbook.md](./03-operations/runbook.md) |
| 審查架構決策 | [ARCH.md](./01-specs/architecture/ARCH.md) |
| 了解安全威脅 | [threat-model.md](./01-specs/threat-model.md) |

---

## 文件統計

| 目錄 | 檔案數 | SDLC 迴圈 |
|------|--------|----------|
| 01-specs/ | 6 + architecture(6) + api(15) + implementation(3) + guides(2) | Outer Loop |
| 02-testing/ | 3 | Middle Loop |
| 03-operations/ | 8 | Delivery Loop |
| **合計** | **43** (.md) | — |
