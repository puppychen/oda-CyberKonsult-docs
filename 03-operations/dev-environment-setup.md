# CyberKonsult 開發測試環境設定指南

> 最後更新：2026-03-28
> 適用版本：ODA Cyber Konsult v1.x
> 目的：讓開發者或 AI 代理能在 10 分鐘內建立完整可運行的開發測試環境

---

## 一、環境需求

### 系統需求

| 項目 | 版本 | 安裝方式 |
|------|------|---------|
| Node.js | 22+（`.nvmrc` 鎖定） | `nvm install` |
| pnpm | 9.15+ | `corepack enable && corepack prepare pnpm@9.15.4 --activate` |
| Python | 3.12+（`.python-version` 鎖定） | pyenv 或系統安裝 |
| uv | 最新版 | `curl -LsSf https://astral.sh/uv/install.sh \| sh` |
| Docker | 最新版 | Desktop 或 CLI |
| Git | 2.x+ | 系統安裝 |

### 連接埠分配

| 服務 | Port | 用途 |
|------|------|------|
| NestJS API | 3051 | 統一 API Gateway |
| Admin Dashboard | 5501 | 管理後台（React + Ant Design） |
| Chatbot UI | 5502 | 聊天介面（React + TailwindCSS） |
| Cleaner App | 5503 | 清洗審核管理（React + Ant Design） |
| FastAPI RAG | 3502 | RAG 檢索 + 資料清洗 |
| PostgreSQL | 5432 | 關聯式資料庫 |
| Qdrant REST | 6333 | 向量資料庫 |
| Qdrant gRPC | 6334 | 向量資料庫（gRPC） |
| SearXNG | 8080 | 網路搜尋引擎（Docker 內部） |

---

## 二、快速建置（一鍵腳本）

```bash
# 在專案根目錄執行
bash scripts/setup-dev-env.sh
```

或依以下步驟手動建置：

---

## 三、手動建置步驟

### Step 1：安裝依賴

```bash
# Node.js
nvm install
nvm use

# pnpm
corepack enable
corepack prepare pnpm@9.15.4 --activate
pnpm install

# Python
cd python && uv sync && cd ..
```

### Step 2：環境變數

```bash
cp .env.example .env
```

必須設定的變數（其餘可用預設值）：

| 變數 | 說明 | 範例 |
|------|------|------|
| `DATABASE_URL` | PostgreSQL 連線 | `postgresql://postgres:postgres@localhost:5432/oda_cyber?schema=public` |
| `JWT_SECRET` | JWT 簽章密鑰 | 隨機 64 字元字串 |
| `INTERNAL_API_KEY` | NestJS→FastAPI 認證 | 隨機 64 字元 hex |
| `RAG_INTERNAL_API_KEY` | 同上（Python 端讀取） | 與 INTERNAL_API_KEY 相同 |
| `LLM_PROVIDER` | LLM 提供者 | `gemini` 或 `openai` |
| `RAG_GOOGLE_API_KEY` | Gemini API Key | Google AI Studio 取得 |
| `RAG_OPENAI_API_KEY` | OpenAI API Key | OpenAI Platform 取得 |
| `RAG_EMBEDDING_PROVIDER` | 嵌入向量提供者 | `openai`（建議）或 `gemini` |

### Step 3：啟動 Docker 基礎設施 + 確認本機 PostgreSQL

```bash
# PostgreSQL 17：本機運行（不由本專案啟動容器）
# 假設本機 5432 已就緒，例：
#   brew services start postgresql@17
# 或啟動其他 ECMap 子專案既有的 PG 容器
nc -z localhost 5432 || echo "請先啟動本機 PostgreSQL 17"
# dev-start.sh / setup-dev-env.sh 會自動 createdb oda_cyber（不存在時）

# Qdrant 向量資料庫
docker run -d --name oda-qdrant \
  -p 6333:6333 -p 6334:6334 \
  -v qdrant_data:/qdrant/storage \
  --restart unless-stopped \
  qdrant/qdrant:latest

# SearXNG 搜尋引擎
docker run -d --name oda-searxng \
  -p 8080:8080 \
  --restart unless-stopped \
  searxng/searxng:latest
```

### Step 4：資料庫初始化

