# 需求追溯矩陣 (RTM)

> **ODA Cyber Konsult - 資安助手 RAG 系統**
>
> 文件版本：1.0.0
> 建立日期：2026-03-01
> 文件類型：需求追溯矩陣（Requirements Traceability Matrix）

---

## 文件目的

本文件建立從 User Story 到測試案例的完整追溯鏈路，確保每個需求都有對應的實作與驗證。

### 追溯鏈路

```
User Story (PRD.md)
  → 功能需求 (SRS_TECHNICAL.md FR-XX)
    → API 端點 (docs/api/)
      → 實作模組 (NestJS / Python)
        → 測試案例 (Unit / Integration / E2E)
          → 驗收測試 (SRS_TECHNICAL.md §11 TC-XX)
```

### 狀態圖例

| 圖示 | 意義 | 說明 |
|------|------|------|
| ✅ | 已實作 | 功能完成且有測試覆蓋 |
| 🔄 | 規劃中 | Phase 2 排程實作 |
| 🔮 | 未來願景 | Phase 3 以後 |

### 文件引用

| 縮寫 | 全名 | 路徑 |
|------|------|------|
| PRD | 產品需求文件 | [specs/PRD.md](./PRD.md) |
| SRS-B | 業務需求規格 | [specs/SRS_BUSINESS.md](./SRS_BUSINESS.md) |
| SRS-T | 技術需求規格 | [specs/SRS_TECHNICAL.md](./SRS_TECHNICAL.md) |

---

## 1. 追溯矩陣 — 已實作功能

### FR-01：使用者認證與授權 🔄

| US | 說明 | API 端點 | NestJS 模組 | NestJS 測試 | Python 測試 | TC | 狀態 |
|----|------|----------|------------|-------------|-------------|-----|------|
| US-01-01 | 帳號密碼登入 | `POST /api/auth/login` | auth/ | auth.service.spec, token.service.spec, login.dto.spec, password-strength.validator.spec, password-change-required.guard.spec | — | TC-01-001, TC-01-002 | ✅ |
| US-01-01 | 註冊 | `POST /api/auth/register` | auth/ | auth.service.spec, register.dto.spec | — | — | ✅ |
| US-01-01 | Token 刷新 | `POST /api/auth/refresh` | auth/ | auth.service.spec, refresh.dto.spec | — | — | ✅ |
| US-01-01 | 密碼變更 | `POST /api/auth/change-password` | auth/ | auth.service.spec, change-password.dto.spec | — | — | ✅ |
| US-01-02 | 帳號管理 | `GET/POST/PUT /api/users` | users/ | users.service.spec, update-role.dto.spec, update-status.dto.spec | — | — | ✅ |
| US-01-03 | 安全登出 | `POST /api/auth/logout` | auth/ | auth.service.spec | — | — | ✅ |

**E2E 覆蓋**：`app.e2e-spec.ts`（登入/登出/Token 流程）

---

### FR-02：RAG 智慧問答 ✅

| US | 說明 | API 端點 | NestJS 模組 | NestJS 測試 | Python 測試 | TC | 狀態 |
|----|------|----------|------------|-------------|-------------|-----|------|
| US-02-01 | 自然語言資安查詢 | `POST /api/chat` | chat/ | chat.service.spec, rag-proxy.service.spec, context-builder.service.spec | test_rag_chain, test_retriever | TC-02-001 | ✅ |
| US-02-01 | SSE 串流 | `POST /api/chat`（SSE） | chat/ | chat.service.spec | — | — | ✅ |
| US-02-02 | 引用來源顯示 | `POST /api/chat`（response.sources） | chat/ | context-builder.service.spec | test_retriever | — | ✅ |
| US-02-03 | 歷史對話管理 | `GET /api/chat/conversations` | chat/ | chat.service.spec | — | — | ✅ |
| US-02-04 | 多輪對話 + Query 改寫 | `POST /api/chat`（history injection） | chat/ | query-preprocessor.service.spec | test_prompts | — | ✅ |
| US-02-05 | 分數過濾與品質保障 | `POST /api/v1/rag/retrieve`（閾值） | chat/ + rag-service | rag-proxy.service.spec | test_retriever, test_rag_chain | — | ✅ |
| — | 網路搜尋補充 | `GET /api/websearch/config` | websearch/ | searxng.service.spec, websearch-config.service.spec, web-fetcher.service.spec | — | — | ✅ |
| — | 三層模式切換 | `GET /api/chat/modes` | chat/ | chat.dto.spec | — | — | ✅ |

