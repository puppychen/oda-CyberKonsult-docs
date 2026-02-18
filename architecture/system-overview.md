# 系統架構概述

## 整體架構圖

```
                          +--------------------------+
                          |      使用者 / 管理者       |
                          +-----------+--------------+
                                      |
                    +-----------------+-----------------+
                    |                                   |
                    v                                   v
        +-------------------+   +-------------------+   +-------------------+
        |   Chatbot UI      |   |  Cleaner App      |   |  Admin Dashboard  |
        |   React 19        |   |  React 19         |   |  React 19         |
        |   port: 5174      |   |  port: 5175       |   |  port: 5173       |
        +--------+----------+   +--------+----------+   +--------+----------+
                 |                       |                       |
                 |  Vite Proxy           |  Vite Proxy           |  Vite Proxy
                 |  /api -> :4000        |  /api -> :4000        |  /api -> :4000
                 +----------------+------+-----------------------+
                                  |
                                  v
                      +-----------------------+
                      |     NestJS API        |
                      |     port: 4000        |
                      |                       |
                      |  +--- REST API -----+ |
                      |  | Upload / Clean   | |
                      |  | Tasks / Rules    | |
                      |  | Download         | |
                      |  +-----------------+ |
                      |                       |
                      |  +--- WebSocket ---+ |
                      |  | /ws/tasks/:id   | |
                      |  | /ws/all         | |
                      |  +-----------------+ |
                      +---------+---+---+---------+
                                |   |   |
                   +------------+   |   +------------+
                   |                |                 |
                   v                v                 v
        +-------------------+  +-----------+  +-------------------+
        |   PostgreSQL 17   |  | Gemini /  |  | RAG Service       |
        |   port: 5432      |  | OpenAI    |  | FastAPI           |
        |                   |  | API       |  | port: 8000        |
        |  Prisma Tables:   |  | (LLM生成) |  |                   |
        |  - users          |  +-----------+  | +--- BM25 (SQLite)|
        |  - conversations  |                 | +--- Reranker     |
        |  - settings       |  +-----------+  |                   |
        |  - web_search_    |  | SearXNG   |  +--------+----------+
        |    configs        |  | Docker    |           |
        |                   |  | port:8080 |           v
        |  SQLAlchemy/      |  | (網路搜尋) |  +-------------------+
        |  Alembic Tables:  |  +-----------+  |   Qdrant          |
        |  - files          |                 |   port: 6333/6334 |
        |  - rule_profiles  |                 |                   |
        |  - anon_rules     |                 |   向量嵌入儲存     |
        |  - tasks          |                 +-------------------+
        |  - task_files     |
        |  - audit_logs     |
        +-------------------+
```

---

## 元件說明

### 前端層

#### Admin Dashboard（管理後台）

| 項目 | 說明 |
|------|------|
| **技術** | React 19 + Vite 6 + Ant Design v6 |
| **連接埠** | 5173 |
| **用途** | 僅限系統管理員使用，負責訓練資料上傳、去識別化任務管理、規則設定、稽核日誌查看、系統設定 |
| **路由代理** | `/api/*` 轉發至 `localhost:4000` |

主要頁面：

- **首頁 (Home)** - 檔案上傳區域、快速啟動清洗、即時任務狀態監控
- **規則管理 (Rules)** - 規則設定檔 CRUD、規則匯入/匯出、預設規則設定
- **任務列表 (Tasks)** - 歷史任務瀏覽、進度追蹤、結果下載、清洗報告檢視

#### Chatbot UI（使用者聊天介面）

| 項目 | 說明 |
|------|------|
| **技術** | React 19 + Vite 6 + TailwindCSS v4 |
| **連接埠** | 5174 |
| **用途** | 所有角色（User / Consultant / Admin）的統一對話介面，支援 SSE 串流回應 |
| **路由代理** | `/api/*` 轉發至 `localhost:4000` |

#### Cleaner App（清洗審核管理介面）

| 項目 | 說明 |
|------|------|
| **技術** | React 19 + Vite 6 + Ant Design v6 + TanStack React Query |
| **連接埠** | 5175 |
| **用途** | 供資料清洗人員 (data_cleaner) 及系統管理員使用，負責清洗結果審核、來源資料瀏覽、知識庫管理 |
| **路由代理** | `/api/*` 轉發至 `localhost:4000` |

主要頁面：