```bash
# Prisma Migration（NestJS 端表）
cd apps/api
pnpm prisma:generate
dotenv -e ../../.env -- npx prisma migrate dev
cd ../..

# Alembic Migration（Python 端表）
cd python/rag-service
uv run alembic upgrade head
cd ../..

# Seed 預設資料（7 使用者 + 8 提示詞範本 + 搜尋設定 + demo 對話）
cd apps/api && pnpm seed && cd ../..
```

### Step 5：啟動全部服務

```bash
# 終端 1：TypeScript 服務（NestJS + 3 React）
pnpm dev

# 終端 2：Python RAG Service
cd python/rag-service
uv run uvicorn rag_service.api.main:app --host 0.0.0.0 --port 3502 --reload
```

### Step 6：驗證

```bash
# NestJS API
curl http://localhost:3051/health

# RAG Service
curl http://localhost:3502/health

# 瀏覽器開啟
open http://localhost:5501  # Admin
open http://localhost:5502  # Chatbot
open http://localhost:5503  # Cleaner
```

---

## 四、預設帳號

| 帳號 | 密碼 | 角色 | 可用應用 |
|------|------|------|---------|
| admin@oda-cyber.com | OdaPoc2026! | admin | Admin + Chatbot + Cleaner |
| cleaner@oda-cyber.com | OdaPoc2026! | data_cleaner | Cleaner |
| reviewer@oda-cyber.com | OdaPoc2026! | data_reviewer | Cleaner |
| consultant@oda-cyber.com | OdaPoc2026! | consultant | Chatbot |
| user@oda-cyber.com | OdaPoc2026! | user | Chatbot |
| ituser@oda-cyber.com | OdaPoc2026! | it_user | Chatbot |
| basic@oda-cyber.com | OdaPoc2026! | basic_user | Chatbot（僅新手模式） |

> 首次登入系統會要求變更密碼（資通安全「普」級密碼政策）

---

## 五、測試指令

### TypeScript 測試

```bash
# 全部測試（透過 Turbo 平行執行）
pnpm test

# NestJS API 單元測試
pnpm --filter @oda-cyber/api test

# NestJS API 測試覆蓋率
pnpm --filter @oda-cyber/api test:cov

# 前端測試
pnpm --filter @oda-cyber/admin test
pnpm --filter @oda-cyber/chatbot test
pnpm --filter @oda-cyber/cleaner test

# Watch 模式（開發中即時回饋）
pnpm --filter @oda-cyber/api test:watch
```

### Python 測試

```bash
cd python

# 全部 Python 測試
uv run pytest

# RAG Service 測試
uv run pytest rag-service/tests/ -v

# Data Pipeline 測試
uv run pytest data-pipeline/tests/ -v

# 指定測試檔案
uv run pytest rag-service/tests/test_retriever.py -v
```

### 測試覆蓋率總覽

| 層級 | 測試規模 | 框架 |
|------|----------|------|
| NestJS API | 43 suites, 445 tests | Jest + ts-jest |
| Python rag-service | 21 files + 8 middleware tests | pytest + pytest-asyncio |
| Python data-pipeline | 13 files | pytest |
| React Admin | 4 files, 25 tests | Vitest + RTL + jsdom |
| React Cleaner | 5 files, 54 tests | Vitest + RTL + jsdom |
| React Chatbot | 4 files, 43 tests | Vitest + RTL + jsdom |

---

## 六、程式碼品質工具

### Linting

```bash
# ESLint 檢查
pnpm lint

# ESLint 自動修正
pnpm lint:fix

# 安全性專用掃描（eslint-plugin-security）
pnpm security:lint

# 依賴漏洞掃描
pnpm audit:check           # Node.js
cd python && uv pip audit   # Python
```

### Formatting

```bash
pnpm format    # Prettier 格式化（120 字元寬度、單引號、分號、LF）
```

### Git Hooks（自動化）

| Hook | 觸發時機 | 動作 |
|------|---------|------|
| pre-commit | `git commit` | lint-staged：ESLint --fix + Prettier |

已透過 Husky 自動安裝，`pnpm install` 後即啟用。

---

## 七、資料庫操作

### Prisma（NestJS 端）

```bash
cd apps/api

# 建立新 migration
dotenv -e ../../.env -- npx prisma migrate dev --name <description>

# 套用 migration（生產環境）
dotenv -e ../../.env -- npx prisma migrate deploy

# 重設資料庫（開發用，會清空所有資料）
dotenv -e ../../.env -- npx prisma migrate reset

# 開啟 Prisma Studio（GUI 資料瀏覽器）
pnpm prisma:studio

# 重新產生 Prisma Client
pnpm prisma:generate

# 執行 Seed
pnpm seed
```

