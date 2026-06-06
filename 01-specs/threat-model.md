# 威脅模型 (Threat Model)

> **ODA Cyber Konsult - 資安助手 RAG 系統**
>
> 文件版本：1.4.0
> 建立日期：2026-03-01
> 最後更新：2026-06-06
> 文件類型：威脅模型（Threat Model）

---

## 1. 文件目的

本文件以 STRIDE 方法論分析系統面臨的安全威脅，識別攻擊面與風險等級，並追蹤緩解措施的實作狀態。適用對象為安全審查人員、架構師與運維團隊。

---

## 2. 資料分類

### 2.1 PII 資料流

使用者上傳的文件可能包含個人可識別資訊（PII），系統透過去識別化管線處理。

```
文件上傳 → 格式解析（PDF/DOCX/XLSX）
         → Presidio + spaCy 偵測（20 種 PII 實體）
         → 去識別化策略套用（mask/partial_mask/pseudonymize/generalize/keep_labeled/encrypt）
         → 清洗後輸出（審核 → 批准 → 可選送入 RAG 知識庫）
```

| 資料類型 | 敏感等級 | 儲存位置 | 保護措施 |
|----------|---------|---------|---------|
| 原始上傳檔案 | 高 | 檔案系統 uploads/ | 存取控制 + RBAC |
| PII 偵測結果 | 高 | PostgreSQL (tasks/task_files) | 資料庫存取控制 |
| 去識別化後檔案 | 中 | 檔案系統 outputs/ | 審核流程把關 |
| 知識庫向量 | 低 | Qdrant | 僅含去識別化後內容 |

### 2.2 對話資料流

使用者透過 Chatbot UI 進行資安諮詢，系統整合 RAG 檢索與 LLM 生成回應。

```
使用者查詢 → Query 改寫（多輪歷史注入）
           → RAG 檢索（Qdrant 向量 + BM25 關鍵字）
           → 分數過濾（RRF < 0.005 / cosine < 0.3 過濾）
           → 可選 SearXNG 網路搜尋補充
           → LLM 生成（Gemini / OpenAI）
           → SSE 串流回應
```

| 資料類型 | 敏感等級 | 儲存位置 | 保護措施 |
|----------|---------|---------|---------|
| 使用者查詢 | 中 | PostgreSQL (messages) | JWT 認證 + 使用者隔離 |
| 對話歷史 | 中 | PostgreSQL (conversations/messages) | RBAC + 使用者僅存取自身對話 |
| LLM API 請求 | 中 | 外部 API（Google/OpenAI） | HTTPS 傳輸加密 |
| 檢索結果 | 低 | 記憶體（不持久化） | 請求級生命週期 |

### 2.3 認證資料

| 資料類型 | 敏感等級 | 儲存位置 | 保護措施 |
|----------|---------|---------|---------|
| 密碼 hash | 極高 | PostgreSQL (users.password) | bcrypt 12 rounds |
| JWT Access Token | 高 | 客戶端記憶體 | 短效期 + HTTPS Only |
| JWT Refresh Token | 高 | 客戶端 / PostgreSQL | 單次使用 + 過期機制 |
| 密碼歷史 hash | 高 | PostgreSQL (password_histories) | bcrypt 12 rounds + 僅存 2 代 |
| X-Internal-Token | 高 | 環境變數 (.env) | HMAC 驗證 + 不對外暴露 |

---

## 3. STRIDE 威脅分析

### 3.1 威脅矩陣