- **總覽 (Dashboard)** - 資料管線漏斗（上傳→清洗→審核→批准→入庫）、清洗效能指標、知識庫概況、最近操作記錄
- **來源資料 (Files)** - 所有上傳檔案列表，支援搜尋、類型篩選、管線狀態追蹤（7 種狀態：未處理/清洗中/清洗失敗/待審核/已批准/已入庫/已退回）
- **任務列表 (Tasks)** - 清洗任務列表、進度追蹤、審核操作、送入 RAG
- **知識庫 (Knowledge Base)** - 三 Tab 管理：文件管理（以文件為核心的向量文件瀏覽）、語意搜尋（RAG 檢索品質測試）、來源分析（來源/標籤分佈）

### API 層

#### NestJS API（核心後端服務）

| 項目 | 說明 |
|------|------|
| **技術** | NestJS 11 + TypeScript 5.7+ + Prisma ORM |
| **連接埠** | 4000 |
| **用途** | 核心後端，處理認證、業務邏輯、資料持久化、清洗任務編排 |
| **ORM** | Prisma（管理 users、conversations、settings 等主表） |

職責範圍：

- RESTful API 端點（Upload、Clean、Tasks、Rules、Download）
- WebSocket 即時通知（任務進度推送）
- 檔案儲存管理
- 任務排程與狀態追蹤
- LLM 編排層（直接呼叫 Gemini/OpenAI API 進行回答生成）
- SearXNG 網路搜尋整合（RAG 結果不足時自動觸發）
- 上下文建構與提示詞管理
- 作為前端與 RAG 服務之間的編排者

#### RAG Service（檢索服務）

| 項目 | 說明 |
|------|------|
| **技術** | FastAPI + Qdrant |
| **連接埠** | 8000 |
| **用途** | 文件檢索（向量搜尋 + BM25 + Reranking），不含 LLM 回答生成 |
| **ORM** | SQLAlchemy + Alembic（管理 files、tasks、rules 等清洗相關表） |

職責範圍：

- 文件解析（PDF、DOCX、XLSX、CSV、TXT）
- PII 實體偵測（基於 Microsoft Presidio + spaCy）
- 資料去識別化處理（六種去識別化策略）
- 向量化與 Qdrant 索引
- RAG 檢索（/retrieve 端點：向量搜尋 + BM25 + 可選 Reranking）

> **注意**：LLM 回答生成已遷移至 NestJS LLM 模組。Python 端 LLM 僅供 Reranker 內部使用。

### 資料層

#### PostgreSQL 17

| 項目 | 說明 |
|------|------|
| **連接埠** | 5432 |
| **管理方式** | 本機 Docker 容器 |
| **用途** | 統一關聯式資料儲存 |

雙 ORM 架構（共用同一個 PostgreSQL 實例）：

- **Prisma**（NestJS 端）- 管理使用者、對話、系統設定等主業務表
- **SQLAlchemy + Alembic**（Python 端）- 管理檔案、任務、規則等清洗業務表

#### Qdrant 向量資料庫

| 項目 | 說明 |
|------|------|
| **REST 連接埠** | 6333 |
| **gRPC 連接埠** | 6334 |
| **管理方式** | 本機 Docker 容器 |
| **用途** | 文件嵌入向量的儲存與相似度檢索 |

---

## 資料清洗流程

資料清洗是本系統的核心功能之一，處理流程如下：

```
+----------+     +---------+     +-------------+     +------------+     +----------+
|          |     |         |     |             |     |            |     |          |
|  Upload  +---->+  Parse  +---->+ Detect PII  +---->+ Anonymize  +---->+  Output  |
|          |     |         |     |             |     |            |     |          |
+----------+     +---------+     +-------------+     +------------+     +----------+
    |                |                 |                   |                  |
    v                v                 v                   v                  v
 接收檔案        文件解析          PII 實體偵測        去識別化處理           輸出結果
 (PDF/DOCX/     (提取純文字)     (Presidio +        (mask/pseudonymize  (清洗後檔案
  XLSX/CSV/                       spaCy NER)        /generalize/...)    + JSON 報告)
  TXT)
```

### 各階段詳細說明

#### 1. Upload（檔案上傳）

- 系統管理員透過 Admin Dashboard 上傳待去識別化的訓練資料檔案
- NestJS API 接收 multipart/form-data 請求
- 檔案儲存至本機磁碟，建立 `files` 資料庫記錄
- 回傳檔案 ID 供後續操作引用