### Alembic（Python 端）

```bash
cd python/rag-service

# 建立新 migration
uv run alembic revision --autogenerate -m "<description>"

# 套用所有 migration
uv run alembic upgrade head

# 回退一個版本
uv run alembic downgrade -1

# 查看目前版本
uv run alembic current
```

### 雙 ORM 注意事項

- NestJS 端（Prisma）管理：users, conversations, messages, prompt_templates, audit_logs, web_search_configs, password_histories
- Python 端（SQLAlchemy）管理：files, tasks, task_files, cleaning_audit_logs, gdrive_configs, gdrive_sync_files, gdrive_sync_history
- 兩端共用同一 PostgreSQL 實例，migration 各自獨立

---

## 八、RAG 知識庫管理

### 匯入文件

```bash
# 方式 1：目錄批次匯入（需 RAG Service 運行 + X-Internal-Token）
TOKEN=$(grep RAG_INTERNAL_API_KEY .env | cut -d= -f2)
curl -X POST http://localhost:3502/api/v1/ingest/directory \
  -H "Content-Type: application/json" \
  -H "X-Internal-Token: $TOKEN" \
  -d '{"directory": "/path/to/docs", "tags": ["tag1"], "hierarchical": true}'

# 方式 2：種子知識庫匯入
cd data/seed-knowledge && bash ingest.sh

# 方式 3：透過 Admin Dashboard UI 上傳→清洗→審核→送入 RAG
```

### 查詢知識庫

```bash
TOKEN=$(grep RAG_INTERNAL_API_KEY .env | cut -d= -f2)

# 統計
curl -s http://localhost:3502/api/v1/rag/stats \
  -H "X-Internal-Token: $TOKEN"

# 檢索測試
curl -X POST http://localhost:3502/api/v1/rag/retrieve \
  -H "Content-Type: application/json" \
  -H "X-Internal-Token: $TOKEN" \
  -d '{"question": "什麼是釣魚信件", "top_k": 5, "hybrid": true, "hierarchical": true}'
```

---

## 九、常見問題排除

| 問題 | 原因 | 解決方式 |
|------|------|---------|
| `pnpm dev` 報錯 | Node 版本不對 | `nvm use` |
| Prisma migrate 失敗 | 本機 PostgreSQL 未啟動 | `brew services start postgresql@17`，或啟動既有 PG 容器 |
| RAG Service 401 | 缺少 X-Internal-Token | 確認 .env 的 RAG_INTERNAL_API_KEY |
| Embedding 429 | Gemini 免費額度用完 | `.env` 改 `RAG_EMBEDDING_PROVIDER=openai` |
| Chatbot 無回應 | RAG Service 未啟動 | 第二終端啟動 FastAPI |
| 前端 API 404 | NestJS 未啟動 | 確認 `pnpm dev` 含 @oda-cyber/api |
| 密碼過期強制變更 | 90 天政策 | 依系統提示變更密碼 |
| Qdrant 連線失敗 | Docker 未啟動 | `docker start oda-qdrant` |

---

## 十、專案架構速覽

```
oda-cyber-konsult/
├── apps/
│   ├── api/           # NestJS 11 + Prisma（Port 3051）
│   ├── admin/         # React 19 + Ant Design v6（Port 5501）
│   ├── chatbot/       # React 19 + TailwindCSS v4（Port 5502）
│   └── cleaner/       # React 19 + Ant Design + TanStack Query（Port 5503）
├── packages/
│   ├── shared-types/  # 跨應用共用 TypeScript 型別
│   └── tsconfig/      # 共用 TS 設定
├── python/
│   ├── data-pipeline/ # PII 偵測 + 去識別化（Presidio + spaCy）
│   └── rag-service/   # FastAPI + Qdrant + BM25（Port 3502）
├── docker/            # Docker Compose（SearXNG + 生產環境）
├── data/
│   └── seed-knowledge/ # RAG 種子知識庫（11 份 MD + ingest.sh）
├── docs/
│   ├── 01-specs/      # PRD, SRS, ARCH, RTM, threat-model
│   ├── 02-testing/    # 測試策略, 測試指南
│   └── 03-operations/ # 運維手冊, 部署指南, 本文件
├── .env.example       # 環境變數範本（90+ 變數）
├── turbo.json         # Turborepo 任務編排
└── pnpm-workspace.yaml
```