| 威脅類別 | 威脅描述 | 攻擊情境 | 現有緩解措施 | 狀態 |
|----------|---------|---------|-------------|------|
| **Spoofing（偽冒）** | 攻擊者冒充合法使用者存取系統 | 竊取或偽造 JWT Token、暴力破解密碼 | JWT 認證（access + refresh token）、bcrypt 12 rounds 密碼雜湊、帳號鎖定（5 次失敗鎖 15 分鐘） | ✅ 已緩解 |
| **Tampering（竄改）** | 攻擊者竄改請求資料或資料庫內容 | SQL Injection、XSS、API 參數竄改、ZIP 路徑穿越（Zip Slip） | ValidationPipe + class-validator 輸入驗證、Prisma/SQLAlchemy 參數化查詢、HMAC 內部 API 簽章、ZIP 路徑穿越驗證 | ✅ 已緩解 |
| **Repudiation（否認）** | 使用者否認曾執行特定操作 | 刪除清洗任務後否認、修改審核結果後否認 | AuditLog Interceptor 全域攔截記錄、稽核日誌含使用者 ID + 時間戳 + 操作詳情、日誌不可刪除（僅 admin 可查詢匯出） | ✅ 已緩解 |
| **Information Disclosure（資訊洩漏）** | 敏感資料未經授權被存取 | PII 外洩、未授權存取他人對話、內部 API Token 洩漏 | PII 去識別化管線（Presidio + 20 種實體）、RBAC Guard 角色存取控制、X-Internal-Token 內部 API 認證、使用者僅能存取自身對話 | ✅ 已緩解 |
| **Denial of Service（阻斷服務）** | 攻擊者耗盡系統資源導致服務不可用 | 大量請求灌爆 API、上傳超大檔案、LLM API 配額耗盡、ZIP bomb 壓縮炸彈 | ThrottlerModule 60 req/min 限流、50MB 檔案上傳大小限制、Nginx 反向代理層額外防護、ZIP 解壓縮安全驗證 | ✅ 已緩解 |
| **Elevation of Privilege（權限提升）** | 低權限使用者存取高權限功能 | 一般使用者存取管理功能、繞過密碼變更要求 | RBAC Guard + @Roles 裝飾器強制角色檢查、PasswordChangeRequiredGuard 全域攔截過期密碼、前端路由守衛 + 後端雙重驗證 | ✅ 已緩解 |
| **LLM Prompt Injection（提示注入）** | 攻擊者透過使用者輸入或 RAG 文件夾帶指令操控 LLM | 直接注入（覆寫系統提示、越獄）、間接注入（被污染的知識庫文件/網路搜尋結果夾帶指令）、提示洩漏（誘導吐出系統提示）、跨模式越權（誘導 LLM 回應超出角色模式範圍） | 系統提示與使用者輸入分離（結構化 prompt）、RAG 來源限定為經 Maker-Checker 審核的知識庫、SearXNG 結果作為「參考資料」而非指令注入點、回應模式由後端 ChatModeGuard 強制（非 LLM 自決） | ⚠️ 部分緩解（見 §3.4） |

### 3.2 STRIDE 深度分析

#### S — Spoofing（偽冒）

| 攻擊向量 | 風險等級 | 緩解措施 | 實作位置 |
|----------|---------|---------|---------|
| JWT Token 竊取 | 高 | Access Token 短效期、Refresh Token 單次使用 | `apps/api/src/modules/auth/` |
| 密碼暴力破解 | 中 | bcrypt 12 rounds + 帳號鎖定（5 次 / 15 分鐘） | `apps/api/src/modules/auth/services/auth.service.ts` |
| Token 偽造 | 高 | JWT_SECRET >= 64 字元 + HS256 簽章驗證 | `apps/api/src/common/guards/jwt-auth.guard.ts` |
| 密碼重複使用 | 中 | password_histories 2 代不重複檢查 | `apps/api/src/modules/auth/services/auth.service.ts` |

#### T — Tampering（竄改）

| 攻擊向量 | 風險等級 | 緩解措施 | 實作位置 |
|----------|---------|---------|---------|
| SQL Injection | 高 | Prisma + SQLAlchemy 參數化查詢 | ORM 層自動防護 |
| API 參數竄改 | 中 | ValidationPipe + class-validator DTO 驗證 | `apps/api/src/common/` |
| 內部 API 偽造 | 高 | X-Internal-Token HMAC 驗證 | `python/rag-service/middleware/` |
| 檔案類型偽裝 | 中 | MIME type 檢查 + 副檔名白名單 | FastAPI upload 端點 |
| ZIP 路徑穿越（Zip Slip） | 高 | ZIP 內所有檔案路徑不得包含 `../` 等路徑穿越字元，違規直接拒絕解壓 | FastAPI ZIP 上傳處理 |
| CORS/CSP 配置不當 | 中 | CORS 白名單配置、CSP 內容安全政策（Phase 2 規劃中） | 基礎設施層 |