#### 2. Parse（文件解析）

- 依檔案類型使用對應解析器提取純文字內容
- **PDF**: pdfminer.six
- **DOCX**: python-docx
- **XLSX**: openpyxl
- **CSV/TXT**: 直接讀取
- 保留文件結構資訊以利後續回寫

#### 3. Detect PII（實體偵測）

- 使用 Microsoft Presidio Analyzer 進行 PII 偵測
- 搭配 spaCy NER 模型處理中文人名、地址等
- 自訂 Recognizer 處理台灣特有格式（身分證、手機、統編）
- 每個偵測結果包含實體類型、位置、信心分數

#### 4. Anonymize（去識別化處理）

- 根據管理員設定的規則（Rule Profile）進行去識別化
- 六種去識別化策略：mask、partial_mask、pseudonymize、generalize、keep_labeled、encrypt
- 假名替換使用一致性對映（同一人名在全文中替換為相同假名）
- 處理過程記錄於 `audit_logs` 表

#### 5. Output（結果輸出）

- 產出清洗後檔案（保持原格式）
- 產出 JSON 清洗報告（含所有偵測實體與處理細節）
- 支援單檔下載或 ZIP 批次打包下載

---

## 技術棧總覽

| 層級 | 技術選擇 | 版本 | 說明 |
|------|----------|------|------|
| **Monorepo** | pnpm workspaces + Turborepo | pnpm 9.15+ | TypeScript 專案管理與建置編排 |
| **後端 API** | NestJS + TypeScript | NestJS 11, TS 5.7+ | RESTful API + WebSocket 服務 |
| **ORM (Node)** | Prisma | - | NestJS 端資料庫存取 |
| **前端 (Admin)** | React + Vite + Ant Design | React 19, Vite 6, Ant Design v6 | 管理後台 |
| **前端 (Cleaner)** | React + Vite + Ant Design + TanStack Query | React 19, Vite 6, Ant Design v6 | 清洗審核管理介面 |
| **前端 (Chatbot)** | React + Vite + TailwindCSS | React 19, Vite 6, Tailwind v4 | 使用者聊天介面 |
| **Python 管理** | uv workspace | Python 3.12+ | Python 套件管理 |
| **RAG 服務** | FastAPI + Qdrant | - | 檢索服務（向量搜尋 + BM25 + Reranking） |
| **LLM 編排** | NestJS + @google/generative-ai + openai | - | NestJS 直接呼叫 Gemini/OpenAI 生成回答 |
| **網路搜尋** | SearXNG + Cheerio | Docker | RAG 不足時網路搜尋補充 |
| **ORM (Python)** | SQLAlchemy + Alembic | - | Python 端資料庫存取與遷移 |
| **PII 偵測** | Microsoft Presidio + spaCy | - | 個資偵測引擎 |
| **文件解析** | pdfminer.six, python-docx, openpyxl, markitdown | - | 多格式文件解析 |
| **關聯式資料庫** | PostgreSQL | 17 | 主資料儲存 |
| **向量資料庫** | Qdrant | - | 文件向量儲存與檢索 |
| **npm scope** | @oda-cyber/* | - | 所有 TypeScript 套件統一命名空間 |

---

## 目錄結構

```
oda-cyber-konsult/
├── apps/
│   ├── api/                    # NestJS 後端 API (port 4000)
│   │   ├── src/
│   │   │   ├── modules/        # 功能模組（upload, clean, tasks, rules, download）
│   │   │   ├── common/         # 共用元件（guards, pipes, filters）
│   │   │   └── main.ts
│   │   └── prisma/             # Prisma schema & migrations
│   ├── admin/                  # 管理後台 (port 5173)
│   │   ├── src/
│   │   │   ├── pages/          # 頁面元件（Home, Rules, Tasks）
│   │   │   ├── components/     # 共用 UI 元件
│   │   │   ├── hooks/          # 自訂 React Hooks
│   │   │   └── services/       # API 呼叫服務層
│   │   └── vite.config.ts
│   ├── cleaner/                 # 清洗審核管理介面 (port 5175)
│   │   ├── src/
│   │   │   ├── pages/          # 頁面元件（Dashboard, Files, Tasks, KnowledgeBase）
│   │   │   ├── components/     # 共用 UI 元件（Layout）
│   │   │   ├── hooks/          # 自訂 React Hooks
│   │   │   └── api/            # API 呼叫服務層
│   │   └── vite.config.ts
│   └── chatbot/                # 使用者聊天介面 (port 5174)
│       ├── src/
│       └── vite.config.ts
├── packages/
│   ├── shared-types/           # 前後端共用 TypeScript 型別
│   │   └── src/
│   │       ├── api-response.ts # ApiResponse<T> 定義
│   │       ├── cleaning.ts     # 清洗相關型別
│   │       └── index.ts
│   └── tsconfig/               # 共用 TypeScript 設定
├── python/
│   ├── data-pipeline/          # 資料清洗管線
│   │   ├── src/
│   │   │   ├── loaders/        # 檔案解析器（pdf, docx, xlsx, csv）
│   │   │   ├── analyzers/      # PII 偵測器
│   │   │   ├── anonymizers/    # 去識別化處理器
│   │   │   └── pipeline.py     # 管線編排
│   │   └── pyproject.toml
│   └── rag-service/            # RAG FastAPI 服務 (port 8000)
│       ├── src/
│       │   ├── api/            # FastAPI 路由
│       │   ├── models/         # SQLAlchemy 模型
│       │   ├── services/       # 業務邏輯
│       │   └── alembic/        # 資料庫遷移
│       └── pyproject.toml
├── docs/                       # 專案文件
│   ├── api/                    # API 文件
│   ├── architecture/           # 架構文件
│   └── setup/                  # 環境建置指南
├── turbo.json                  # Turborepo 建置設定
├── pnpm-workspace.yaml         # pnpm workspace 設定
└── package.json                # 根層級 package.json
```

---

## 服務間通訊

### Vite Proxy 規則

前端開發伺服器透過 Vite proxy 轉發 API 請求，避免跨域問題：

```typescript
// apps/admin/vite.config.ts & apps/chatbot/vite.config.ts
export default defineConfig({
  server: {
    proxy: {
      '/api': {
        target: 'http://localhost:4000',
        changeOrigin: true,
      },
      '/ws': {
        target: 'ws://localhost:4000',
        ws: true,
      },
    },
  },
});
```

### 服務通訊矩陣

| 來源 | 目標 | 協定 | 路徑 |
|------|------|------|------|
| Admin Dashboard | NestJS API | HTTP (via Proxy) | `/api/v1/*` |
| Admin Dashboard | NestJS API | WebSocket (via Proxy) | `/ws/*` |
| Chatbot UI | NestJS API | HTTP (via Proxy) | `/api/v1/*` |
| Cleaner App | NestJS API | HTTP (via Proxy) | `/api/v1/*` |
| NestJS API | RAG Service | HTTP | `http://localhost:8000/api/v1/rag/retrieve` |
| NestJS API | Gemini/OpenAI API | HTTPS | LLM 回答生成（直接呼叫） |
| NestJS API | SearXNG | HTTP | `http://searxng:8080/search`（Docker 內部） |
| NestJS API | PostgreSQL | TCP | `localhost:5432` |
| RAG Service | PostgreSQL | TCP | `localhost:5432` |
| RAG Service | Qdrant | HTTP/gRPC | `localhost:6333` / `6334` |

---

## 資料庫架構

系統採用雙 ORM 策略，兩端共用同一個 PostgreSQL 17 實例，但各自管理不同的資料表：

### Prisma 管理的資料表（NestJS 端）

負責主業務資料：

- `users` - 系統使用者
- `conversations` - 對話記錄
- `messages` - 訊息記錄
- `settings` - 系統設定

### SQLAlchemy + Alembic 管理的資料表（Python 端）

負責清洗業務資料：

- `files` - 上傳檔案記錄
- `rule_profiles` - 去識別化規則設定檔
- `anonymization_rules` - 具體去識別化規則（隸屬於 rule_profiles）
- `tasks` - 清洗任務
- `task_files` - 任務與檔案的關聯（多對多）
- `audit_logs` - 操作稽核日誌

### 資料表關聯

```
rule_profiles  1---N  anonymization_rules
tasks          N---M  files              (透過 task_files)
tasks          1---N  audit_logs
```

> **注意事項**：兩端的遷移（migration）各自獨立管理。Prisma 使用 `prisma migrate`，Python 端使用 `alembic upgrade head`。部署時需分別執行兩端的遷移指令。
