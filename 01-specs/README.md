# 規格與設計文件（Outer Loop）

> ODA Cyber Konsult — 需求定義、架構設計、API 規格、實作說明

本目錄涵蓋系統「做什麼」與「怎麼做」的所有規格文件，對應 AI-Native SDLC 的 Outer Loop 階段。

---

## 文件清單

### 需求與追溯

| 文件 | 說明 | 更新日期 |
|------|------|---------|
| [PRD.md](./PRD.md) | 產品需求文件 — 41 個 User Story、85 個驗收條件、優先級與角色矩陣 | 2026-03-01 |
| [SRS_BUSINESS.md](./SRS_BUSINESS.md) | 商業需求規格書 — 使用情境、利害關係人、KPI 指標 | 2026-02-17 |
| [SRS_TECHNICAL.md](./SRS_TECHNICAL.md) | 技術需求規格書 — FR-01~21 功能需求、§11 驗收標準（功能/效能/安全） | 2026-02-13 |
| [RTM.md](./RTM.md) | 需求追溯矩陣 — User Story → FR → API 端點 → 模組 → 測試的完整追溯鏈 | 2026-03-01 |
| [threat-model.md](./threat-model.md) | 威脅模型 — STRIDE 分析、7 個攻擊面、風險矩陣、22 項緩解措施追蹤 | 2026-03-01 |

### 架構設計（architecture/）

| 文件 | 說明 | 更新日期 |
|------|------|---------|
| [ARCH.md](./architecture/ARCH.md) | 架構決策紀錄 — 10 個 ADR（Monorepo、Gateway、Hybrid Search 等） | 2026-03-01 |
| [system-overview.md](./architecture/system-overview.md) | 系統架構圖、服務拓撲、通訊矩陣 | 2026-02-17 |
| [rag-system-blueprint.md](./architecture/rag-system-blueprint.md) | RAG 系統 9 階段建構指南 | 2026-02-24 |
| [rag-pipeline.md](./architecture/rag-pipeline.md) | RAG 管線流程 — 載入 → 切塊 → 索引 → 檢索 | 2026-02-24 |
| [rag-enhancements.md](./architecture/rag-enhancements.md) | RAG 品質優化策略 — Query 改寫、分數過濾、Reranking | 2026-02-24 |
| [data-cleaning-workflow.md](./architecture/data-cleaning-workflow.md) | 資料去識別化工作流程設計 | 2026-02-12 |

### API 規格（api/）

| 文件 | 說明 | 路徑前綴 | 更新日期 |
|------|------|---------|---------|
| [README.md](./api/README.md) | API 總索引 — 認證規範、流程圖、通用格式 | — | 2026-02-24 |
| [auth-api.md](./api/auth-api.md) | 認證與授權（登入/註冊/Token/密碼變更） | `/api/auth` | 2026-02-24 |
| [chat-api.md](./api/chat-api.md) | 聊天與對話管理（SSE/模式切換） | `/api/chat` | 2026-02-24 |
| [audit-users-api.md](./api/audit-users-api.md) | 使用者管理 + 稽核日誌 | `/api/users`, `/api/audit-logs` | 2026-02-09 |
| [prompts-api.md](./api/prompts-api.md) | 提示詞範本 CRUD + 測試 | `/api/prompts` | 2026-02-11 |
| [health-api.md](./api/health-api.md) | 健康檢查（DB/RAG/Kubernetes 探針） | `/health` | 2026-02-11 |
| [cleaning-proxy.md](./api/cleaning-proxy.md) | NestJS 清洗代理層設計 | `/api/v1/*` | 2026-02-09 |
| [cleaning-api.md](./api/cleaning-api.md) | FastAPI 去識別化處理 | `/api/v1/*` | 2026-02-23 |
| [review-api.md](./api/review-api.md) | 審核工作流（審核/標籤/批准/送入 RAG） | `/api/v1/review` | 2026-02-11 |
| [analytics-api.md](./api/analytics-api.md) | 資料分析統計（清洗/知識庫/時間軸） | `/api/v1/analytics` | 2026-02-11 |
| [knowledge-base-api.md](./api/knowledge-base-api.md) | 知識庫文件瀏覽與管理 | `/api/v1/knowledge-base` | 2026-02-12 |
| [rag-api.md](./api/rag-api.md) | RAG 查詢與文件管理（Retrieve/Ingest） | `/api/v1/rag` | 2026-02-24 |
| [websearch-api.md](./api/websearch-api.md) | 網路搜尋設定與整合 | `/api/websearch` | 2026-02-13 |
| [gdrive-api.md](./api/gdrive-api.md) | Google Drive 資料來源 | `/api/datasources/gdrive` | 2026-02-11 |
| [data-pipeline-loaders-api.md](./api/data-pipeline-loaders-api.md) | 文件解析器 API（PDF/DOCX/XLSX/PPTX） | `data_pipeline.loaders` | 2026-02-11 |

### 實作說明（implementation/）

| 文件 | 說明 | 更新日期 |
|------|------|---------|
| [nestjs-modules.md](./implementation/nestjs-modules.md) | NestJS 模組架構總覽 — 分層設計、依賴注入 | 2026-02-09 |
| [chat-module.md](./implementation/chat-module.md) | 聊天模組實作細節 — SSE 串流、RAG 編排、多輪對話 | 2026-02-09 |
| [chatbot-frontend.md](./implementation/chatbot-frontend.md) | Chatbot UI 前端實作 — 元件結構、狀態管理 | 2026-02-09 |

### 操作指南（guides/）

| 文件 | 說明 | 更新日期 |
|------|------|---------|
| [chat-quick-start.md](./guides/chat-quick-start.md) | 聊天功能快速上手 — 使用者導向操作手冊 | 2026-02-09 |
| [rag-cookbook.md](./guides/rag-cookbook.md) | RAG + 清洗整合食譜 — 開發者導向實戰指引 | 2026-02-13 |

---

## 文件鏈

```
SRS_BUSINESS → SRS_TECHNICAL → PRD（US/AC 萃取）→ RTM（追溯串連）
                    ↓                                    ↓
             threat-model（安全分析）              ARCH（架構決策）
                                                    ↓
                                        rag-*/data-cleaning-*（子系統設計）
                                                    ↓
                                              api/（端點規格）
                                                    ↓
                                         implementation/（實作說明）
```
