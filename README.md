# ODA Cyber Konsult - 文件總覽

三層式 AI 資安顧問系統，基於 RAG 技術搭配在地化法規知識庫，服務中小企業。

本文件以 **SDLC 五階段**組織全部文件，便於快速定位。

```
Phase 1          Phase 2          Phase 3          Phase 4          Phase 5
Requirements  →  Design        →  API & Impl   →  Testing       →  Operations
 specs/          architecture/    api/             testing/         setup/
                                  implementation/                   operations/
```

---

## Phase 1：需求分析（Requirements）

定義系統做什麼、為誰做、驗收標準是什麼。

| 文件 | 說明 | 類型 |
|------|------|------|
| [specs/PRD.md](./specs/PRD.md) | 產品需求文件 — 41 個 User Story + 85 AC | US/AC 格式 |
| [specs/RTM.md](./specs/RTM.md) | 需求追溯矩陣 — US→FR→API→模組→測試 | 追溯矩陣 |
| [specs/SRS_BUSINESS.md](./specs/SRS_BUSINESS.md) | 商業需求規格書 — 使用情境、角色、KPI | 業務需求 |
| [specs/SRS_TECHNICAL.md](./specs/SRS_TECHNICAL.md) | 技術需求規格書 — FR-01~21、驗收標準 §11 | 技術需求 |
| [SRS_BUSINESS.docx](./SRS_BUSINESS.docx) | 商業需求 Word 原始版本 | 原始文件 |

**文件鏈**：SRS_BUSINESS → SRS_TECHNICAL → PRD（US/AC 萃取） → RTM（追溯串連）

---

## Phase 2：系統設計（Design）

定義系統怎麼做、架構決策、技術選型。

| 文件 | 說明 | 類型 |
|------|------|------|
| [architecture/ARCH.md](./architecture/ARCH.md) | 架構決策紀錄 — 10 個 ADR（Monorepo/Gateway/Hybrid Search...） | ADR |
| [architecture/system-overview.md](./architecture/system-overview.md) | 系統架構圖、服務拓撲、通訊矩陣 | 架構概覽 |
| [architecture/rag-system-blueprint.md](./architecture/rag-system-blueprint.md) | RAG 系統 9 階段建構指南 | 設計藍圖 |
| [architecture/rag-pipeline.md](./architecture/rag-pipeline.md) | RAG 管線流程（載入→切塊→索引→檢索） | 管線設計 |
| [architecture/rag-enhancements.md](./architecture/rag-enhancements.md) | RAG 品質優化策略（Query 改寫、分數過濾、Reranking） | 優化設計 |
| [architecture/data-cleaning-workflow.md](./architecture/data-cleaning-workflow.md) | 資料去識別化工作流程設計 | 流程設計 |

**文件鏈**：system-overview（全局） → ARCH（決策） → rag-*/data-cleaning-*（各子系統設計）

---

## Phase 3：API 規格與實作（API & Implementation）

定義端點契約、模組架構、前端實作細節。

### API 文件

| 文件 | 說明 | 路徑前綴 |
|------|------|----------|
| [api/README.md](./api/README.md) | API 總索引 — 認證規範、流程圖、通用格式 | — |
| [api/auth-api.md](./api/auth-api.md) | 認證與授權（登入/註冊/Token/密碼變更） | `/api/auth` |
| [api/chat-api.md](./api/chat-api.md) | 聊天與對話管理（SSE/模式切換） | `/api/chat` |
| [api/audit-users-api.md](./api/audit-users-api.md) | 使用者管理 + 稽核日誌 | `/api/users`, `/api/audit-logs` |
| [api/prompts-api.md](./api/prompts-api.md) | 提示詞範本 CRUD + 測試 | `/api/prompts` |
| [api/health-api.md](./api/health-api.md) | 健康檢查（DB/RAG/Kubernetes 探針） | `/health` |
| [api/cleaning-proxy.md](./api/cleaning-proxy.md) | NestJS 清洗代理層設計 | `/api/v1/*` |
| [api/cleaning-api.md](./api/cleaning-api.md) | FastAPI 去識別化處理 | `/api/v1/*` |
| [api/review-api.md](./api/review-api.md) | 審核工作流（審核/標籤/批准/送入 RAG） | `/api/v1/review` |
| [api/analytics-api.md](./api/analytics-api.md) | 資料分析統計（清洗/知識庫/時間軸） | `/api/v1/analytics` |
| [api/knowledge-base-api.md](./api/knowledge-base-api.md) | 知識庫文件瀏覽與管理 | `/api/v1/knowledge-base` |
| [api/rag-api.md](./api/rag-api.md) | RAG 查詢與文件管理（Retrieve/Ingest） | `/api/v1/rag` |
| [api/websearch-api.md](./api/websearch-api.md) | 網路搜尋設定與整合 | `/api/websearch` |
| [api/gdrive-api.md](./api/gdrive-api.md) | Google Drive 資料來源 | `/api/datasources/gdrive` |
| [api/data-pipeline-loaders-api.md](./api/data-pipeline-loaders-api.md) | 文件解析器 API（PDF/DOCX/XLSX/PPTX） | `data_pipeline.loaders` |

