# 需求追溯矩陣 (RTM)

> **ODA Cyber Konsult - 資安助手 RAG 系統**
>
> 文件版本：1.6.0
> 建立日期：2026-03-01
> 最後更新：2026-04-29
> 文件類型：需求追溯矩陣（Requirements Traceability Matrix）

---

## 版本歷史

| 版本 | 日期 | 變更說明 |
|------|------|----------|
| v1.0.0 | 2026-03-01 | 初版建立，FR-01~21 追溯矩陣 |
| v1.1.0 | 2026-03-06 | 新增 FR-18 Maker-Checker 追溯、FR-19~21 追溯、驗收測試更新 |
| v1.2.0 | 2026-03-11 | 新增 US-05-03 ZIP 追溯、更新角色定義為 7 角色、測試覆蓋統計更新（107 test files） |
| v1.3.0 | 2026-03-16 | ML-15：新增 FR-22 回饋機制追溯、FR-23 信心度追溯；驗收案例 TC-05-006/007 確認 |
| v1.4.0 | 2026-04-18 | **Wave 1-5 功能 + 機制修正追溯同步**：<br/>① 新增 19 項 Wave 功能追溯（標籤範本/報告模板/Demo 問題庫/Golden Test/測試執行器/法規版本管理）<br/>② gap-analysis P1 修補映射（GAP-01 ~ GAP-05 全數 closed）<br/>③ 記錄本 session 揭露的 Maker-Checker UUID 比較 bug 修復、changePassword tokenVersion 補齊、Expert maxTokens 規格校正、ZIP 閾值校正、query history 5 輪修正、RejectTaskRequest validator、file content audit log、SELECT FOR UPDATE 行鎖<br/>④ 新增反偽測試覆蓋率指標（舊 15 處 `inspect.getsource()` 已識別，待 Q2 取代）<br/>⑤ 補充 AC-02-05 vs AC-23-01 閾值語意差異說明<br/>⑥ 明確標註 3 項 NFR 未驗證項目（PII 95% 召回率、P95 < 10s、10 files/分吞吐量） |
| v1.6.0 | 2026-04-29 | **SDD contracts SSoT + spec pack 試點（B+D 任務批次）**：<br/>① **`contracts/` 跨層級 SSoT 建立**：root `contracts/thresholds.yml` 收 7 個 RAG 閾值常數；`contracts/enums.yml` 收 user_roles 7 角色；`packages/contracts` workspace 提供 TS 載入；`python/shared/contracts.py` Python stub<br/>② **`chat.service.ts` 7 處 magic number 替換**為 `RAG_THRESHOLDS.*`（line 326/341/385-386）；`create-user.dto.ts` 改用 `USER_ROLES`<br/>③ **anti-drift 行為驗證**：`chat.service.contract.spec.ts` 用 jest.mock 注入 0.999 極端值（5/5 綠）；`python/tests/test_contract_parity.py` 跨語言 schema parity（7/7 綠）<br/>④ **chat-rag 試點 spec pack**：`docs/04-features/chat-rag/{requirements,design,tasks}.md` 守 D0 紅線（指針 + 功能特殊規則 + 踩坑歷史，禁止複製 PRD）<br/>⑤ **+7 天驗收期**：CLAUDE_TASK.md 持續追蹤條目（D5 三題：人審查/AI context/D0 紅線；任一 ❌ 即撤除試點）<br/>⑥ 對應 ai-native-sdlc skill QG-3 強化（references/qg3-defense-lines.md）+ pattern-anti-pseudo-test 自動載入 + domain-code-review IDOR 4-checklist |
| v1.5.1 | 2026-04-29 | **文件對齊（A 任務批次）**：<br/>① **PRD AC-23-01-04 補閾值語意交叉註腳**（PRD line 582）：明示「過濾閾值 vs 信心度判定閾值」兩層不同語意，引用 line 127-131 閾值語意說明<br/>② **system-cross-validation-2026-04-18.md §1.2「矛盾 1」+ §3.2「飄移 #6」校正撤回**：經 PRD line 127-131 確認 0.005/0.003 為兩層獨立閾值（filterLowScoreResults vs determineConfidenceLevel 兩函式），原稽核判斷錯誤已標註撤回；保留軌跡供後續 reviewer 追溯<br/>③ **新增 AC-01-01-05 tokenVersion 失效規格**：補 fd20f53 修復對應的規格條文（密碼變更後 tokenVersion 遞增、既發 JWT 立即失效）。CLAUDE_LESSONS「Credential Rotation 事件窮舉」對應 |
| v1.5.0 | 2026-04-22 | **codex 稽核 4 項 + 自我稽核 2 項「文件完成 ≠ 實作完成」缺陷修正**：<br/>① **變數注入**（AC-03-01-03）：context-builder 僅處理 `{context}`，補齊 `{user_name}` / `{query}` 替換，新增共用 `prompt-renderer.util.ts`，chat.service fetch `users.getDisplayName` 注入 user_name<br/>② **提示詞測試 vs 正式執行格式統一**（US-03-03）：prompts.service.testPrompt 原用 `{{key}}` 雙括號與正式 `{key}` 單括號不符，統一改用單括號 renderer，新增 `prompts.service.spec.ts` 等價性 golden test 保證未來兩路徑不再分歧<br/>③ **三層模式 topK 差異化**（AC-16-02-01~03）：chatbot 前端固定送 `topK: 5` 覆蓋後端 mode-aware fallback (3/5/8)，移除硬編碼，新增 `useChat.test.ts` smoke test<br/>④ **Prompt UI 角色擴充**（AC-01-02-03）：admin PromptsPage 補齊 `basic_user` / `it_user` 角色（colors / labels / filters / Select options 四處），使 SRS §4.4 規定的「it_user 專屬提示詞」得以透過 UI 建立<br/>⑤ **UpdatePromptDto `it_user` 遺漏補齊**（自我稽核發現）：create-prompt.dto.ts 已列 5 角色，但兄弟 update-prompt.dto.ts 只列 4 角色——意即 it_user 提示詞可建立、不可編輯；同時兩份 DTO spec 的 it.each 也遺漏了 it_user。修復方案：提煉 `PROMPT_ROLES` / `PROMPT_MODES` 共用常數於 create-prompt.dto.ts，create/update DTO 與兩份 spec 全部引用同一來源，根除列舉雙源漂移<br/>⑥ **SRS §5.17.3 表格校正**：原「it_user | standard | beginner, standard」與 PRD v1.5.0 版本歷史 line 21「it_user 開放 expert 模式」矛盾，seed.ts 已有 it_user expert 提示詞；表格更新為「beginner, standard, expert」<br/>⑦ 本次修正對應的測試新增：`prompt-renderer.util.spec`、`prompts.service.spec`、`context-builder.service.spec`（新增變數替換案例）、`useChat.test`；既有 `create-prompt.dto.spec` / `update-prompt.dto.spec` 改為引用 PROMPT_ROLES 常數驅動<br/>⑧ CLAUDE_LESSONS.md 新增「規格漂移與驗證漏洞」段落記錄四項通用教訓（含新增的「Create/Update DTO 兄弟對稱 + 列舉覆蓋測試漂移」） |

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