**E2E 覆蓋**：`app.e2e-spec.ts`（聊天流程）

---

### FR-03：提示詞管理 🔄

| US | 說明 | API 端點 | NestJS 模組 | NestJS 測試 | Python 測試 | TC | 狀態 |
|----|------|----------|------------|-------------|-------------|-----|------|
| US-03-01 | 提示詞範本 CRUD | `GET/POST/PUT/DELETE /api/prompts` | prompts/ | create-prompt.dto.spec, update-prompt.dto.spec | — | — | ✅ |
| US-03-02 | 角色差異化回應 | （提示詞+chat 配合） | prompts/ + chat/ | — | test_prompts | — | ✅ |
| US-03-03 | 提示詞測試 | `POST /api/prompts/test` | prompts/ | test-prompt.dto.spec | — | — | ✅ |
| US-03-04 | 角色×模式組合 | `POST /api/prompts`（role+mode） | prompts/ | create-prompt.dto.spec | — | — | 🔄 |

---

### FR-04：資料去識別化 ✅

| US | 說明 | API 端點 | NestJS 模組 | NestJS 測試 | Python 測試 | TC | 狀態 |
|----|------|----------|------------|-------------|-------------|-----|------|
| US-04-01 | PII 自動偵測 | `POST /api/v1/clean` | cleaning/ → FastAPI | clean.controller.spec, cleaning-proxy.service.spec | test_detector, test_recognizers, test_name_context_integration | TC-04-001 | ✅ |
| US-04-02 | 去識別化策略 | `POST /api/v1/clean`（strategy 參數） | cleaning/ → FastAPI | clean.controller.spec | test_anonymizer, test_strategies, test_anonymizer_overlap, test_encrypt_key_rotation | — | ✅ |
| US-04-03 | 規則組合管理 | `GET/POST/PUT/DELETE /api/v1/rules` | cleaning/ → FastAPI | rules.controller.spec | test_rules_api, test_default_rules | — | ✅ |
| US-04-04 | 批次處理 | `POST /api/v1/clean`（多檔） | cleaning/ → FastAPI | clean.controller.spec | test_clean_api | — | ✅ |

**Python data-pipeline 測試**：test_parsers, test_text_parser, test_excel_parser, test_loaders_binary, test_language_detector, test_relation_keeper

---

### FR-05：檔案上傳與下載 ✅

| US | 說明 | API 端點 | NestJS 模組 | NestJS 測試 | Python 測試 | TC | 狀態 |
|----|------|----------|------------|-------------|-------------|-----|------|
| US-05-01 | 多格式檔案上傳 | `POST /api/v1/upload` | cleaning/ → FastAPI | upload.controller.spec | test_upload_api, test_upload_integration | TC-03-001 | ✅ |
| US-05-02 | 清洗結果下載 | `GET /api/v1/download/:taskId` | cleaning/ → FastAPI | download.controller.spec | test_download_api | — | ✅ |

---

### FR-06：即時通知 ✅

| US | 說明 | API 端點 | NestJS 模組 | NestJS 測試 | Python 測試 | TC | 狀態 |
|----|------|----------|------------|-------------|-------------|-----|------|
| US-06-01 | WebSocket 任務進度 | `ws://` WebSocket gateway | websocket/ | — | test_ws_manager | — | ✅ |

---

### FR-07：稽核日誌 ✅