#### R — Repudiation（否認）

| 攻擊向量 | 風險等級 | 緩解措施 | 實作位置 |
|----------|---------|---------|---------|
| 操作否認 | 中 | AuditLog Interceptor 自動記錄所有 API 操作 | `apps/api/src/common/interceptors/audit-log.interceptor.ts` |
| 清洗結果篡改 | 中 | cleaning_audit_logs 記錄完整清洗歷程 | `python/data-pipeline/models/` |
| 稽核日誌竄改 | 高 | 日誌為 append-only，無刪除 API | `apps/api/src/modules/audit/` |

#### I — Information Disclosure（資訊洩漏）

| 攻擊向量 | 風險等級 | 緩解措施 | 實作位置 |
|----------|---------|---------|---------|
| PII 原始資料外洩 | 極高 | Presidio 偵測 + 6 種去識別化策略 | `python/data-pipeline/anonymizer/` |
| 跨使用者對話存取 | 高 | 查詢條件強制綁定 userId | `apps/api/src/modules/chat/services/chat.service.ts` |
| 內部服務暴露 | 高 | FastAPI/Qdrant/SearXNG 僅內部存取 | Docker 網路隔離 + 防火牆 |
| 錯誤訊息洩漏 | 低 | HttpExceptionFilter 統一格式化，不暴露 stack trace | `apps/api/src/common/filters/` |

#### D — Denial of Service（阻斷服務）

| 攻擊向量 | 風險等級 | 緩解措施 | 實作位置 |
|----------|---------|---------|---------|
| API 洪水攻擊 | 高 | ThrottlerModule 60 req/min | `apps/api/src/app.module.ts` |
| 大檔案上傳 | 中 | 50MB 上傳限制 | NestJS MulterModule 設定 |
| LLM API 耗盡 | 中 | 請求排隊 + 錯誤處理 + 配額監控 | `apps/api/src/modules/llm/` |
| Qdrant 記憶體耗盡 | 低 | Collection 向量數量監控 | 運維監控（規劃中） |
| ZIP bomb（壓縮炸彈） | 中 | ZIP 解壓縮前驗證壓縮比 ≤ 20:1、檔案數 ≤ 100、解壓後總大小 ≤ 200MB | FastAPI ZIP 上傳處理 |

#### E — Elevation of Privilege（權限提升）

| 攻擊向量 | 風險等級 | 緩解措施 | 實作位置 |
|----------|---------|---------|---------|
| 角色繞過 | 高 | RolesGuard + @Roles 裝飾器 | `apps/api/src/common/guards/roles.guard.ts` |
| 密碼過期繞過 | 中 | PasswordChangeRequiredGuard 全域攔截 | `apps/api/src/common/guards/password-change-required.guard.ts` |
| 前端路由繞過 | 低 | 前端守衛 + 後端 Guard 雙重驗證 | 前端 router + 後端 Guard |

### 3.3 Maker-Checker 審核流程 — STRIDE 分析

Maker-Checker 職責分離機制引入獨立攻擊面，以下為各威脅類別的分析。

| 威脅類別 | 攻擊向量 | 風險等級 | 緩解措施 | 實作位置 |
|----------|---------|---------|---------|---------|
| **Spoofing** | 送審者偽造身份繞過職責分離（如 cleaner 冒充 reviewer 批准自己的任務） | 高 | JWT `@CurrentUser` 由伺服器注入操作者 ID，不接受客戶端傳入；submitted_by / approved_by 皆由後端寫入 | `apps/api/src/modules/cleaning/controllers/review.controller.ts`、`python/rag-service/src/rag_service/api/v1/review.py` |
| **Tampering** | 送審後篡改檔案內容（繞過審核結果） | 高 | `_FROZEN_STATUSES` 凍結機制：任務進入 `review_requested` / `approved` / `ingested` 狀態後，檔案內容不可編輯（API 回傳 400） | `python/rag-service/src/rag_service/api/v1/review.py` |
| **Repudiation** | 操作者否認審核決定（批准或退回） | 中 | 完整稽核軌跡：`submitted_by/at`、`approved_by/at`、`edited_by/at`、`reviewed_by/at` 皆記錄於 task/task_files 表；cleaning_audit_logs 記錄所有操作 | `python/rag-service/src/rag_service/db/models.py` |
| **Info Disclosure** | 透過 reject 理由洩漏 PII 資訊 | 低 | reject 理由由 reviewer 手動輸入，不含系統生成的 PII 內容；稽核日誌僅限 admin 存取 | 流程設計 + RBAC |
| **Elevation** | cleaner 直接呼叫 approve/ingest API 繞過角色限制 | 高 | NestJS method-level `@Roles('admin', 'data_reviewer')` Guard 強制檢查；FastAPI 端同步驗證 `user_role` | `apps/api/src/modules/cleaning/controllers/review.controller.ts` |