> **角色定義**（7 角色）：basic_user / user / it_user / consultant / data_cleaner / data_reviewer / admin

| US | 說明 | API 端點 | NestJS 模組 | NestJS 測試 | Python 測試 | TC | Screen | Security | 狀態 |
|----|------|----------|------------|-------------|-------------|-----|--------|----------|------|
| US-01-01 | 帳號密碼登入 | `POST /api/auth/login` | auth/ | auth.service.spec, token.service.spec, login.dto.spec, password-strength.validator.spec, password-change-required.guard.spec | — | TC-01-001, TC-01-002 | Admin `/login`, Cleaner `/login`, Chatbot `/login` | JWT, bcrypt, RBAC, Rate Limit, 帳號鎖定, 密碼政策 | ✅ |
| US-01-01 | 註冊 | `POST /api/auth/register` | auth/ | auth.service.spec, register.dto.spec | — | — | Chatbot `/login` | JWT, bcrypt, RBAC, Rate Limit, 帳號鎖定, 密碼政策 | ✅ |
| US-01-01 | Token 刷新 | `POST /api/auth/refresh` | auth/ | auth.service.spec, refresh.dto.spec | — | — | —（自動） | JWT, bcrypt, RBAC, Rate Limit, 帳號鎖定, 密碼政策 | ✅ |
| US-01-01 | 密碼變更 | `POST /api/auth/change-password` | auth/ | auth.service.spec, change-password.dto.spec | — | — | Admin `/change-password`, Cleaner `/change-password`, Chatbot `/change-password` | JWT, bcrypt, RBAC, Rate Limit, 帳號鎖定, 密碼政策 | ✅ |
| US-01-02 | 帳號管理 | `GET/POST/PUT /api/users` | users/ | users.service.spec, update-role.dto.spec, update-status.dto.spec | — | — | Admin `/users` | JWT, bcrypt, RBAC, Rate Limit, 帳號鎖定, 密碼政策 | ✅ |
| US-01-03 | 安全登出 | `POST /api/auth/logout` | auth/ | auth.service.spec | — | — | 全應用 | JWT, bcrypt, RBAC, Rate Limit, 帳號鎖定, 密碼政策 | ✅ |

**E2E 覆蓋**：`app.e2e-spec.ts`（登入/登出/Token 流程）

---

### FR-02：RAG 智慧問答 ✅

