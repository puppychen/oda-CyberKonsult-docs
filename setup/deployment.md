# 部署與維運指南

## 部署架構

```
                    ┌─── Nginx / Caddy (反向代理) ───┐
                    │                                 │
               ┌────▼────┐                     ┌─────▼─────┐
               │ NestJS  │                     │ Static    │
               │ API     │──── proxy ────▶     │ Admin +   │
               │ :4000   │                     │ Chatbot   │
               └────┬────┘                     └───────────┘
                    │
               ┌────▼────┐
               │ FastAPI  │
               │ RAG Svc  │
               │ :8000    │
               └────┬────┘
                    │
          ┌────────┼────────┐
     ┌────▼────┐  ┌▼──────┐  ┌▼────────┐
     │PostgreSQL│  │Qdrant │  │File     │
     │ :5432   │  │:6333  │  │Storage  │
     └─────────┘  └───────┘  └─────────┘
```

## 編譯產出

### TypeScript 服務

```bash
# 編譯所有專案
pnpm build

# 產出位置
# apps/api/dist/         → NestJS 編譯結果
# apps/admin/dist/       → Admin 靜態檔案
# apps/chatbot/dist/     → Chatbot 靜態檔案
```

### Python 服務

Python 無需預編譯，直接以 uv 或 pip 安裝相依後啟動。

## 環境變數（正式環境）

在正式環境中，以下設定需調整：

```bash
# 安全性
JWT_SECRET=<至少 64 字元的隨機字串>
RAG_DEBUG=false

# 資料庫
DATABASE_URL=postgresql://<user>:<password>@<host>:5432/oda_cyber
RAG_DATABASE_URL=postgresql+asyncpg://<user>:<password>@<host>:5432/oda_cyber

# Qdrant
RAG_QDRANT_HOST=<qdrant-host>
RAG_QDRANT_PORT=6333

# API Keys
RAG_GOOGLE_API_KEY=<production-key>

# GDrive（如有使用）
RAG_GDRIVE_ENCRYPTION_KEY=<fernet-key>
```

## 部署方式

### 方式 A：Docker Compose（推薦）

建議使用 `docker-compose.yml` 統一管理所有服務：

```yaml
# docker-compose.yml (範例結構)
services:
  postgres:
    image: postgres:17
    environment:
      POSTGRES_DB: oda_cyber
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - pg_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  qdrant:
    image: qdrant/qdrant:latest
    volumes:
      - qdrant_data:/qdrant/storage
    ports:
      - "6333:6333"

  api:
    build:
      context: .
      dockerfile: apps/api/Dockerfile
    env_file: .env
    depends_on: [postgres]
    ports:
      - "4000:4000"

  rag-service:
    build:
      context: .
      dockerfile: python/rag-service/Dockerfile
    env_file: .env
    depends_on: [postgres, qdrant]
    ports:
      - "8000:8000"

  nginx:
    image: nginx:alpine
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./apps/admin/dist:/usr/share/nginx/admin:ro
      - ./apps/chatbot/dist:/usr/share/nginx/chatbot:ro
    depends_on: [api]
    ports:
      - "80:80"
      - "443:443"

volumes:
  pg_data:
  qdrant_data:
```

### 方式 B：GCP Cloud Run

各服務可獨立部署為 Cloud Run 服務：

| 服務 | Cloud Run | 說明 |
|------|-----------|------|
| NestJS API | `oda-api` | CPU always allocated（WebSocket 需要） |
| RAG Service | `oda-rag` | CPU always allocated |
| Admin/Chatbot | Cloud Storage + CDN | 靜態檔案直接託管 |

搭配：
- **Cloud SQL** (PostgreSQL 17)：替代本地 PostgreSQL
- **GCE 或 Cloud Run**：運行 Qdrant（需持久化 volume）

### 方式 C：VM 直接部署

```bash
# 1. 安裝相依
# Node.js 22, pnpm, Python 3.12, uv, Nginx

# 2. 編譯
pnpm install && pnpm build
cd python && uv sync

# 3. 資料庫遷移
cd apps/api && npx prisma migrate deploy
cd python/rag-service && uv run alembic upgrade head

# 4. 啟動服務（使用 PM2 或 systemd）
# NestJS
pm2 start apps/api/dist/main.js --name oda-api

# RAG Service（使用 gunicorn + uvicorn workers）
cd python && uv run gunicorn rag_service.api.main:app \
  -k uvicorn.workers.UvicornWorker \
  -w 2 --bind 0.0.0.0:8000
```

## 資料庫遷移（正式環境）

```bash
# Prisma — 使用 deploy（不產生新遷移）
cd apps/api && npx prisma migrate deploy

# Alembic — 套用所有待套遷移
cd python/rag-service && uv run alembic upgrade head
```

## Nginx 反向代理範例

```nginx
server {
    listen 80;
    server_name your-domain.com;

    # Admin Dashboard
    location / {
        root /usr/share/nginx/admin;
        try_files $uri $uri/ /index.html;
    }

    # Chatbot UI（子路徑或子網域）
    location /chatbot/ {
        alias /usr/share/nginx/chatbot/;
        try_files $uri $uri/ /chatbot/index.html;
    }

    # NestJS API
    location /api/ {
        proxy_pass http://localhost:4000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # Health check
    location /health {
        proxy_pass http://localhost:4000;
    }
}
```

## 健康檢查

| 端點 | 方法 | 檢查項目 |
|------|------|----------|
| `GET /health` | NestJS | API 存活 + DB 連線 |
| `GET http://rag:8000/health` | FastAPI | RAG 服務 + Qdrant 連線 |

建議在 Load Balancer 或容器編排中設定 health check interval = 30s。

## 備份策略

| 資料 | 備份方式 | 頻率 |
|------|----------|------|
| PostgreSQL | `pg_dump` / Cloud SQL 自動備份 | 每日 |
| Qdrant Snapshots | `POST /collections/{name}/snapshots` | 每週 |
| 上傳檔案 | rsync 到備份磁碟 / Cloud Storage | 每日 |
| .env 設定 | 加密保存於密碼管理器 | 變更時 |

## 日誌與監控

- NestJS：標準輸出（stdout），可由 PM2/Docker 收集
- FastAPI：標準輸出 + `RAG_DEBUG=true` 啟用詳細日誌
- 建議整合 Cloud Logging 或 ELK Stack 做集中式日誌管理
- 稽核日誌儲存於 PostgreSQL `audit_logs` 表

## 安全性檢查清單

- [ ] `JWT_SECRET` 使用強隨機字串（>= 64 字元）
- [ ] `RAG_DEBUG=false`
- [ ] PostgreSQL 不對外開放（僅內網存取）
- [ ] Qdrant 不對外開放（僅內網存取）
- [ ] HTTPS 憑證設定（Let's Encrypt 或 Cloud 管理）
- [ ] 環境變數不寫入版本控制
- [ ] 檔案上傳大小限制已設定（預設 50MB）
- [ ] Rate limiting 已啟用（預設 60 req/min）
