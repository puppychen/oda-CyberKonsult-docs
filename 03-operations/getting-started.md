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
- SearXNG (port 8080) — 選用，網路搜尋補充

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

需要兩個終端機分別啟動 Node.js 和 Python 服務：

```bash
# 終端機 1：TypeScript 服務（NestJS + Admin + Cleaner + Chatbot）
pnpm dev

# 終端機 2：FastAPI 服務（RAG + 清洗 + 檔案管理）
cd python/rag-service && uv run uvicorn rag_service.api.main:app --host 0.0.0.0 --port 3502 --reload
```

> **重要**：`pnpm dev` 不會啟動 FastAPI。沒有啟動 FastAPI 時，Cleaner 清洗介面和檔案管理功能會回傳 500 錯誤。

## 驗證

| 服務 | URL | 說明 |
|------|-----|------|
| NestJS API | http://localhost:3051 | 後端主 API |
| Admin Dashboard | http://localhost:5501 | 管理後台 |
| Chatbot UI | http://localhost:5502 | 聊天介面 |
| Cleaner App | http://localhost:5503 | 資料清洗審核介面 |
| FastAPI (RAG) | http://localhost:3502/health | 健康檢查 |