> 📦 **Spec Pack**（試點功能）：[`docs/04-features/chat-rag/`](../04-features/chat-rag/) — requirements / design / tasks 三件套；含 SEC-IDOR-01、SEM-THRESHOLD-01、CONTRACTS-01 三條功能特殊規則；於 2026-04-29 建立，+7 天驗收（D5 plan）。

| US | 說明 | API 端點 | NestJS 模組 | NestJS 測試 | Python 測試 | TC | Screen | Security | 狀態 |
|----|------|----------|------------|-------------|-------------|-----|--------|----------|------|
| US-02-01 | 自然語言資安查詢 | `POST /api/chat` | chat/ | chat.service.spec, rag-proxy.service.spec, context-builder.service.spec | test_rag_chain, test_retriever | TC-02-001 | Chatbot 主畫面 | JWT, Input Validation, Rate Limit | ✅ |
| US-02-01 | SSE 串流 | `POST /api/chat`（SSE） | chat/ | chat.service.spec | — | — | Chatbot 主畫面 | JWT, Input Validation, Rate Limit | ✅ |
| US-02-02 | 引用來源顯示 | `POST /api/chat`（response.sources） | chat/ | context-builder.service.spec | test_retriever | — | Chatbot 主畫面 | JWT, Input Validation, Rate Limit | ✅ |
| US-02-03 | 歷史對話管理 | `GET /api/chat/conversations` | chat/ | chat.service.spec | — | — | Chatbot 主畫面 | JWT, Input Validation, Rate Limit | ✅ |
| US-02-04 | 多輪對話 + Query 改寫 | `POST /api/chat`（history injection） | chat/ | query-preprocessor.service.spec | test_prompts | — | Chatbot 主畫面 | JWT, Input Validation, Rate Limit | ✅ |
| US-02-05 | 分數過濾與品質保障 | `POST /api/v1/rag/retrieve`（閾值） | chat/ + rag-service | rag-proxy.service.spec | test_retriever, test_rag_chain | — | Chatbot 主畫面 | JWT, Input Validation, Rate Limit | ✅ |
| — | 網路搜尋補充 | `GET /api/websearch/config` | websearch/ | searxng.service.spec, websearch-config.service.spec, web-fetcher.service.spec | — | — | Chatbot 主畫面, Admin `/settings` | JWT, Input Validation, Rate Limit | ✅ |
| — | 三層模式切換 | `GET /api/chat/modes` | chat/ | chat.dto.spec | — | — | Chatbot 主畫面, Admin `/settings` | JWT, Input Validation, Rate Limit | ✅ |

**E2E 覆蓋**：`app.e2e-spec.ts`（聊天流程）

---

### FR-03：提示詞管理 🔄

| US | 說明 | API 端點 | NestJS 模組 | NestJS 測試 | Python 測試 | TC | Screen | Security | 狀態 |
|----|------|----------|------------|-------------|-------------|-----|--------|----------|------|
| US-03-01 | 提示詞範本 CRUD | `GET/POST/PUT/DELETE /api/prompts` | prompts/ | create-prompt.dto.spec, update-prompt.dto.spec, prompt-renderer.util.spec | — | — | Admin `/prompts` | JWT, RBAC(admin), Input Validation | ✅ |
| US-03-02 | 角色差異化回應 | （提示詞+chat 配合） | prompts/ + chat/ | context-builder.service.spec（變數注入） | test_prompts | — | Admin `/prompts` | JWT, RBAC(admin), Input Validation | ✅ |
| US-03-03 | 提示詞測試 | `POST /api/prompts/test` | prompts/ | test-prompt.dto.spec, prompts.service.spec（含等價性 golden test） | — | — | Admin `/prompts` | JWT, RBAC(admin), Input Validation | ✅ |
| US-03-04 | 角色×模式組合 | `POST /api/prompts`（role+mode） | prompts/ | create-prompt.dto.spec | — | — | Admin `/prompts` | JWT, RBAC(admin), Input Validation | 🔄 |

---

### FR-04：資料去識別化 ✅

| US | 說明 | API 端點 | NestJS 模組 | NestJS 測試 | Python 測試 | TC | Screen | Security | 狀態 |
|----|------|----------|------------|-------------|-------------|-----|--------|----------|------|
| US-04-01 | PII 自動偵測 | `POST /api/v1/clean` | cleaning/ → FastAPI | clean.controller.spec, cleaning-proxy.service.spec | test_detector, test_recognizers, test_name_context_integration | TC-04-001 | Admin `/` | JWT, RBAC(admin), X-Internal-Token, PII 處理 | ✅ |
| US-04-02 | 去識別化策略 | `POST /api/v1/clean`（strategy 參數） | cleaning/ → FastAPI | clean.controller.spec | test_anonymizer, test_strategies, test_anonymizer_overlap, test_encrypt_key_rotation | — | Admin `/` | JWT, RBAC(admin), X-Internal-Token, PII 處理 | ✅ |
| US-04-03 | 固定規則套用 | `POST /api/v1/clean`（自動套用 `get_default_rules()`） | cleaning/ → FastAPI | clean.controller.spec | test_clean_api, test_default_rules | — | Admin `/` | JWT, RBAC(admin), X-Internal-Token, PII 處理 | ✅ |
| US-04-04 | 批次處理 | `POST /api/v1/clean`（多檔） | cleaning/ → FastAPI | clean.controller.spec | test_clean_api | — | Admin `/` | JWT, RBAC(admin), X-Internal-Token, PII 處理 | ✅ |