### 3.4 LLM / RAG 特有威脅 — Prompt Injection 分析

本系統核心為 RAG + LLM 問答（三層回應模式），LLM 攻擊面為 STRIDE 之外的獨立風險類別（對應 OWASP LLM Top 10 之 LLM01: Prompt Injection），於 v1.4.0 補入。

| 攻擊向量 | 風險等級 | 說明 | 緩解措施 | 狀態 | 實作位置 |
|----------|---------|------|---------|------|---------|
| 直接提示注入 / 越獄 | 高 | 使用者於對話輸入「忽略先前指令」「你現在是…」覆寫系統提示，誘導 LLM 脫離資安顧問角色或洩漏系統提示 | 系統提示與使用者輸入以結構化分層組裝；回應模式參數（topK/temperature/maxTokens/rerank）由後端依角色固定，LLM 不可自選 | ⚠️ 部分緩解 | `apps/api/src/modules/chat/services/chat.service.ts` |
| 間接提示注入（RAG 文件夾帶） | 高 | 被污染的知識庫文件內含「對 AI 的指令」，於檢索後注入 prompt 操控回答 | 知識庫文件須經 Maker-Checker（送審者 ≠ 審批者）審核才能 ingest，惡意文件不易進入；建議再加「檢索內容以引用區塊包裹、明示為資料非指令」 | ⚠️ 部分緩解（依賴審核流程） | `python/rag-service` ingest + `chat.service.ts` 組裝 |
| 網路搜尋結果注入 / SSRF | 中 | SearXNG 補充結果含惡意指令或誘導 LLM 抓取內部資源 | SearXNG 僅內部存取、結果作為「參考資料」標示；WebFetcher 限定外部 URL | ⚠️ 部分緩解 | `apps/api/src/modules/websearch/` |
| 跨模式越權（誘導逾越角色模式） | 中 | 誘導 LLM 提供超出該角色 mode 範圍的深度（如 user 誘導取得 expert 級法規分析） | 回應模式授權由 `ChatModeGuard` + `ROLE_MODE_MATRIX` 後端強制（非 LLM 自決），即使 LLM 被誘導，mode 參數已在伺服器端鎖定 | ✅ 已緩解 | `apps/api/src/modules/chat/guards/chat-mode.guard.ts`、`policies/role-mode.policy.ts` |
| 提示洩漏（System Prompt Leak） | 低 | 誘導 LLM 吐出系統提示或內部設定 | 系統提示不含機密（無金鑰/內部路徑）；提示模板由 prompts 模組管理 | ⚠️ 殘留風險（POC 可接受） | `apps/api/src/modules/prompts/` |

**結論**：跨模式越權已由後端 `ChatModeGuard` 硬性緩解（攻擊面驗證已確認 it_user 無法取得 expert 模式）；直接/間接注入屬**部分緩解**，主要依賴「知識庫經 Maker-Checker 審核」與「mode 後端強制」兩道結構性防線，建議 Phase 2 補強檢索內容的指令/資料分離標記與輸出側過濾。

---

## 4. 攻擊面分析

### 4.1 NestJS API（Port 3051）

| 項目 | 說明 |
|------|------|
| 暴露方式 | 對外（透過 Nginx 反向代理） |
| 攻擊面 | REST API 端點（認證、聊天、管理、代理轉發） |
| 認證機制 | JWT Bearer Token（全域 JwtAuthGuard） |
| 主要威脅 | API 濫用、JWT 竊取、輸入注入 |
| 緩解措施 | ThrottlerModule、ValidationPipe、JwtAuthGuard、RolesGuard |