| US | 說明 | API 端點 | NestJS 模組 | NestJS 測試 | Python 測試 | TC | 狀態 |
|----|------|----------|------------|-------------|-------------|-----|------|
| US-07-01 | 操作記錄查詢 | `GET /api/audit-logs` | audit/ | audit-query.dto.spec, audit-export.dto.spec, audit-log.interceptor.spec | test_audit_repo | — | ✅ |
| US-07-01 | CSV/JSON 匯出 | `GET /api/audit-logs/export` | audit/ | audit-export.dto.spec | — | — | ✅ |

---

### FR-08：去識別化規則管理 ✅

| US | 說明 | API 端點 | NestJS 模組 | NestJS 測試 | Python 測試 | TC | 狀態 |
|----|------|----------|------------|-------------|-------------|-----|------|
| US-08-01 | 規則設定檔 CRUD | `GET/POST/PUT/DELETE /api/v1/rules` | cleaning/ → FastAPI | rules.controller.spec | test_rules_api, test_default_rules | — | ✅ |

---

### FR-09：任務管理 ✅

| US | 說明 | API 端點 | NestJS 模組 | NestJS 測試 | Python 測試 | TC | 狀態 |
|----|------|----------|------------|-------------|-------------|-----|------|
| US-09-01 | 任務生命週期 | `GET/POST /api/v1/tasks` | cleaning/ → FastAPI | tasks.controller.spec | test_tasks_api | — | ✅ |

---

### FR-16：三層式回應機制 🔄

| US | 說明 | API 端點 | NestJS 模組 | NestJS 測試 | Python 測試 | TC | 狀態 |
|----|------|----------|------------|-------------|-------------|-----|------|
| US-16-01 | 角色預設模式 | `GET /api/chat/modes` | chat/ | chat.dto.spec | — | — | ✅ |
| US-16-02 | 模式切換+差異化 | `POST /api/chat`（mode 參數） | chat/ + llm/ | chat.service.spec, llm.service.spec | test_llm_temperature, test_retriever | — | ✅ |
| US-16-03 | 模式提示詞管理 | `GET/PUT /api/prompts` | prompts/ | create-prompt.dto.spec | test_prompts | — | 🔄 |

---

### FR-17：在地化法規知識庫 ✅/🔄

| US | 說明 | API 端點 | NestJS 模組 | NestJS 測試 | Python 測試 | TC | 狀態 |
|----|------|----------|------------|-------------|-------------|-----|------|
| US-17-01 | 法規知識查詢 | `POST /api/v1/rag/retrieve` + `POST /api/chat` | chat/ + rag-service | rag-proxy.service.spec | test_retriever, test_rag_chain, test_chunker, test_preprocessor | — | ✅ |

**RAG 基礎建設測試**：test_bm25_store, test_embedding_cache, test_embedding_factory, test_llm_factory, test_qdrant_store_browse, test_config, test_models, test_schemas

---

### FR-18：清洗審核管理 — Cleaner App ✅

| US | 說明 | API 端點 | NestJS 模組 | NestJS 測試 | Python 測試 | TC | 狀態 |
|----|------|----------|------------|-------------|-------------|-----|------|
| US-18-01 | 待審核任務瀏覽 | `GET /api/v1/review/:taskId` | cleaning/ → FastAPI | review.controller.spec, review.dto.spec | test_review_api | TC-05-001 | ✅ |
| US-18-02 | 檔案內容檢視 | `GET /api/v1/review/:taskId/files/:fileId/content` | cleaning/ → FastAPI | review.controller.spec | test_review_api | — | ✅ |
| US-18-03 | 內容手動修正 | `PUT /api/v1/review/:taskId/files/:fileId/content` | cleaning/ → FastAPI | review.controller.spec | test_review_api | TC-05-002 | ✅ |
| US-18-04 | 標籤管理 | `PUT /api/v1/review/:taskId/files/:fileId/tags` | cleaning/ → FastAPI | review.controller.spec | test_review_api | TC-05-003 | ✅ |
| US-18-05 | 任務批准/駁回 | `POST /api/v1/review/:taskId/approve` | cleaning/ → FastAPI | review.controller.spec | test_review_api, test_review_integration | TC-05-004 | ✅ |
| US-18-06 | 送入 RAG 知識庫 | `POST /api/v1/review/:taskId/ingest` | cleaning/ → FastAPI → rag-service | review.controller.spec | test_ingest_api, test_ingest_integration | TC-05-005 | ✅ |
| US-18-07 | 來源資料瀏覽 | `GET /api/v1/files` | cleaning/ → FastAPI | files.controller.spec | test_files_list_api | — | ✅ |
| US-18-08 | 知識庫文件瀏覽 | `GET /api/v1/knowledge-base/documents/*` | cleaning/ → FastAPI → rag-service | knowledge-base.controller.spec | test_knowledge_base_api, test_knowledge_base_integration | — | ✅ |
| US-18-09 | 知識庫文件刪除 | `DELETE /api/v1/knowledge-base/documents/by-source` | cleaning/ → FastAPI → rag-service | knowledge-base.controller.spec | test_knowledge_base_api | — | ✅ |

