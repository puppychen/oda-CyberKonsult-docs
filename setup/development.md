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
| `FASTAPI_BASE_URL` | RAG Service URL | `http://localhost:8000` |

## 啟動開發伺服器

### 全部一起啟動

```bash
# 終端機 1：TypeScript 服務（NestJS + Admin + Chatbot）
pnpm dev

# 終端機 2：RAG Service
cd python && uv run uvicorn rag_service.api.main:app --reload --port 8000
```

### 個別啟動

```bash
# NestJS API only
pnpm --filter @oda-cyber/api dev

# Admin Dashboard only
pnpm --filter @oda-cyber/admin dev

# Chatbot UI only
pnpm --filter @oda-cyber/chatbot dev
```

### 服務端點

| 服務 | URL | 說明 |
|------|-----|------|
| NestJS API | http://localhost:4000 | 後端主 API |
| Swagger UI | http://localhost:4000/api/docs | API 互動文件 |
| Admin Dashboard | http://localhost:5173 | 管理後台 |
| Chatbot UI | http://localhost:5174 | 使用者聊天介面 |
| RAG Service | http://localhost:8000 | Python RAG API |
| RAG Health | http://localhost:8000/health | 健康檢查 |

## 專案結構

```
oda-cyber-konsult/
├── apps/
│   ├── api/          # NestJS 後端 (Port 4000)
│   ├── admin/        # React 管理後台 (Port 5173)
│   └── chatbot/      # React 聊天介面 (Port 5174)
├── packages/
│   ├── shared-types/ # 共用 TypeScript 型別
│   └── tsconfig/     # 共用 TSConfig
├── python/
│   ├── rag-service/  # FastAPI RAG 服務 (Port 8000)
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

## 注意事項

- 前端開發模式透過 Vite proxy 將 API 請求轉發到 NestJS (port 4000)
- NestJS 再將清洗相關請求 (`/api/v1/*`) 透明轉發到 FastAPI (port 8000)
- 修改 Prisma schema 後需執行 `npx prisma generate` 重新產生 client
- Python 端使用 `RAG_` 前綴的環境變數（pydantic-settings 自動讀取）
- Git hooks 由 Husky 管理，commit 前會自動執行 lint-staged
