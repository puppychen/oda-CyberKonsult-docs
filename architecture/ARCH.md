# 架構決策紀錄 (ADR)

> **ODA Cyber Konsult - 資安助手 RAG 系統**
>
> 文件版本：1.0.0
> 建立日期：2026-03-01
> 文件類型：架構決策紀錄（Architecture Decision Records）

---

## 文件目的

本文件以 ADR（Architecture Decision Record）格式，記錄系統關鍵架構決策的背景、理由與後果。每個 ADR 獨立且不可變（Immutable），新決策以新 ADR 追加或標記取代（Superseded）。

### 狀態定義

| 狀態 | 說明 |
|------|------|
| **Accepted** | 已採納並實作 |
| **Proposed** | 提議中，尚未實作 |
| **Superseded** | 已被後續 ADR 取代 |

### 相關文件

- 系統架構概述：[architecture/system-overview.md](./system-overview.md)
- RAG 系統藍圖：[architecture/rag-system-blueprint.md](./rag-system-blueprint.md)
- 技術規格書：[specs/SRS_TECHNICAL.md](../specs/SRS_TECHNICAL.md)

---

## ADR-001：Monorepo 架構（pnpm workspaces + Turborepo）

| 項目 | 內容 |
|------|------|
| **狀態** | Accepted |
| **日期** | 2026-01 |
| **決策者** | 架構團隊 |

### 背景

系統包含 NestJS API、三個 React 前端（Admin / Chatbot / Cleaner）、共用型別套件，需要統一管理。

### 決策

採用 **pnpm workspaces + Turborepo** 管理 Monorepo，npm scope 為 `@oda-cyber/*`。

### 理由

1. **共用型別**：`@oda-cyber/shared-types` 可被所有 TypeScript 套件直接引用，確保前後端型別一致
2. **統一建置**：Turborepo 提供 pipeline 定義與快取，`pnpm dev` 一鍵啟動全部服務
3. **原子性變更**：跨套件的 breaking change 可在同一 commit 中修復
4. **pnpm 效能**：symlink 機制避免 node_modules 重複安裝，磁碟與安裝時間大幅降低

### 後果

- 正面：開發者體驗提升，新成員 onboarding 只需一次 `pnpm install`
- 負面：CI/CD 需處理 Turborepo 的 cache invalidation；Docker 建置需考慮 context 範圍
- 注意：Python 服務（uv workspace）獨立於 pnpm，需分別管理依賴

---

## ADR-002：NestJS 作為統一 API Gateway

| 項目 | 內容 |
|------|------|
| **狀態** | Accepted |
| **日期** | 2026-01 |
| **決策者** | 架構團隊 |

### 背景

系統有多個後端服務（NestJS 主 API + FastAPI RAG/Cleaning），前端需要統一的 API 入口。

### 決策

NestJS 作為**統一 API Gateway**，前端僅與 NestJS (port 4000) 通訊。清洗/資料來源請求由 NestJS **代理轉發**至 FastAPI (port 8000)，保持 `/api/v1/` 前綴。

### 理由

1. **單一入口**：前端不需知道後端服務拓撲，簡化部署與安全控制
2. **統一認證**：JWT 驗證集中在 NestJS，FastAPI 端透過 `X-Internal-Token` 進行內部認證
3. **LLM 編排**：NestJS 直接呼叫 Gemini/OpenAI API，FastAPI 僅負責檢索
4. **彈性擴展**：後端服務可獨立 scale，前端無感知

### 後果

- 正面：安全邊界清晰，一個 port 對外
- 負面：NestJS 成為單點，需確保高可用；代理層增加延遲（~5ms）
- 權衡：清洗 API 代理轉發保持 `/api/v1/` 路徑，不需前端修改

---

## ADR-003：雙 ORM 策略（Prisma + SQLAlchemy）

| 項目 | 內容 |
|------|------|
| **狀態** | Accepted |
| **日期** | 2026-01 |
| **決策者** | 架構團隊 |

### 背景