---

### FR-19：資料分析儀表板 ✅

| US | 說明 | API 端點 | NestJS 模組 | NestJS 測試 | Python 測試 | TC | 狀態 |
|----|------|----------|------------|-------------|-------------|-----|------|
| US-19-01 | 清洗統計 | `GET /api/v1/analytics/cleaning` | cleaning/ → FastAPI | analytics.controller.spec, analytics.dto.spec | test_analytics_api | TC-06-001 | ✅ |
| US-19-02 | 知識庫統計 | `GET /api/v1/analytics/knowledge-base` | cleaning/ → FastAPI | analytics.controller.spec | test_analytics_api | — | ✅ |
| US-19-03 | 時間軸統計 | `GET /api/v1/analytics/timeline` | cleaning/ → FastAPI | analytics.controller.spec | test_analytics_api | TC-06-002 | ✅ |

---

### FR-20：來源資料瀏覽 ✅

| US | 說明 | API 端點 | NestJS 模組 | NestJS 測試 | Python 測試 | TC | 狀態 |
|----|------|----------|------------|-------------|-------------|-----|------|
| US-20-01 | 來源檔案列表 | `GET /api/v1/files` | cleaning/ → FastAPI | files.controller.spec | test_files_list_api | — | ✅ |

---

### FR-21：知識庫文件瀏覽 ✅

| US | 說明 | API 端點 | NestJS 模組 | NestJS 測試 | Python 測試 | TC | 狀態 |
|----|------|----------|------------|-------------|-------------|-----|------|
| US-21-01 | 知識庫文件聚合 | `GET /api/v1/knowledge-base/documents/grouped` | cleaning/ → FastAPI → rag-service | knowledge-base.controller.spec | test_knowledge_base_api, test_knowledge_base_integration | — | ✅ |
| US-21-01 | 依來源篩選 | `GET /api/v1/knowledge-base/documents/by-source` | cleaning/ → FastAPI → rag-service | knowledge-base.controller.spec | test_knowledge_base_api | — | ✅ |

---

## 2. 追溯矩陣 — 未來功能

| FR | 說明 | 狀態 | 目標階段 |
|----|------|------|---------|
| FR-10 | 進階 RBAC 權限管理 | 🔮 | Phase 3 |
| FR-11 | 多會員架構 | 🔮 | Phase 3 |
| FR-12 | 多知識庫管理 | 🔮 | Phase 3 |
| FR-13 | 智慧分析儀表板 | 🔮 | Phase 3 |
| FR-14 | 第三方整合 | 🔮 | Phase 3 |
| FR-15 | 運維監控與告警 | 🔮 | Phase 3 |

---

## 3. 驗收測試追溯（SRS_TECHNICAL.md §11）