### 實作說明

| 文件 | 說明 | 層級 |
|------|------|------|
| [implementation/nestjs-modules.md](./implementation/nestjs-modules.md) | NestJS 模組架構總覽 | 後端 |
| [implementation/chat-module.md](./implementation/chat-module.md) | 聊天模組實作細節（SSE/RAG 編排） | 後端 |
| [implementation/chatbot-frontend.md](./implementation/chatbot-frontend.md) | Chatbot UI 前端實作 | 前端 |

---

## Phase 4：測試（Testing）

定義測試策略、執行方式、涵蓋率目標。

| 文件 | 說明 | 類型 |
|------|------|------|
| [testing/test-strategy.md](./testing/test-strategy.md) | 測試策略 — 金字塔、工具鏈、涵蓋率、安全檢核 | 策略文件 |
| [testing/test-guide.md](./testing/test-guide.md) | 測試指南 — 啟動方式、手動測試案例、角色權限矩陣 | 操作指南 |
| [testing/api-e2e-testing.md](./testing/api-e2e-testing.md) | E2E 測試架構 — Mock 策略、測試範圍 | 技術文件 |

**文件鏈**：test-strategy（策略與目標） → test-guide（手動執行） → api-e2e-testing（自動化 E2E）

---

## Phase 5：部署與運維（Operations）

定義環境建置、部署流程、故障排除。

### 環境建置

| 文件 | 說明 | 對象 |
|------|------|------|
| [setup/getting-started.md](./setup/getting-started.md) | 快速開始（5 分鐘上手） | 新成員 |
| [setup/development.md](./setup/development.md) | 開發環境完整說明 | 開發者 |
| [setup/deployment.md](./setup/deployment.md) | 部署與維運指南（Docker/GCP/VM） | DevOps |
| [setup/auth-deps.md](./setup/auth-deps.md) | 認證模組依賴安裝 | 開發者 |
| [setup/cleaning-proxy-setup.md](./setup/cleaning-proxy-setup.md) | 清洗代理層設定 | 開發者 |
| [setup/gdrive-setup.md](./setup/gdrive-setup.md) | Google Drive 整合設定 | 管理員 |
| [setup/linting-setup.md](./setup/linting-setup.md) | ESLint + Prettier + Husky 設定 | 開發者 |

### 運維手冊

| 文件 | 說明 | 對象 |
|------|------|------|
| [operations/runbook.md](./operations/runbook.md) | 運維手冊 — 健康檢查、故障排除、備份還原、安全事件應變 | 運維/SRE |

---

## 操作指南（跨階段）

| 文件 | 說明 | 適用對象 |
|------|------|---------|
| [guides/chat-quick-start.md](./guides/chat-quick-start.md) | 聊天功能快速上手 | 使用者 |
| [guides/rag-cookbook.md](./guides/rag-cookbook.md) | RAG + 清洗整合食譜 | 開發者 |

---

## 文件統計

| 目錄 | 檔案數 | SDLC 階段 |
|------|--------|----------|
| specs/ | 4 (+1 docx) | Phase 1 |
| architecture/ | 6 | Phase 2 |
| api/ | 15 | Phase 3 |
| implementation/ | 3 | Phase 3 |
| testing/ | 3 | Phase 4 |
| setup/ | 7 | Phase 5 |
| operations/ | 1 | Phase 5 |
| guides/ | 2 | 跨階段 |
| **合計** | **42** (.md) | — |

---

## 快速導引

- **我是新成員** → [setup/getting-started.md](./setup/getting-started.md) → [architecture/system-overview.md](./architecture/system-overview.md)
- **我要了解需求** → [specs/PRD.md](./specs/PRD.md) → [specs/RTM.md](./specs/RTM.md)
- **我要查 API** → [api/README.md](./api/README.md)
- **我要跑測試** → [testing/test-guide.md](./testing/test-guide.md)
- **系統出問題** → [operations/runbook.md](./operations/runbook.md)
- **我要審查架構決策** → [architecture/ARCH.md](./architecture/ARCH.md)