### 4.2 FastAPI RAG Service（Port 3502）

| 項目 | 說明 |
|------|------|
| 暴露方式 | 僅內部（Docker 網路 / localhost） |
| 攻擊面 | RAG 檢索、清洗管線、檔案解析 |
| 認證機制 | X-Internal-Token HMAC 驗證 |
| 主要威脅 | 內部 API 偽造、惡意檔案上傳、資源耗盡 |
| 緩解措施 | 內部 Token 認證、檔案類型白名單、上傳大小限制 |

### 4.3 Qdrant（Port 6333）

| 項目 | 說明 |
|------|------|
| 暴露方式 | 僅內部（Docker 網路） |
| 攻擊面 | 向量資料庫 REST API |
| 認證機制 | 無（依賴網路隔離） |
| 主要威脅 | 未授權存取向量資料、資料刪除 |
| 緩解措施 | Docker 網路隔離、防火牆規則、正式環境不對外開放 |

### 4.4 PostgreSQL（Port 5432）

| 項目 | 說明 |
|------|------|
| 暴露方式 | 僅內部（Docker 網路） |
| 攻擊面 | 資料庫連線 |
| 認證機制 | 帳號密碼認證 |
| 主要威脅 | 資料庫直接存取、資料外洩 |
| 緩解措施 | 強密碼、網路隔離、ORM 參數化查詢 |

### 4.5 SearXNG（Port 8080）

| 項目 | 說明 |
|------|------|
| 暴露方式 | 僅 Docker 內部通訊 |
| 攻擊面 | 元搜尋引擎 API |
| 認證機制 | 無（僅內部存取） |
| 主要威脅 | 搜尋結果注入、SSRF |
| 緩解措施 | Docker 網路隔離、正式環境不對外開放 port |

### 4.6 三個前端 React 應用

| 應用 | Port (dev) | 暴露方式 | 主要威脅 |
|------|-----------|---------|---------|
| Admin Dashboard | 5501 | 對外（Nginx 靜態） | XSS、CSRF、未授權存取管理功能 |
| Chatbot UI | 5502 | 對外（Nginx 靜態） | XSS、對話資料洩漏 |
| Cleaner App | 5503 | 對外（Nginx 靜態） | XSS、清洗資料未授權存取 |

**共通緩解措施**：
- React 自動轉義防 XSS
- Helmet HTTP headers
- 前端路由守衛 + 後端 RBAC Guard 雙重驗證
- Vite proxy 統一指向 NestJS API

---

## 5. 風險矩陣

### 5.1 風險評估準則

**可能性等級**：

| 等級 | 說明 |
|------|------|
| 高 | 攻擊工具公開可用，攻擊門檻低 |
| 中 | 需要特定知識或內部資訊 |
| 低 | 需要高度專業或物理存取 |

**影響等級**：

| 等級 | 說明 |
|------|------|
| 嚴重 | PII 大量外洩、系統完全失控 |
| 高 | 單一使用者資料外洩、服務中斷 > 1 小時 |
| 中 | 功能受損、短暫服務降級 |
| 低 | 資訊揭露有限、使用者體驗受影響 |

### 5.2 風險矩陣圖

|  | **可能性 — 低** | **可能性 — 中** | **可能性 — 高** |
|--|-----------------|-----------------|-----------------|
| **影響 — 嚴重** | 高風險 | 極高風險 | 極高風險 |
| **影響 — 高** | 中風險 | 高風險 | 極高風險 |
| **影響 — 中** | 低風險 | 中風險 | 高風險 |
| **影響 — 低** | 可接受 | 低風險 | 中風險 |

### 5.3 風險分布