| TC 編號 | 對應 FR | 對應 US | 說明 | NestJS 測試 | Python 測試 | 狀態 |
|---------|---------|---------|------|-------------|-------------|------|
| TC-01-001 | FR-01 | US-01-01 | 正常登入 | auth.service.spec + E2E | — | ✅ |
| TC-01-002 | FR-01 | US-01-01 | 登入失敗（帳密錯誤/鎖定） | auth.service.spec + E2E | — | ✅ |
| TC-02-001 | FR-02 | US-02-01 | RAG 查詢回應 | chat.service.spec, rag-proxy.service.spec | test_rag_chain, test_retriever | ✅ |
| TC-03-001 | FR-05 | US-05-01 | 檔案上傳 | upload.controller.spec | test_upload_api | ✅ |
| TC-04-001 | FR-04 | US-04-01 | 清洗任務執行 | clean.controller.spec | test_clean_api, test_detector | ✅ |
| TC-05-001 | FR-18 | US-18-01 | 清洗審核瀏覽 | review.controller.spec | test_review_api | ✅ |
| TC-05-002 | FR-18 | US-18-03 | 內容手動編輯 | review.controller.spec | test_review_api | ✅ |
| TC-05-003 | FR-18 | US-18-04 | 標籤管理 | review.controller.spec | test_review_api | ✅ |
| TC-05-004 | FR-18 | US-18-05 | 任務批准 | review.controller.spec | test_review_integration | ✅ |
| TC-05-005 | FR-18 | US-18-06 | 送入 RAG | review.controller.spec | test_ingest_api, test_ingest_integration | ✅ |
| TC-06-001 | FR-19 | US-19-01 | 清洗統計 | analytics.controller.spec | test_analytics_api | ✅ |
| TC-06-002 | FR-19 | US-19-03 | 時間軸統計 | analytics.controller.spec | test_analytics_api | ✅ |

---

## 4. 跨切面需求追溯

以下為非功能性需求與架構決策的追溯。

### 4.1 安全性需求