**Python data-pipeline 測試**：test_parsers, test_text_parser, test_excel_parser, test_loaders_binary, test_language_detector, test_relation_keeper

---

### FR-05：檔案上傳與下載 ✅

| US | 說明 | API 端點 | NestJS 模組 | NestJS 測試 | Python 測試 | TC | Screen | Security | 狀態 |
|----|------|----------|------------|-------------|-------------|-----|--------|----------|------|
| US-05-01 | 多格式檔案上傳 | `POST /api/v1/upload` | cleaning/ → FastAPI | upload.controller.spec | test_upload_api, test_upload_integration | TC-03-001 | Admin `/` | JWT, RBAC(admin), 50MB 限制, Input Validation | ✅ |
| US-05-02 | 清洗結果下載 | `GET /api/v1/download/:taskId` | cleaning/ → FastAPI | download.controller.spec | test_download_api | — | Admin `/tasks` | JWT, RBAC(admin), 50MB 限制, Input Validation | ✅ |
| US-05-03 | ZIP 上傳 | `POST /api/v1/upload`（ZIP 自動解壓+auto_tags） | cleaning/ → FastAPI | upload.controller.spec | test_upload_api, test_upload_integration | AC-05-03-01~04 | Admin `/` | JWT, RBAC(admin), 50MB 限制, Input Validation | ✅ |

---

### FR-06：即時通知 ✅

| US | 說明 | API 端點 | NestJS 模組 | NestJS 測試 | Python 測試 | TC | Screen | Security | 狀態 |
|----|------|----------|------------|-------------|-------------|-----|--------|----------|------|
| US-06-01 | WebSocket 任務進度 | `ws://` WebSocket gateway | websocket/ | — | test_ws_manager | — | Admin `/tasks` | JWT, WebSocket 認證 | ✅ |

---

### FR-07：稽核日誌 ✅

| US | 說明 | API 端點 | NestJS 模組 | NestJS 測試 | Python 測試 | TC | Screen | Security | 狀態 |
|----|------|----------|------------|-------------|-------------|-----|--------|----------|------|
| US-07-01 | 操作記錄查詢 | `GET /api/audit-logs` | audit/ | audit-query.dto.spec, audit-export.dto.spec, audit-log.interceptor.spec | test_audit_repo | — | Admin `/audit-logs` | JWT, RBAC(admin) | ✅ |
| US-07-01 | CSV/JSON 匯出 | `GET /api/audit-logs/export` | audit/ | audit-export.dto.spec | — | — | Admin `/audit-logs` | JWT, RBAC(admin) | ✅ |

---

### FR-08：去識別化規則管理 — 已簡化

> **變更說明**：原規劃為自訂規則 CRUD API，已簡化為固定規則模式（`get_default_rules()`）。原有的 `rules.controller.spec`、`test_rules_api` 測試已移除，由 `test_default_rules` 覆蓋固定規則邏輯。

| US | 說明 | API 端點 | NestJS 模組 | NestJS 測試 | Python 測試 | TC | Screen | Security | 狀態 |
|----|------|----------|------------|-------------|-------------|-----|--------|----------|------|
| US-08-01 | 固定去識別化規則 | `POST /api/v1/clean`（自動套用） | cleaning/ → FastAPI | clean.controller.spec | test_default_rules, test_clean_api | — | Admin `/`（自動套用） | JWT, RBAC(admin), X-Internal-Token | ✅ |

---

### FR-09：任務管理 ✅

| US | 說明 | API 端點 | NestJS 模組 | NestJS 測試 | Python 測試 | TC | Screen | Security | 狀態 |
|----|------|----------|------------|-------------|-------------|-----|--------|----------|------|
| US-09-01 | 任務生命週期 | `GET/POST /api/v1/tasks` | cleaning/ → FastAPI | tasks.controller.spec | test_tasks_api | — | Admin `/tasks` | JWT, RBAC(admin), X-Internal-Token | ✅ |

---

### FR-16：三層式回應機制 🔄