系統有兩種技術棧（NestJS TypeScript + FastAPI Python），都需要存取同一 PostgreSQL 資料庫。

### 決策

- **NestJS 端**：Prisma ORM（管理 users、conversations、messages、prompt_templates、audit_logs、web_search_configs、password_histories）
- **Python 端**：SQLAlchemy + Alembic（管理 files、rule_profiles、anonymization_rules、tasks、task_files、cleaning_audit_logs、gdrive_*）
- 兩端共用同一 PostgreSQL 17 實例

### 理由

1. **技術棧適配**：Prisma 是 TypeScript 生態最佳 ORM；SQLAlchemy 是 Python 生態標準
2. **Migration 獨立**：`prisma migrate` 與 `alembic upgrade head` 各自管理，不互相干擾
3. **共用資料庫**：避免資料同步問題，兩端可透過 SQL JOIN 或 FK 關聯（如 `users.id`）

### 後果

- 正面：各技術棧使用最熟悉的 ORM，開發效率高
- 負面：部署時需分別執行兩端 migration；schema 變更需注意不衝突
- 注意：`_prisma_migrations` 與 `alembic_version` 兩張 migration tracking 表共存

---

## ADR-004：Hybrid Search（向量搜尋 + BM25 + Reranking）

| 項目 | 內容 |
|------|------|
| **狀態** | Accepted |
| **日期** | 2026-01 |
| **決策者** | RAG 團隊 |

### 背景

純向量搜尋對精確術語（如法條編號、ISO 標準號）的召回率不佳，需要補充關鍵字搜尋。

### 決策

採用 **Hybrid Search**：向量搜尋（Qdrant）+ BM25（SQLite FTS）+ 可選 Reranking，以 **RRF（Reciprocal Rank Fusion）** 融合排序。

### 理由

1. **互補性**：向量搜尋擅長語意匹配，BM25 擅長精確關鍵字匹配
2. **RRF 穩定**：無需調整權重超參數，天然抗噪
3. **Reranking 提升**：expert 模式啟用 Reranking（cross-encoder），topK=8 確保召回率
4. **分數過濾**：hybrid 模式 RRF < 0.005 自動過濾，vector 模式 cosine < 0.3 自動過濾

### 後果

- 正面：法規術語查詢召回率顯著提升
- 負面：BM25 索引需額外維護（SQLite 檔案）；Reranking 增加 1~2 秒延遲
- 相關文件：[architecture/rag-pipeline.md](./rag-pipeline.md)

---

## ADR-005：SSE 串流回應

| 項目 | 內容 |
|------|------|
| **狀態** | Accepted |
| **日期** | 2026-02 |
| **決策者** | 前後端團隊 |

### 背景

LLM 生成回答耗時 3~8 秒，使用者等待體驗差。需要逐字顯示回應。

### 決策

採用 **Server-Sent Events (SSE)** 實作串流回應，NestJS Chat 端點返回 `text/event-stream`。

### 備選方案

| 方案 | 優點 | 缺點 |
|------|------|------|
| **SSE** ✅ | HTTP 原生、自動重連、瀏覽器原生支援 | 單向通訊 |
| WebSocket | 雙向通訊 | 連線管理複雜、需心跳 |
| Long Polling | 最簡實作 | 延遲高、伺服器負載大 |

### 理由

1. **LLM 回應天然是單向串流**，不需雙向通訊
2. **HTTP 相容**：通過 Nginx、Load Balancer 無需特殊設定
3. **Chatbot UI 已實作 `EventSource` API**，瀏覽器原生支援自動重連

### 後果

- 正面：使用者感知延遲從 ~5 秒降至 < 0.5 秒（首個 token）
- 負面：WebSocket 已用於清洗任務進度，形成兩種即時通訊並存的狀態
- 注意：SSE 的 event type 分為 `message`（逐字）、`sources`（引用來源）、`done`（完成）

---

## ADR-006：三前端應用（Admin + Chatbot + Cleaner）

| 項目 | 內容 |
|------|------|
| **狀態** | Accepted |
| **日期** | 2026-02 |
| **決策者** | 產品與前端團隊 |

