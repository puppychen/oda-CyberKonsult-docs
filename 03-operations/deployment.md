---
audience: both
purpose: runbook
status: approved
owner: ODA Cyber Konsult
---

# 部署與維運指南

## 部署架構

```
                    ┌─── Nginx / Caddy (反向代理) ───┐
                    │                                 │
               ┌────▼────┐                     ┌─────▼─────┐
               │ NestJS  │                     │ Static    │
               │ API     │──── proxy ────▶     │ Admin +   │
               │ :3051   │                     │ Chatbot   │
               └────┬────┘                     └───────────┘
                    │
               ┌────▼────┐
               │ FastAPI  │
               │ RAG Svc  │
               │ :3502    │
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
JWT_ACCESS_EXPIRES_IN=60m
JWT_REFRESH_EXPIRES_IN=7d
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
      - "4000:3051"

  rag-service:
    build:
      context: .
      dockerfile: python/rag-service/Dockerfile
    env_file: .env
    depends_on: [postgres, qdrant]
    ports:
      - "8000:3502"

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
pnpm exec dotenv -e .env -- pnpm --filter @oda-cyber/api exec prisma migrate deploy
cd python/rag-service && uv run alembic upgrade head

# 4. 啟動服務（使用 PM2 或 systemd）
# NestJS
pm2 start apps/api/dist/main.js --name oda-api

# RAG Service（使用 gunicorn + uvicorn workers）
cd python && uv run gunicorn rag_service.api.main:app \
  -k uvicorn.workers.UvicornWorker \
  -w 2 --bind 0.0.0.0:3502
```

## 資料庫遷移（正式環境）

資料庫 migration 應以單次 release job 執行，不要讓每個 API／RAG replica 在啟動時各自執行。先備份 PostgreSQL，再依序套用 Prisma 與 Alembic；應用程式部署後再驗證版本。

```bash
# Prisma — 從 monorepo 根目錄執行 deploy（不產生新遷移）
pnpm exec dotenv -e .env.production -- pnpm --filter @oda-cyber/api exec prisma migrate deploy

# Alembic — 套用所有待套遷移
cd python/rag-service && uv run alembic upgrade head
```

Docker Compose 環境使用映像內既有 CLI：

```bash
docker compose -f docker/docker-compose.prod.yml run --rm api ./node_modules/.bin/prisma migrate deploy
docker compose -f docker/docker-compose.prod.yml run --rm rag-service alembic upgrade head
```

### Prisma 0010：移除 SearXNG 本機預設值

`0010_remove_websearch_url_default` 只移除 `web_search_configs.searxng_url` 欄位的資料庫預設值，不刪除或改寫既有資料。部署前應在各環境明確設定 `SEARXNG_URL`；例如同一 Docker 網路使用 `http://searxng:8080`，直接映射到主機時才使用 `http://localhost:8080`。

```bash
# Staging
pnpm exec dotenv -e .env.staging -- pnpm --filter @oda-cyber/api exec prisma migrate deploy
ENV_FILE=.env.staging ./scripts/verify-migration-0010.sh

# Production
pnpm exec dotenv -e .env.production -- pnpm --filter @oda-cyber/api exec prisma migrate deploy
ENV_FILE=.env.production ./scripts/verify-migration-0010.sh
```

驗證腳本預期輸出 `PASS: migration 0010 removed the environment-specific SearXNG URL default`。若舊資料仍為 `http://localhost:8888`，新版 API 第一次讀取時會改成該環境的 `SEARXNG_URL`；其他管理者自訂 URL 不會被覆寫。

回退新版應用程式時可保留此 migration。只有必須回退到「建立設定時不會提供 `searxng_url`」的舊 API，才需先將欄位預設設為該環境可連線的 SearXNG URL；不得重新使用舊版 `localhost:8888`，除非該環境確實在該位址提供服務。

### Prisma 0009：獨立認證工作階段

`0009_add_auth_sessions` 新增 `auth_sessions`，供 Admin、Cleaner、Chatbot 與 API 用戶端各自保存 Refresh Token 雜湊、目前 `jti`、到期與撤銷時間。這是 expand-contract migration：既有 `users.refresh_token` 不刪除，舊 Refresh Token 會在第一次刷新時惰性轉換。

各環境使用對應環境檔執行；不要使用 `prisma migrate dev`：

```bash
# Staging
pnpm exec dotenv -e .env.staging -- pnpm --filter @oda-cyber/api exec prisma migrate deploy
ENV_FILE=.env.staging ./scripts/verify-migration-0009.sh

# Production
pnpm exec dotenv -e .env.production -- pnpm --filter @oda-cyber/api exec prisma migrate deploy
ENV_FILE=.env.production ./scripts/verify-migration-0009.sh
```