| US | 說明 | API 端點 | NestJS 模組 | NestJS 測試 | Python 測試 | TC | Screen | Security | 狀態 |
|----|------|----------|------------|-------------|-------------|-----|--------|----------|------|
| US-16-01 | 角色預設模式 | `GET /api/chat/modes` | chat/ | chat.dto.spec | — | — | Chatbot 主畫面, Admin `/prompts` | JWT, Input Validation | ✅ |
| US-16-02 | 模式切換+差異化 | `POST /api/chat`（mode 參數） | chat/ + llm/ | chat.service.spec, llm.service.spec, chatbot useChat.test（前端不覆蓋 topK） | test_llm_temperature, test_retriever | — | Chatbot 主畫面, Admin `/prompts` | JWT, Input Validation | ✅ |
| US-16-03 | 模式提示詞管理 | `GET/PUT /api/prompts` | prompts/ | create-prompt.dto.spec | test_prompts | — | Chatbot 主畫面, Admin `/prompts` | JWT, Input Validation | 🔄 |

---

### FR-17：在地化法規知識庫 ✅/🔄

| US | 說明 | API 端點 | NestJS 模組 | NestJS 測試 | Python 測試 | TC | Screen | Security | 狀態 |
|----|------|----------|------------|-------------|-------------|-----|--------|----------|------|
| US-17-01 | 法規知識查詢 | `POST /api/v1/rag/retrieve` + `POST /api/chat` | chat/ + rag-service | rag-proxy.service.spec | test_retriever, test_rag_chain, test_chunker, test_preprocessor | — | Chatbot 主畫面 | JWT, X-Internal-Token | ✅ |

**RAG 基礎建設測試**：test_bm25_store, test_embedding_cache, test_embedding_factory, test_llm_factory, test_qdrant_store_browse, test_config, test_models, test_schemas

---

### FR-18：清洗審核管理 — Cleaner App ✅

| US | 說明 | API 端點 | NestJS 模組 | NestJS 測試 | Python 測試 | TC | Screen | Security | 狀態 |
|----|------|----------|------------|-------------|-------------|-----|--------|----------|------|
| US-18-01 | 待審核任務瀏覽 | `GET /api/v1/review/:taskId` | cleaning/ → FastAPI | review.controller.spec, review.dto.spec | test_review_api | TC-05-001 | Cleaner `/tasks` | JWT, RBAC(admin/cleaner/reviewer), X-Internal-Token, PII 隔離 | ✅ |
| US-18-02 | 檔案內容檢視 | `GET /api/v1/review/:taskId/files/:fileId/content` | cleaning/ → FastAPI | review.controller.spec | test_review_api | — | Cleaner `/tasks/:taskId/files/:fileId` | JWT, RBAC(admin/cleaner/reviewer), X-Internal-Token, PII 隔離 | ✅ |
| US-18-03 | 內容手動修正 | `PUT /api/v1/review/:taskId/files/:fileId/content` | cleaning/ → FastAPI | review.controller.spec | test_review_api | TC-05-002 | Cleaner `/tasks/:taskId/files/:fileId` | JWT, RBAC(admin/cleaner/reviewer), X-Internal-Token, PII 隔離 | ✅ |
| US-18-04 | 標籤管理 | `PUT /api/v1/review/:taskId/files/:fileId/tags` | cleaning/ → FastAPI | review.controller.spec | test_review_api | TC-05-003 | Cleaner `/tasks/:taskId/files/:fileId` | JWT, RBAC(admin/cleaner/reviewer), X-Internal-Token, PII 隔離 | ✅ |
| US-18-05 | 任務批准/駁回 | `POST /api/v1/review/:taskId/approve` | cleaning/ → FastAPI | review.controller.spec | test_review_api, test_review_integration | TC-05-004 | Cleaner `/tasks/:taskId` | JWT, RBAC(admin/cleaner/reviewer), X-Internal-Token, PII 隔離, Maker-Checker | ✅ |
| US-18-06 | 送入 RAG 知識庫 | `POST /api/v1/review/:taskId/ingest` | cleaning/ → FastAPI → rag-service | review.controller.spec | test_ingest_api, test_ingest_integration | TC-05-005 | Cleaner `/tasks/:taskId/ingest` | JWT, RBAC(admin/reviewer), X-Internal-Token, PII 隔離 | ✅ |
| US-18-07 | 來源資料瀏覽 | `GET /api/v1/files` | cleaning/ → FastAPI | files.controller.spec | test_files_list_api | — | Cleaner `/files` | JWT, RBAC(admin/cleaner/reviewer), X-Internal-Token, PII 隔離 | ✅ |
| US-18-08 | 知識庫文件瀏覽 | `GET /api/v1/knowledge-base/documents/*` | cleaning/ → FastAPI → rag-service | knowledge-base.controller.spec | test_knowledge_base_api, test_knowledge_base_integration | — | Cleaner `/knowledge-base` | JWT, RBAC(admin/cleaner/reviewer), X-Internal-Token, PII 隔離 | ✅ |
| US-18-09 | 知識庫文件刪除 | `DELETE /api/v1/knowledge-base/documents/by-source` | cleaning/ → FastAPI → rag-service | knowledge-base.controller.spec | test_knowledge_base_api | — | Cleaner `/knowledge-base` | JWT, RBAC(admin/cleaner/reviewer), X-Internal-Token, PII 隔離 | ✅ |
| US-18-10 | 送審任務 | `POST /api/v1/review/:taskId/submit` | cleaning/ → FastAPI | review.controller.spec | test_review_api | TC-05-006 | Cleaner `/tasks/:taskId` | JWT, RBAC(admin/cleaner), X-Internal-Token, Maker-Checker | ✅ |
| US-18-11 | 退回任務 | `POST /api/v1/review/:taskId/reject` | cleaning/ → FastAPI | review.controller.spec | test_review_api | TC-05-007 | Cleaner `/tasks/:taskId` | JWT, RBAC(admin/reviewer), X-Internal-Token, Maker-Checker | ✅ |
| US-18-12 | Maker-Checker 職責分離 | submit/approve/reject 端點聯動 | cleaning/ → FastAPI | review.controller.spec | test_review_api, test_review_integration | TC-05-006, TC-05-007 | Cleaner `/tasks/:taskId` | JWT, RBAC, Maker-Checker（submitted_by ≠ approved_by） | ✅ |

