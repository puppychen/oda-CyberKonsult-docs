# 開發環境設定

## 系統需求

| 工具 | 版本 | 用途 |
|------|------|------|
| Node.js | 22+ | TypeScript 服務（NestJS / React） |
| pnpm | 9.15+ | TypeScript 套件管理 |
| Python | 3.12+ | RAG Service / Data Pipeline |
| uv | latest | Python 套件管理 |
| Docker | 24+ | PostgreSQL / Qdrant |
| Git | 2.40+ | 版本控制 |

## 安裝步驟

### 1. Node.js

```bash
# 使用 nvm 安裝（推薦）
nvm install    # 自動讀取 .nvmrc → Node 22
nvm use

# 啟用 pnpm
corepack enable
corepack prepare pnpm@9.15.4 --activate
```

### 2. Python

```bash
# 安裝 uv
curl -LsSf https://astral.sh/uv/install.sh | sh

# 確認版本
python3 --version  # >= 3.12
uv --version
```

### 3. Docker 服務

```bash
# PostgreSQL 17
docker run -d --name oda-postgres \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_PASSWORD=postgres \
  -e POSTGRES_DB=oda_cyber \
  -p 5432:5432 \
  postgres:17

# Qdrant 向量資料庫
docker run -d --name oda-qdrant \
  -p 6333:6333 \
  -p 6334:6334 \
  -v qdrant_data:/qdrant/storage \
  qdrant/qdrant:latest

# SearXNG 搜尋引擎（網路搜尋補充用）
docker run -d --name oda-searxng \
  -p 8080:8080 \
  searxng/searxng:latest
```

### 4. 專案初始化

```bash
# 複製環境變數
cp .env.example .env
# 編輯 .env，至少填入：
#   DATABASE_URL, JWT_SECRET, RAG_DATABASE_URL
#   RAG_GOOGLE_API_KEY 或 RAG_OPENAI_API_KEY

# TypeScript 相依
pnpm install

# Python 相依（同時安裝 rag-service 與 data-pipeline）
cd python && uv sync && cd ..

# Prisma 資料庫遷移（NestJS 端）
cd apps/api && npx prisma migrate dev && cd ../..

# Alembic 資料庫遷移（Python 端）
cd python/rag-service && uv run alembic upgrade head && cd ../..

# Prisma seed（可選，建立預設管理員帳號）
cd apps/api && npx prisma db seed && cd ../..
```

## 環境變數說明

完整變數見 `.env.example`，以下為必要項目：

| 變數 | 說明 | 範例 |
|------|------|------|
| `DATABASE_URL` | PostgreSQL 連線（Prisma） | `postgresql://postgres:postgres@localhost:5432/oda_cyber` |
| `JWT_SECRET` | JWT 簽章金鑰 | 隨機字串（至少 32 字元） |
| `RAG_DATABASE_URL` | PostgreSQL 連線（SQLAlchemy） | `postgresql+asyncpg://postgres:postgres@localhost:5432/oda_cyber` |
| `RAG_QDRANT_HOST` | Qdrant 位址 | `localhost` |
| `RAG_GOOGLE_API_KEY` | Gemini API 金鑰 | `AIza...` |
| `FASTAPI_BASE_URL` | RAG Service URL | `http://localhost:3502` |

## 啟動開發伺服器

### 全部一起啟動

```bash
# 終端機 1：TypeScript 服務（NestJS + Admin + Cleaner + Chatbot）
pnpm dev

# 終端機 2：FastAPI 服務（RAG + 清洗 + 檔案管理）
cd python/rag-service && uv run uvicorn rag_service.api.main:app --host 0.0.0.0 --port 3502 --reload
```

> **注意**：`pnpm dev` 僅啟動 Node.js 服務。FastAPI 必須在獨立終端機手動啟動，否則所有清洗相關功能（`/api/v1/*`）和 RAG 檢索會回傳 500。

### 個別啟動

```bash
# NestJS API only
pnpm --filter @oda-cyber/api dev

# Admin Dashboard only
pnpm --filter @oda-cyber/admin dev

# Cleaner App only
pnpm --filter @oda-cyber/cleaner dev

# Chatbot UI only
pnpm --filter @oda-cyber/chatbot dev
```

### 服務端點

| 服務 | URL | 說明 |
|------|-----|------|
| NestJS API | http://localhost:3051 | 後端主 API |
| Swagger UI | http://localhost:3051/api/docs | API 互動文件 |
| Admin Dashboard | http://localhost:5501 | 管理後台 |
| Chatbot UI | http://localhost:5502 | 使用者聊天介面 |
| Cleaner App | http://localhost:5503 | 資料清洗審核介面 |
| FastAPI (RAG) | http://localhost:3502 | Python RAG + 清洗 API |
| RAG Health | http://localhost:3502/health | 健康檢查 |