| 風險項目 | 影響 | 可能性 | 風險等級 | 緩解狀態 |
|----------|------|--------|---------|---------|
| PII 原始資料外洩 | 嚴重 | 低 | 高 | ✅ Presidio 去識別化 + RBAC |
| JWT Token 洩漏 | 高 | 中 | 高 | ✅ 短效期 + Refresh 機制 |
| SQL Injection | 高 | 低 | 中 | ✅ ORM 參數化查詢 |
| API 洪水攻擊 | 中 | 高 | 高 | ✅ ThrottlerModule 60 req/min |
| 密碼暴力破解 | 高 | 中 | 高 | ✅ bcrypt + 帳號鎖定 |
| 內部 API 偽造 | 高 | 低 | 中 | ✅ X-Internal-Token HMAC |
| Qdrant 未授權存取 | 中 | 低 | 低 | ✅ Docker 網路隔離 |
| XSS 攻擊 | 中 | 中 | 中 | ✅ React 自動轉義 + Helmet |
| 權限提升 | 高 | 低 | 中 | ✅ RBAC Guard 雙重驗證 |
| LLM API 配額耗盡 | 中 | 中 | 中 | ⚠️ 部分緩解（需加強監控） |
| ZIP bomb 壓縮炸彈 | 高 | 中 | 高 | ✅ 壓縮比 / 檔案數 / 總大小三重驗證 |
| ZIP 路徑穿越（Zip Slip） | 高 | 中 | 高 | ✅ 路徑穿越字元驗證 + 拒絕解壓 |

---

## 6. 緩解措施追蹤表

| 編號 | 威脅類別 | 緩解措施 | 實作狀態 | 負責模組 | 驗證方式 |
|------|---------|---------|---------|---------|---------|
| M-001 | Spoofing | JWT 認證（access + refresh token） | ✅ 已實作 | auth/ | auth.service.spec + E2E |
| M-002 | Spoofing | bcrypt 12 rounds 密碼雜湊 | ✅ 已實作 | auth/ | auth.service.spec |
| M-003 | Spoofing | 帳號鎖定（5 次失敗 / 15 分鐘） | ✅ 已實作 | auth/ | auth.service.spec |
| M-004 | Spoofing | 密碼歷史 2 代不重複 | ✅ 已實作 | auth/ | auth.service.spec |
| M-005 | Spoofing | 密碼複雜度（8 碼 + 大小寫 + 數字 + 特殊字元） | ✅ 已實作 | common/validators/ | password-strength.validator.spec |
| M-006 | Tampering | ValidationPipe + class-validator | ✅ 已實作 | common/ | DTO spec（15+ 檔案） |
| M-007 | Tampering | Prisma 參數化查詢 | ✅ 已實作 | prisma/ | ORM 內建防護 |
| M-008 | Tampering | SQLAlchemy 參數化查詢 | ✅ 已實作 | Python ORM | ORM 內建防護 |
| M-009 | Tampering | X-Internal-Token HMAC 驗證 | ✅ 已實作 | FastAPI middleware | test_internal_auth_middleware |
| M-010 | Repudiation | AuditLog Interceptor | ✅ 已實作 | common/interceptors/ | audit-log.interceptor.spec |
| M-011 | Repudiation | 稽核日誌查詢與匯出 | ✅ 已實作 | audit/ | audit-query.dto.spec |
| M-012 | Info Disclosure | Presidio PII 偵測（20 種實體） | ✅ 已實作 | data-pipeline | test_detector, test_recognizers |
| M-013 | Info Disclosure | 6 種去識別化策略 | ✅ 已實作 | data-pipeline | test_anonymizer, test_strategies |
| M-014 | Info Disclosure | RBAC Guard 角色控制 | ✅ 已實作 | common/guards/ | roles.guard.spec |
| M-015 | Info Disclosure | HttpExceptionFilter 統一錯誤格式 | ✅ 已實作 | common/filters/ | 不暴露 stack trace |
| M-016 | DoS | ThrottlerModule 60 req/min | ✅ 已實作 | app.module.ts | E2E 驗證 |
| M-017 | DoS | 50MB 檔案上傳限制 | ✅ 已實作 | NestJS MulterModule | 上傳測試 |
| M-018 | EoP | RolesGuard + @Roles 裝飾器 | ✅ 已實作 | common/guards/ | roles.guard.spec |
| M-019 | EoP | PasswordChangeRequiredGuard | ✅ 已實作 | common/guards/ | password-change-required.guard.spec |
| M-020 | DoS | LLM API 配額監控 | ⚠️ 規劃中 | llm/ | Phase 2 |
| M-021 | Multiple | Nginx WAF / 進階防護 | ⚠️ 規劃中 | 基礎設施 | Phase 2 |
| M-022 | Spoofing | 2FA / MFA 雙因素認證 | 🔮 未來 | auth/ | Phase 3 |
| M-023 | Spoofing | Maker-Checker 職責分離（submitted_by ≠ approved_by） | ✅ 已實作 | cleaning/ review API | test_review_api, test_review_integration |
| M-024 | Tampering | 送審後凍結機制（_FROZEN_STATUSES） | ✅ 已實作 | FastAPI review.py | test_review_api |
| M-025 | Repudiation | Maker-Checker 完整稽核軌跡 | ✅ 已實作 | DB models (task/task_files) | test_review_integration |
| M-026 | EoP | Maker-Checker method-level @Roles Guard | ✅ 已實作 | review.controller.ts | review.controller.spec |
| M-027 | DoS | ZIP 解壓縮安全驗證（壓縮比 ≤ 20:1、檔案數 ≤ 100、解壓後總大小 ≤ 200MB） | ✅ 已實作 | FastAPI ZIP 上傳處理 | test_zip_bomb_protection |
| M-028 | Tampering | ZIP 路徑穿越驗證（拒絕含 `../` 路徑的檔案） | ✅ 已實作 | FastAPI ZIP 上傳處理 | test_zip_slip_protection |
| M-029 | Tampering | CORS 白名單 + CSP 內容安全政策 | 📋 規劃中 | 基礎設施 | Phase 2 |
| M-030 | Prompt Injection | 回應模式後端強制（ChatModeGuard + ROLE_MODE_MATRIX），LLM 不可自選 mode，防跨模式越權 | ✅ 已實作 | chat/guards/chat-mode.guard.ts、policies/role-mode.policy.ts | chat-mode.guard.spec、cleaning-authz.spec |
| M-031 | Prompt Injection | RAG 知識庫 ingest 須經 Maker-Checker 審核，降低間接注入（被污染文件夾帶指令）風險 | ⚠️ 部分緩解 | cleaning review API + rag-service ingest | test_review_api |
| M-032 | Prompt Injection | 檢索內容指令/資料分離標記 + 輸出側過濾 | 📋 規劃中 | chat.service.ts | Phase 2 |