---

### FR-19：資料分析儀表板 ✅

| US | 說明 | API 端點 | NestJS 模組 | NestJS 測試 | Python 測試 | TC | Screen | Security | 狀態 |
|----|------|----------|------------|-------------|-------------|-----|--------|----------|------|
| US-19-01 | 清洗統計 | `GET /api/v1/analytics/cleaning` | cleaning/ → FastAPI | analytics.controller.spec, analytics.dto.spec | test_analytics_api | TC-06-001 | Cleaner `/` | JWT, RBAC(admin/cleaner), X-Internal-Token | ✅ |
| US-19-02 | 知識庫統計 | `GET /api/v1/analytics/knowledge-base` | cleaning/ → FastAPI | analytics.controller.spec | test_analytics_api | — | Cleaner `/` | JWT, RBAC(admin/cleaner), X-Internal-Token | ✅ |
| US-19-03 | 時間軸統計 | `GET /api/v1/analytics/timeline` | cleaning/ → FastAPI | analytics.controller.spec | test_analytics_api | TC-06-002 | Cleaner `/` | JWT, RBAC(admin/cleaner), X-Internal-Token | ✅ |

---

### FR-20：來源資料瀏覽 ✅

| US | 說明 | API 端點 | NestJS 模組 | NestJS 測試 | Python 測試 | TC | Screen | Security | 狀態 |
|----|------|----------|------------|-------------|-------------|-----|--------|----------|------|
| US-20-01 | 來源檔案列表 | `GET /api/v1/files` | cleaning/ → FastAPI | files.controller.spec | test_files_list_api | — | Cleaner `/files` | JWT, RBAC(admin/cleaner), X-Internal-Token | ✅ |

---

### FR-21：知識庫文件瀏覽 ✅

| US | 說明 | API 端點 | NestJS 模組 | NestJS 測試 | Python 測試 | TC | Screen | Security | 狀態 |
|----|------|----------|------------|-------------|-------------|-----|--------|----------|------|
| US-21-01 | 知識庫文件聚合 | `GET /api/v1/knowledge-base/documents/grouped` | cleaning/ → FastAPI → rag-service | knowledge-base.controller.spec | test_knowledge_base_api, test_knowledge_base_integration | — | Cleaner `/knowledge-base` | JWT, RBAC(admin/cleaner), X-Internal-Token | ✅ |
| US-21-01 | 依來源篩選 | `GET /api/v1/knowledge-base/documents/by-source` | cleaning/ → FastAPI → rag-service | knowledge-base.controller.spec | test_knowledge_base_api | — | Cleaner `/knowledge-base` | JWT, RBAC(admin/cleaner), X-Internal-Token | ✅ |

---

### FR-22：回答品質回饋 ✅

| US | 說明 | API 端點 | NestJS 模組 | NestJS 測試 | Python 測試 | TC | Screen | Security | 狀態 |
|----|------|----------|------------|-------------|-------------|-----|--------|----------|------|
| US-22-01 | 回答品質回饋 | `POST /api/chat/messages/:messageId/feedback` | chat/ | chat.service.spec | — | — | Chatbot 主畫面 | JWT, Input Validation | ✅ |

---

### FR-23：RAG 信心度指示器 ✅

| US | 說明 | API 端點 | NestJS 模組 | NestJS 測試 | Python 測試 | TC | Screen | Security | 狀態 |
|----|------|----------|------------|-------------|-------------|-----|--------|----------|------|
| US-23-01 | 信心度彩色標籤 | `POST /api/chat`（response.confidenceLevel） | chat/ | chat.service.spec | — | — | Chatbot 主畫面 | JWT | ✅ |

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