若部署平台由 Secret Manager 直接注入 `DATABASE_URL`，可不設定 `ENV_FILE`；在工作目錄沒有 `.env` 時，驗證腳本會直接使用目前環境變數。預期驗證訊息為 `PASS: migration 0009 auth_sessions schema verified`。

手動唯讀檢查：

```sql
SELECT migration_name, finished_at
FROM _prisma_migrations
WHERE migration_name = '0009_add_auth_sessions';

SELECT column_name, is_nullable
FROM information_schema.columns
WHERE table_schema = 'public' AND table_name = 'auth_sessions'
ORDER BY ordinal_position;
```

若新版應用程式需回退，必須保留 `auth_sessions` 表。不要在緊急回退時 drop table，以免刪除仍有效的新版 Session；移除舊欄位應另立後續 contract migration。

#### 認證協定 rollout fence

本版 Refresh API 要求 `X-ODA-Auth-Session-Protocol: 2`。部署順序必須是：先套用 Prisma 0009，再部署新版 API、切換 100% 流量並確認舊 API replica 已排空，最後才發布 Admin、Cleaner、Chatbot。不得讓新舊 API revision 分流，否則缺少協定檢查的舊 API 仍可能接受舊頁面的 Refresh。新版 API 上線後，仍開啟的舊頁面 Refresh 會得到 426 且不輪替 Token；重新整理載入新版前端即可恢復。

Protocol 2 前端發布後，API 只能 roll forward 修復，或回退到已回補相同 426 fence 的版本；禁止回退到未檢查 `X-ODA-Auth-Session-Protocol` 的 API。Admin、Cleaner、Chatbot 的回退版本也必須持續送出 protocol 2 header，否則會被目前 API 以 426 拒絕。建立 rollback candidate 時，必須先執行 Controller 的版本拒絕測試、三端 Refresh header 測試與 PostgreSQL 平行輪替整合測試，再允許切換流量。

`011` 會將既有 `files.regulation_type IS NULL` 回填為 `general`。套用後驗證：

```sql
SELECT version_num FROM alembic_version;
SELECT COUNT(*) FROM files WHERE regulation_type IS NULL;
SELECT regulation_type, COUNT(*) FROM files GROUP BY regulation_type ORDER BY regulation_type;
SELECT COUNT(*) FROM migration_011_regulation_type_backfill;
SELECT column_name, data_type, character_maximum_length, is_nullable
FROM information_schema.columns
WHERE table_schema = 'public' AND table_name = 'tasks' AND column_name = 'task_name';
```

預期 Alembic 版本為 `012`、未分類數量為 `0`，且 `tasks.task_name` 為 nullable `character varying(100)`。只回復任務名稱功能時，先停止新版建立清洗任務，再執行 `uv run alembic downgrade 011`；此動作會刪除所有已填寫的任務名稱，必須先匯出或確認可捨棄。若要再回復分類 migration，才執行 `uv run alembic downgrade 010`；`011` 降版只還原 ledger 內且仍為 `general` 的檔案，已改成其他類型的值不會被覆寫。

可在名稱以 `_migration_test` 結尾的拋棄式資料庫執行 upgrade／downgrade 行為驗證：

```bash
MIGRATION_TEST_DATABASE_URL='postgresql+asyncpg://.../oda_011_migration_test' \
  ./scripts/verify-migration-011.sh
```

主機未安裝 `psql` 時，可改用一次性 PostgreSQL client 容器：

```bash
PSQL_DOCKER_IMAGE=postgres:17 \
MIGRATION_TEST_DATABASE_URL='postgresql+asyncpg://.../oda_011_migration_test' \
  ./scripts/verify-migration-011.sh
```

`012` 的 nullable 欄位、舊任務保留、寫入與降版移除欄位可用下列指令獨立驗證：

```bash
PSQL_DOCKER_IMAGE=postgres:17 \
MIGRATION_TEST_DATABASE_URL='postgresql+asyncpg://.../oda_012_migration_test' \
  ./scripts/verify-migration-012.sh
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
        proxy_pass http://localhost:3051;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # Health check
    location /health {
        proxy_pass http://localhost:3051;
    }
}
```

## 健康檢查

| 端點 | 方法 | 檢查項目 |
|------|------|----------|
| `GET /health` | NestJS | API 存活 + DB 連線 |
| `GET http://rag:3502/health` | FastAPI | RAG 服務 + Qdrant 連線 |

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