---

## 相關文件

| 文件 | 說明 | 路徑 |
|------|------|------|
| ARCH.md | 架構決策紀錄 | [architecture/ARCH.md](./architecture/ARCH.md) |
| SRS_TECHNICAL.md | 技術需求規格書（§11 安全性驗收） | [SRS_TECHNICAL.md](./SRS_TECHNICAL.md) |
| RTM.md | 需求追溯矩陣（§4.1 安全性需求） | [RTM.md](./RTM.md) |
| runbook.md | 運維手冊（§6 安全事件應變） | [../03-operations/runbook.md](../03-operations/runbook.md) |
| test-strategy.md | 測試策略（§6.3 安全性測試） | [../02-testing/test-strategy.md](../02-testing/test-strategy.md) |

---

## 版本歷史

| 版本 | 日期 | 變更說明 |
|------|------|---------|
| v1.0.0 | 2026-03-01 | 初版：STRIDE 威脅分析、攻擊面分析、風險矩陣、緩解措施追蹤表 |
| v1.1.0 | 2026-03-06 | 新增 Maker-Checker 審核流程 STRIDE 分析、M-023~M-026 緩解措施 |
| v1.2.0 | 2026-03-11 | 新增 ZIP bomb 與 Zip Slip 威脅分析及緩解措施（M-027、M-028） |
| v1.3.0 | 2026-03-16 | ML-15：PII 實體數量確認 20 種；新增 CORS/CSP 威脅分析（M-029） |
| v1.4.0 | 2026-06-06 | 新增 §3.4 LLM/RAG Prompt Injection 分析（OWASP LLM01）+ M-030~M-032；校準 STRIDE 表中 chat/auth service 實作路徑至 `services/` 子目錄 |

---

> **文件結束**
>
> 本文件為 ODA Cyber Konsult 的威脅模型。
> 新增威脅或緩解措施時，請同步更新 STRIDE 分析表與緩解措施追蹤表。