| TC 編號 | 對應 FR | 對應 US | 說明 | NestJS 測試 | Python 測試 | Screen | Security | 狀態 |
|---------|---------|---------|------|-------------|-------------|--------|----------|------|
| TC-01-001 | FR-01 | US-01-01 | 正常登入 | auth.service.spec + E2E | — | — | JWT, bcrypt, 帳號鎖定 | ✅ |
| TC-01-002 | FR-01 | US-01-01 | 登入失敗（帳密錯誤/鎖定） | auth.service.spec + E2E | — | — | JWT, bcrypt, 帳號鎖定 | ✅ |
| TC-02-001 | FR-02 | US-02-01 | RAG 查詢回應 | chat.service.spec, rag-proxy.service.spec | test_rag_chain, test_retriever | — | JWT, Input Validation | ✅ |
| TC-03-001 | FR-05 | US-05-01 | 檔案上傳 | upload.controller.spec | test_upload_api | — | JWT, RBAC(admin), 50MB 限制 | ✅ |
| TC-04-001 | FR-04 | US-04-01 | 清洗任務執行 | clean.controller.spec | test_clean_api, test_detector | — | JWT, RBAC(admin), X-Internal-Token | ✅ |
| TC-05-001 | FR-18 | US-18-01 | 清洗審核瀏覽 | review.controller.spec | test_review_api | — | JWT, RBAC(admin/cleaner), X-Internal-Token | ✅ |
| TC-05-002 | FR-18 | US-18-03 | 內容手動編輯 | review.controller.spec | test_review_api | — | JWT, RBAC(admin/cleaner), X-Internal-Token | ✅ |
| TC-05-003 | FR-18 | US-18-04 | 標籤管理 | review.controller.spec | test_review_api | — | JWT, RBAC(admin/cleaner), X-Internal-Token | ✅ |
| TC-05-004 | FR-18 | US-18-05 | 任務批准 | review.controller.spec | test_review_integration | — | JWT, RBAC(admin/cleaner), X-Internal-Token | ✅ |
| TC-05-005 | FR-18 | US-18-06 | 送入 RAG | review.controller.spec | test_ingest_api, test_ingest_integration | — | JWT, RBAC(admin/reviewer), X-Internal-Token | ✅ |
| TC-05-006 | FR-18 | US-18-10, US-18-12 | Maker-Checker 送審與職責分離 | review.controller.spec | test_review_api, test_review_integration | — | JWT, RBAC, Maker-Checker | ✅ |
| TC-05-007 | FR-18 | US-18-11 | 退回任務流程 | review.controller.spec | test_review_api | — | JWT, RBAC(admin/reviewer), Maker-Checker | ✅ |
| TC-06-001 | FR-19 | US-19-01 | 清洗統計 | analytics.controller.spec | test_analytics_api | — | JWT, RBAC(admin/cleaner), X-Internal-Token | ✅ |
| TC-06-002 | FR-19 | US-19-03 | 時間軸統計 | analytics.controller.spec | test_analytics_api | — | JWT, RBAC(admin/cleaner), X-Internal-Token | ✅ |

---

## 4. 跨切面需求追溯

以下為非功能性需求與架構決策的追溯。

### 4.1 安全性需求