| 需求 | 實作機制 | 測試覆蓋 | ADR 參考 |
|------|---------|----------|----------|
| JWT 認證 | JwtAuthGuard 全域啟用 | auth.service.spec, E2E | — |
| RBAC 授權 | RolesGuard + @Roles 裝飾器 | roles.guard.spec | — |
| 密碼政策（資通安全「普」級） | IsStrongPassword validator | password-strength.validator.spec | [ADR-007](../architecture/ARCH.md#adr-007) |
| 內部 API 認證 | X-Internal-Token 中介軟體 | test_internal_auth_middleware | — |
| Rate Limiting | ThrottlerModule 60req/min | E2E | — |
| 輸入驗證 | ValidationPipe + class-validator | DTO spec 檔案（15+） | — |
| XSS 防護 | React 自動轉義 + helmet headers | — | — |
| SQL Injection | Prisma + SQLAlchemy 參數化查詢 | — | [ADR-003](../architecture/ARCH.md#adr-003) |

### 4.2 效能需求

| 需求 | 目標值 | 相關模組 | 測試狀態 |
|------|--------|---------|---------|
| RAG 查詢回應 | P95 < 10s | chat/ + rag-service | 規劃中（k6） |
| 檔案上傳速度 | 10MB/s | cleaning/ | 規劃中 |
| 清洗處理速度 | > 10 files/min | FastAPI | 規劃中 |
| 並發使用者 | 100 同時在線 | 全系統 | 規劃中 |

### 4.3 架構決策追溯

| ADR | 決策 | 影響的 FR | 參考 |
|-----|------|----------|------|
| ADR-001 | Monorepo (pnpm + Turborepo) | 全部 | [ARCH.md](../architecture/ARCH.md#adr-001) |
| ADR-002 | NestJS 統一 API Gateway | FR-04~09, FR-18~21 | [ARCH.md](../architecture/ARCH.md#adr-002) |
| ADR-003 | 雙 ORM 策略 | FR-01~09 + FR-04~09 | [ARCH.md](../architecture/ARCH.md#adr-003) |
| ADR-004 | Hybrid Search (Vector+BM25+RRF) | FR-02, FR-17 | [ARCH.md](../architecture/ARCH.md#adr-004) |
| ADR-005 | SSE 串流回應 | FR-02 | [ARCH.md](../architecture/ARCH.md#adr-005) |
| ADR-006 | 三前端應用 | FR-01~03, FR-04~09, FR-18~21 | [ARCH.md](../architecture/ARCH.md#adr-006) |
| ADR-007 | 資通安全「普」級密碼政策 | FR-01 | [ARCH.md](../architecture/ARCH.md#adr-007) |
| ADR-008 | SearXNG 搜尋引擎 | FR-02 | [ARCH.md](../architecture/ARCH.md#adr-008) |
| ADR-009 | Presidio PII 偵測 | FR-04 | [ARCH.md](../architecture/ARCH.md#adr-009) |
| ADR-010 | 階層式 Chunking 策略 | FR-17 | [ARCH.md](../architecture/ARCH.md#adr-010) |

---

## 5. 涵蓋度摘要

### 5.1 功能需求分布

| 類別 | FR 數量 | US 數量 | 狀態 |
|------|---------|---------|------|
| 已實作（✅） | 12 | 25 | Unit + Integration 覆蓋 |
| 規劃中（🔄） | 3 | 10 | 部分實作、持續強化 |
| 未來願景（🔮） | 6 | 6（未展開） | Phase 3 以後 |
| **合計** | **21** | **41** | — |

### 5.2 測試覆蓋統計

| 層級 | 檔案數 | 覆蓋範圍 |
|------|--------|---------|
| NestJS Unit (.spec.ts) | 43 | 全部已實作模組（auth, users, chat, prompts, audit, cleaning, health, websearch, llm, datasources） |
| Python rag-service (test_*.py) | 38 | RAG 核心管線、API 路由、整合測試 |
| Python data-pipeline (test_*.py) | 14 | 解析器、偵測器、去識別化策略 |
| NestJS E2E (.e2e-spec.ts) | 1 | 認證 + 聊天 + 管理流程 |
| **合計** | **96** | — |

### 5.3 追溯完整性

| 檢核項目 | 狀態 | 說明 |
|----------|------|------|
| 每個已實作 US 有對應 API 端點 | ✅ | 35/35 US 有明確端點 |
| 每個 API 端點有 NestJS 測試 | ✅ | 43 spec 檔案覆蓋全部 Controller/Service |
| 每個 FastAPI 路由有 Python 測試 | ✅ | 38 test 檔案覆蓋 API + 整合 |
| SRS §11 驗收案例有對應測試 | ✅ | 12/12 TC 全數有測試覆蓋 |
| 每個 ADR 有對應 FR 引用 | ✅ | 10/10 ADR 標注影響範圍 |

---

## 6. Quality Gate 檢核

### 6.1 Phase 1（當前）

- [x] 所有 FR-01~09 + FR-16~21 有 Unit Test 覆蓋
- [x] 雙語言服務（NestJS + Python）皆有獨立測試
- [x] 驗收案例 TC-01~06 全數有對應測試
- [x] 安全性檢核項全數通過（SRS-T §11.3）
- [ ] 行覆蓋率量測（NestJS ≥80%、Python ≥80%）— 規劃中

### 6.2 Phase 2（目標）

- [ ] 效能測試基準建立（k6 壓力測試）
- [ ] 前端元件測試導入（Vitest + RTL）
- [ ] E2E 自動化擴充（Playwright）
- [ ] CI/CD 整合測試閘門

### 6.3 Phase 3（願景）

- [ ] FR-10~15 User Story 展開與追溯
- [ ] 多租戶架構安全測試
- [ ] 合規性自動化稽核

---

## 參考文件

| 文件 | 說明 | 路徑 |
|------|------|------|
| PRD.md | User Story 定義 | [specs/PRD.md](./PRD.md) |
| SRS_TECHNICAL.md | FR 定義與驗收標準 | [specs/SRS_TECHNICAL.md](./SRS_TECHNICAL.md) |
| ARCH.md | 架構決策紀錄 | [architecture/ARCH.md](../architecture/ARCH.md) |
| test-strategy.md | 測試策略 | [testing/test-strategy.md](../testing/test-strategy.md) |
| API 文件索引 | 全部端點規格 | [api/README.md](../api/README.md) |

---

> **文件結束**
>
> 本文件為 ODA Cyber Konsult 的需求追溯矩陣，串連 User Story → 功能需求 → API → 模組 → 測試 的完整鏈路。
> 新增功能時，請同步更新對應的追溯列。
