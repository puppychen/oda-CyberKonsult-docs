# 快速開始

## 前置需求

### 必要工具
- **Node.js 22+** - 建議使用 nvm 管理（`nvm use` 會自動讀取 .nvmrc）
- **pnpm 9.15+** - `corepack enable && corepack prepare pnpm@9.15.4 --activate`
- **Python 3.12+**
- **uv** - `curl -LsSf https://astral.sh/uv/install.sh | sh`

### 基礎設施（Docker）
以下服務需在本機 Docker 中運行：
- PostgreSQL 17 (port 5432)
- Qdrant (REST port 6333, gRPC port 6334)

## 安裝步驟

### 1. TypeScript 相依安裝
```bash
pnpm install
```

### 2. Python 相依安裝
```bash
cd python
uv sync
```

### 3. 環境變數設定
```bash
cp .env.example .env
# 編輯 .env 填入資料庫連線等設定
```

### 4. 資料庫初始化
```bash
cd apps/api
npx prisma migrate dev
```

## 啟動開發伺服器

### TypeScript 服務
```bash
# 從根目錄啟動所有 TypeScript 服務
pnpm dev
```

### RAG 服務
```bash
cd python
uv run uvicorn rag_service.api.main:app --reload --port 8000
```

## 驗證

- NestJS API: http://localhost:4000
- Admin Dashboard: http://localhost:5173
- Chatbot UI: http://localhost:5174
- RAG Service: http://localhost:8000/health