| 需求 | 實作機制 | 測試覆蓋 | ADR 參考 |
|------|---------|----------|----------|
| JWT 認證 | JwtAuthGuard 全域啟用 | auth.service.spec, E2E | — |
| RBAC 授權 | RolesGuard + @Roles 裝飾器 | roles.guard.spec | — |
| 密碼政策（資通安全「普」級） | IsStrongPassword validator | password-strength.validator.spec | [ADR-007](./architecture/ARCH.md#adr-007) |
| 內部 API 認證 | X-Internal-Token 中介軟體 | test_internal_auth_middleware | — |
| Rate Limiting | ThrottlerModule 60req/min | E2E | — |
| 輸入驗證 | ValidationPipe + class-validator | DTO spec 檔案（15+） | — |
| XSS 防護 | React 自動轉義 + helmet headers | — | — |
| SQL Injection | Prisma + SQLAlchemy 參數化查詢 | — | [ADR-003](./architecture/ARCH.md#adr-003) |

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
| ADR-001 | Monorepo (pnpm + Turborepo) | 全部 | [ARCH.md](./architecture/ARCH.md#adr-001) |
| ADR-002 | NestJS 統一 API Gateway | FR-04~09, FR-18~21 | [ARCH.md](./architecture/ARCH.md#adr-002) |
| ADR-003 | 雙 ORM 策略 | FR-01~09 + FR-04~09 | [ARCH.md](./architecture/ARCH.md#adr-003) |
| ADR-004 | Hybrid Search (Vector+BM25+RRF) | FR-02, FR-17 | [ARCH.md](./architecture/ARCH.md#adr-004) |
| ADR-005 | SSE 串流回應 | FR-02 | [ARCH.md](./architecture/ARCH.md#adr-005) |
| ADR-006 | 三前端應用 | FR-01~03, FR-04~09, FR-18~21 | [ARCH.md](./architecture/ARCH.md#adr-006) |
| ADR-007 | 資通安全「普」級密碼政策 | FR-01 | [ARCH.md](./architecture/ARCH.md#adr-007) |
| ADR-008 | SearXNG 搜尋引擎 | FR-02 | [ARCH.md](./architecture/ARCH.md#adr-008) |
| ADR-009 | Presidio PII 偵測 | FR-04 | [ARCH.md](./architecture/ARCH.md#adr-009) |
| ADR-010 | 階層式 Chunking 策略 | FR-17 | [ARCH.md](./architecture/ARCH.md#adr-010) |

---

## 5. 涵蓋度摘要

### 5.1 功能需求分布

| 類別 | FR 數量 | US 數量 | 狀態 |
|------|---------|---------|------|
| 已實作（✅） | 12 | 29 | Unit + Integration 覆蓋 |
| 規劃中（🔄） | 5 | 12 | 部分實作、持續強化 |
| 未來願景（🔮） | 6 | 6（未展開） | Phase 3 以後 |
| **合計** | **23** | **47** | — |

### 5.2 測試覆蓋統計

| 層級 | 檔案數 | 覆蓋範圍 |
|------|--------|---------|
| NestJS Unit (.spec.ts) | 42 | 全部已實作模組（auth, users, chat, prompts, audit, cleaning, health, websearch, llm, datasources） |
| Python rag-service (test_*.py) | 38 | RAG 核心管線、API 路由、整合測試、Maker-Checker 驗證 |
| Python data-pipeline (test_*.py) | 14 | 解析器、偵測器、去識別化策略 |
| React 前端 (*.test.tsx) | 13 | Admin 4 + Cleaner 5 + Chatbot 4（hooks + pages + components） |
| **合計** | **107** | 含前端測試框架（Vitest + RTL） |

### 5.3 追溯完整性

| 檢核項目 | 狀態 | 說明 |
|----------|------|------|
| 每個已實作 US 有對應 API 端點 | ✅ | 39/39 US 有明確端點（含 US-05-03、US-18-10/11/12） |
| 每個 API 端點有 NestJS 測試 | ✅ | 43 spec 檔案覆蓋全部 Controller/Service |
| 每個 FastAPI 路由有 Python 測試 | ✅ | 38 test 檔案覆蓋 API + 整合 |
| SRS §11 驗收案例有對應測試 | ✅ | 14/14 TC 全數有測試覆蓋（含 TC-05-006/007 Maker-Checker） |
| 每個 ADR 有對應 FR 引用 | ✅ | 10/10 ADR 標注影響範圍 |

---

## 6. Quality Gate 檢核

### 6.1 Phase 1（當前）

- [x] 所有 FR-01~09 + FR-16~21 有 Unit Test 覆蓋
- [x] 雙語言服務（NestJS + Python）皆有獨立測試
- [x] 驗收案例 TC-01~06 全數有對應測試
- [x] 安全性檢核項全數通過（SRS-T §11.3）
- [x] SEC-IDOR-01：Chat conversation ownership verification（`chat.service.ts:231-234`，防止 IDOR 跨用戶存取對話）
- [x] SEC-RATE-01：ThrottlerGuard APP_GUARD 全域 60req/min + auth 端點 per-endpoint 5req/min（`app.module.ts:52` + `auth.controller.ts`）
- [x] SEC-AUDIT-01：Audit interceptor 增強 — AuditEvent decorator + before/after 變更追蹤 + required 阻斷 + auditRead 敏感讀取（`audit-log.interceptor.ts` + `audit-actions.ts` + `audit-details.ts`）
- [ ] 行覆蓋率量測（NestJS ≥80%、Python ≥80%）— 規劃中

### 6.2 Phase 2（目標）

- [ ] 效能測試基準建立（k6 壓力測試）
- [x] 前端元件測試導入（Vitest + RTL）— 已完成 134 tests
- [x] E2E 測試基礎建設（supertest + rate-limit e2e）— 26/31 通過
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
| ARCH.md | 架構決策紀錄 | [architecture/ARCH.md](./architecture/ARCH.md) |
| test-strategy.md | 測試策略 | [testing/test-strategy.md](../02-testing/test-strategy.md) |
| API 文件索引 | 全部端點規格 | [api/README.md](./api/README.md) |

---

> **文件結束**
>
> 本文件為 ODA Cyber Konsult 的需求追溯矩陣，串連 User Story → 功能需求 → API → 模組 → 測試 的完整鏈路。
> 新增功能時，請同步更新對應的追溯列。