### 背景

系統有三種截然不同的使用場景：管理後台、AI 對話、清洗審核。

### 決策

拆分為三個獨立 React 應用，各自獨立部署。

| 應用 | Port | UI 框架 | 目標使用者 |
|------|------|---------|-----------|
| Admin Dashboard | 5173 | Ant Design v6 | 系統管理員 |
| Chatbot UI | 5174 | TailwindCSS v4 | 所有角色 |
| Cleaner App | 5175 | Ant Design v6 + TanStack Query | data_cleaner + admin |

### 理由

1. **關注點分離**：每個應用有獨立的路由、狀態管理、權限邏輯
2. **獨立部署**：可依使用頻率獨立 scale（如 Chatbot 高流量時）
3. **UI 框架適配**：Chatbot 需要輕量聊天介面（TailwindCSS），管理類需要表格/表單密集型 UI（Ant Design）
4. **安全隔離**：非授權角色無法存取對應應用（前端攔截 + 後端 Guard）

### 後果

- 正面：各應用可獨立演進，不互相干擾
- 負面：共用邏輯需抽至 `@oda-cyber/shared-types`；認證流程三個應用各自實作
- 注意：三個應用透過 Vite proxy 統一指向 NestJS API (port 4000)

---

## ADR-007：資通安全「普」級密碼政策

| 項目 | 內容 |
|------|------|
| **狀態** | Accepted |
| **日期** | 2026-02 |
| **決策者** | 安全團隊 |

### 背景

系統面向中小企業，需符合台灣《資通安全管理法》「普」級安全要求。

### 決策

實作以下密碼政策：

| 項目 | 規格 |
|------|------|
| 最低長度 | 8 碼 |
| 複雜度 | 大寫 + 小寫 + 數字 + 特殊字元 |
| 過期天數 | 90 天 |
| 歷史代數 | 2 代不重複 |
| 鎖定策略 | 連續 5 次失敗鎖定 15 分鐘 |
| 首次登入 | 強制變更密碼 |

### 理由

1. **法規合規**：資通安全管理法「普」級為中小企業基本要求
2. **bcrypt 12 rounds**：對抗暴力破解
3. **password_histories 表**：記錄歷史密碼 hash，防止重複使用
4. **PasswordChangeRequiredGuard**：全域攔截，確保過期密碼必須變更後才能使用系統

### 後果

- 正面：符合法規要求，提升系統安全性
- 負面：使用者體驗略有影響（定期換密碼）
- 實作位置：`apps/api/src/common/validators/IsStrongPassword.ts`、`apps/api/src/common/guards/`

---

## ADR-008：SearXNG 網路搜尋補充

| 項目 | 內容 |
|------|------|
| **狀態** | Accepted |
| **日期** | 2026-02 |
| **決策者** | RAG 團隊 |

### 背景

RAG 知識庫無法涵蓋所有資安知識（尤其是最新法規修正、新發現的漏洞等），需要即時網路搜尋補充。

### 決策

整合 **SearXNG**（開源元搜尋引擎），當 RAG 檢索結果不足時自動觸發網路搜尋，並將結果合併至 LLM 生成回答的上下文中。

### 備選方案

| 方案 | 優點 | 缺點 |
|------|------|------|
| **SearXNG** ✅ | 開源自建、無 API 成本、隱私保護 | 需自行部署 Docker |
| Google Custom Search | 品質穩定 | 收費、資料送出至第三方 |
| Bing Search API | 品質穩定 | 收費、資料送出至第三方 |

### 理由

1. **隱私保護**：查詢不離開內部網路（Docker 容器內部通訊）
2. **零成本**：無 API 調用費用
3. **可控性**：可自訂搜尋引擎來源、過濾規則
4. **NestJS WebFetcher**：搜尋結果的網頁內容由 NestJS Cheerio 爬取摘要

### 後果

- 正面：知識庫不足時仍能提供有參考價值的回答
- 負面：網路搜尋品質不如 Google；正式環境不對外開放 SearXNG port
- 配置：閾值可透過 Admin Dashboard Settings 頁面調整