### 服務相依關係

`pnpm dev` 和 FastAPI 各自獨立啟動，但功能有上下游相依：

| 如果沒有啟動... | 受影響的功能 |
|----------------|-------------|
| **PostgreSQL** | 全部無法運作（NestJS + FastAPI 都依賴 DB） |
| **FastAPI (:3502)** | Cleaner 全頁 500、檔案上傳/清洗/審核/下載、Chatbot RAG 檢索（降級但不一定 500） |
| **Qdrant** | RAG 向量檢索失敗，Chatbot 回答品質下降 |
| **SearXNG** | 網路搜尋補充失效，Chatbot 仍可運作但少了即時資料 |
| **NestJS (:3051)** | 所有前端 API 請求失敗（三個前端都透過 proxy 打 NestJS） |

## 專案結構

```
oda-cyber-konsult/
├── apps/
│   ├── api/          # NestJS 後端 (Port 3051)
│   ├── admin/        # React 管理後台 (Port 5501)
│   └── chatbot/      # React 聊天介面 (Port 5502)
├── packages/
│   ├── shared-types/ # 共用 TypeScript 型別
│   └── tsconfig/     # 共用 TSConfig
├── python/
│   ├── rag-service/  # FastAPI RAG 服務 (Port 3502)
│   └── data-pipeline/ # 文件解析與清洗管線
├── docs/             # 專案文件
├── scripts/          # 工具腳本
└── .env.example      # 環境變數範本
```

## 常用指令

### TypeScript

```bash
pnpm build              # 編譯所有 TypeScript 專案
pnpm lint               # ESLint 檢查
pnpm format             # Prettier 格式化
pnpm test               # 執行所有測試
pnpm --filter @oda-cyber/api test   # 僅跑 API 測試
```

### Python

```bash
cd python
uv run pytest                          # 全部 Python 測試
uv run pytest rag-service/tests/       # RAG Service 測試
uv run pytest data-pipeline/tests/     # Data Pipeline 測試
uv run oda-rag status                  # RAG 系統狀態
uv run oda-rag ingest <file>           # 匯入文件到知識庫
uv run oda-rag query "問題"            # 測試 RAG 查詢
```

### 資料庫

```bash
# Prisma（NestJS 端）
cd apps/api
npx prisma migrate dev    # 建立/套用遷移
npx prisma studio         # 開啟 DB GUI
npx prisma db seed        # 執行 seed

# Alembic（Python 端）
cd python/rag-service
uv run alembic upgrade head     # 套用遷移
uv run alembic revision -m "描述"  # 建立新遷移
```

## 雙 ORM 架構

本專案使用兩套 ORM 共用同一 PostgreSQL 實例：

| ORM | 管理範圍 | 遷移工具 |
|-----|----------|----------|
| **Prisma** | 使用者/角色/對話/提示詞/稽核日誌 | `prisma migrate` |
| **SQLAlchemy** | 清洗任務/規則/上傳檔案/GDrive 同步 | `alembic` |

兩者的遷移獨立運作，不會互相干擾。Prisma 管理的表有 `_prisma_migrations`，Alembic 管理的表有 `alembic_version`。

## 整合測試

單元測試使用 mock，整合測試打真實 API + 查 DB 驗證跨服務行為。

```bash
# 前提：NestJS + FastAPI + PostgreSQL 已啟動，種子資料已就位

# Auth + RBAC 全鏈路（不需 FastAPI）
uv run tests/integration/test_auth.py --env dev

# Maker-Checker 審核流程（需 FastAPI）
uv run tests/integration/test_review_workflow.py --env dev

# Chat 對話全流程
uv run tests/integration/test_chat.py --env dev

# Admin 管理操作
uv run tests/integration/test_admin.py --env dev

# 檔案上傳 + 清洗（需 FastAPI）
uv run tests/integration/test_cleaning.py --env dev
```

整合測試設定檔：`tests/integration/config.json`

## 注意事項

- 前端開發模式透過 Vite proxy 將 API 請求轉發到 NestJS (port 3051)
- NestJS 再將清洗相關請求 (`/api/v1/*`) 透明轉發到 FastAPI (port 3502)
- **FastAPI 未啟動時清洗功能全部 500**，這是最常見的本機開發問題
- 修改 Prisma schema 後需執行 `npx prisma generate` 重新產生 client
- Python 端使用 `RAG_` 前綴的環境變數（pydantic-settings 自動讀取）
- Git hooks 由 Husky 管理，commit 前會自動執行 lint-staged