---

## ADR-009：Microsoft Presidio + spaCy 實作 PII 偵測

| 項目 | 內容 |
|------|------|
| **狀態** | Accepted |
| **日期** | 2026-01 |
| **決策者** | 資料處理團隊 |

### 背景

系統核心功能之一為去識別化，需要高準確率的 PII 偵測引擎，且需支援台灣特有的實體格式。

### 決策

採用 **Microsoft Presidio Analyzer**（PII 偵測框架）+ **spaCy**（NLP/NER 模型），並自建 3 個台灣專屬 Recognizer。

### 理由

1. **業界標準**：Presidio 是 Microsoft 開源的企業級 PII 偵測框架
2. **可擴展**：透過 RecognizerRegistry 可自訂 Recognizer，已新增 TW_ID / TW_PHONE / TW_UNIFIED_BUSINESS_NO
3. **spaCy 中文支援**：搭配 `zh_core_web_sm` 模型處理中文人名、地址
4. **20 種實體**：涵蓋標準 PII（10 種）+ 台灣專屬（3 種）+ 企業機密（6 種）+ 地址辨識器

### 後果

- 正面：偵測準確率 > 95%，可持續擴充新實體類型
- 負面：spaCy 模型載入需時（首次 ~3 秒）；中文人名辨識存在 false positive（已做排除規則）
- 相關文件：[architecture/data-cleaning-workflow.md](./data-cleaning-workflow.md)

---

## ADR-010：階層式 Chunking 策略

| 項目 | 內容 |
|------|------|
| **狀態** | Accepted |
| **日期** | 2026-02 |
| **決策者** | RAG 團隊 |

### 背景

文件切塊（Chunking）直接影響 RAG 檢索品質。過大的 chunk 導致噪音，過小的 chunk 失去上下文。

### 決策

採用 **DocumentPreprocessor 階層式 Chunking**：

1. **標題偵測**：正則表達式偵測 Markdown 標題、法規條文編號
2. **語意邊界**：優先在段落、句子邊界切割
3. **階層參數調優**：依三層回應模式差異化
   - beginner：較大 chunk（保留完整段落），topK=3
   - standard：中等 chunk，topK=5
   - expert：較小 chunk（精細檢索），topK=8 + Reranking
4. **Metadata 保留**：每個 chunk 保留來源文件名、標題階層、頁碼

### 理由

1. **法規文件特性**：法規條文有明確的編號與階層結構，需保留
2. **回應模式適配**：不同模式需要不同粒度的檢索結果
3. **Metadata 追溯**：使用者可追溯回答的精確來源位置

### 後果

- 正面：法規條文檢索精度提升；使用者可追溯至具體條文
- 負面：Chunking 策略調優需要持續迭代
- 相關文件：[architecture/rag-enhancements.md](./rag-enhancements.md)

---

## ADR 索引

| ADR | 標題 | 狀態 | 相關 FR |
|-----|------|------|---------|
| ADR-001 | Monorepo 架構 | Accepted | 全域 |
| ADR-002 | NestJS API Gateway | Accepted | FR-01~09 |
| ADR-003 | 雙 ORM 策略 | Accepted | FR-04, FR-09 |
| ADR-004 | Hybrid Search | Accepted | FR-02, FR-17 |
| ADR-005 | SSE 串流回應 | Accepted | FR-02, FR-16 |
| ADR-006 | 三前端應用 | Accepted | FR-18, FR-19 |
| ADR-007 | 密碼政策 | Accepted | FR-01 |
| ADR-008 | SearXNG 網路搜尋 | Accepted | FR-02 |
| ADR-009 | Presidio PII 偵測 | Accepted | FR-04 |
| ADR-010 | 階層式 Chunking | Accepted | FR-02, FR-17 |

---

> **文件結束**
>
> 本文件為 ODA Cyber Konsult 的架構決策紀錄。
> 新增 ADR 請沿用編號序列（ADR-011+），並保持既有 ADR 不可變。
