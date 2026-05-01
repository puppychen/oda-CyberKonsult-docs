# Staging 環境建置與展示資料指南

> 最後更新：2026-04-24
> 適用版本：ODA Cyber Konsult POC
> 目的：在新機器上從零建置 STG 環境，含完整展示資料

---

## 一、前置需求

### 1.1 系統需求

| 項目 | 最低版本 | 安裝方式 |
|------|---------|---------|
| Node.js | 22+ | nvm（專案含 `.nvmrc`，`nvm use` 自動切版） |
| pnpm | 9.15+ | `corepack enable && corepack prepare pnpm@9.15.4 --activate` |
| Python | 3.12+ | pyenv 或系統安裝 |
| uv | 最新 | `curl -LsSf https://astral.sh/uv/install.sh \| sh` |
| Docker | 24+ | Docker Desktop 或 Docker Engine |
| 磁碟空間 | 20 GB+ | 含 Docker images、向量資料庫、上傳檔案 |

### 1.2 外部服務金鑰（必要）

| 金鑰 | 用途 | 取得方式 |
|------|------|---------|
| Google Gemini API Key | LLM 回答生成 + 向量嵌入 | [Google AI Studio](https://aistudio.google.com/) |
| OpenAI API Key（選用） | 備選 LLM/Embedding 提供者 | [OpenAI Platform](https://platform.openai.com/) |

> 至少需要 Gemini 或 OpenAI 其中一組金鑰，否則 Chatbot 無法產生回答。

---

## 二、建置步驟

### 2.1 取得程式碼

```bash
git clone <repo-url> oda-cyber-konsult
cd oda-cyber-konsult
```

### 2.2 一鍵自動建置（推薦）

```bash
./scripts/setup-dev-env.sh
```

此腳本自動執行 7 個步驟：

| 步驟 | 動作 | 說明 |
|------|------|------|
| 1 | 環境檢查 | 驗證 Node.js、Python、uv、Docker 版本 |
| 2 | TypeScript 依賴 | `pnpm install --frozen-lockfile` |
| 3 | Python 依賴 | `cd python && uv sync` |
| 4 | 環境變數 | 複製 `.env.example` → `.env`（若不存在） |
| 5 | Docker 基礎設施 | 建立 PostgreSQL 17 + Qdrant + SearXNG 容器 |
| 6 | 資料庫遷移 + 種子 | Prisma migrate + Alembic upgrade + Seed |
| 7 | 健康檢查 | 驗證所有服務就緒 |

### 2.3 手動建置（逐步執行）

若自動腳本環境與 STG 不完全相符，可按以下步驟手動操作：

#### Step 1：安裝依賴

```bash
# TypeScript
pnpm install --frozen-lockfile

# Python
cd python && uv sync && cd ..
```

#### Step 2：設定環境變數

```bash
cp .env.example .env
```

編輯 `.env`，**必須修改**的項目：

```bash
# === 資料庫連線 ===
DATABASE_URL="postgresql://postgres:<密碼>@<host>:5432/oda_cyber?schema=public"
RAG_DATABASE_URL="postgresql+asyncpg://postgres:<密碼>@<host>:5432/oda_cyber"

# === 安全金鑰（每個環境獨立產生）===
JWT_SECRET=$(openssl rand -hex 32)
INTERNAL_API_KEY=$(openssl rand -hex 32)
RAG_INTERNAL_API_KEY=<同 INTERNAL_API_KEY>

# === LLM API 金鑰 ===
RAG_GOOGLE_API_KEY=<你的 Gemini API Key>
# 或 RAG_OPENAI_API_KEY=<你的 OpenAI API Key>

# === SearXNG ===
SEARXNG_URL=http://localhost:8080
```

> **STG 安全原則**：`JWT_SECRET` 和 `INTERNAL_API_KEY` 必須與 Dev 環境不同，使用 `openssl rand -hex 32` 獨立產生。

#### Step 3：啟動 Docker 基礎設施

```bash
# PostgreSQL
docker run -d --name oda-postgres \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_PASSWORD=<密碼> \
  -e POSTGRES_DB=oda_cyber \
  -p 5432:5432 \
  -v pg_data:/var/lib/postgresql/data \
  --restart unless-stopped \
  postgres:17-alpine

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

驗證容器狀態：

```bash
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

#### Step 4：資料庫遷移

```bash
# Prisma（NestJS 端：users, conversations, messages 等）
cd apps/api
pnpm prisma:generate
npx prisma migrate deploy    # STG 用 deploy（非 dev）
cd ../..

# Alembic（Python 端：files, tasks, task_files 等）
cd python/rag-service
uv run alembic upgrade head
cd ../..
```

> `prisma migrate deploy` 只套用已存在的 migration，不會建立新的 migration 檔案——這是非開發環境的正確做法。

#### Step 5：載入種子資料

```bash
cd apps/api && pnpm seed && cd ../..
```

此步驟建立：
- **7 個使用者帳號**（見下方帳號列表）
- **12 個提示詞範本**（role × mode 組合）
- **1 筆網路搜尋設定**
- **3 筆 Demo 對話**（標題含 `[DEMO]` 前綴）

---

## 三、展示資料載入

### 3.1 RAG 知識庫種子資料

知識庫包含 11 份中文資安文件（152 chunks），覆蓋四大領域：

| 領域 | 文件數 | 內容 |
|------|--------|------|
| 資安意識 | 3 | 釣魚信件防護、實務案例、社交工程演練 |
| 個資保護 | 3 | 個資維護計畫、施行細則、ISO 27701 對照 |
| 法規標準 | 3 | 資安法修正重點、新舊條文對照、分級與 ISO 對應 |
| 顧問實務 | 2 | 顧問輔導注意事項、公務/非公務差異 |

**載入步驟：**

```bash
# 確認 RAG Service 已啟動
curl http://localhost:3502/health
# 預期回應：{"status":"ok","service":"CyberKonsult RAG Service"}

# 執行匯入腳本
cd data/seed-knowledge
bash ingest.sh
```

腳本會自動：
1. 健康檢查 RAG Service
2. 記錄匯入前的知識庫統計
3. 逐目錄匯入文件（自動標籤、分層切塊）
4. 匯入後統計 + 以「釣魚信件」驗證檢索

**驗證知識庫：**

```bash
# 檢查向量數量
curl -s http://localhost:3502/api/v1/rag/stats | python3 -m json.tool

# 預期結果：total_points ≈ 152, sources ≈ 11
```

### 3.2 Demo 對話重置

若需清除既有展示對話並重建預設 3 組：

- **方式一**：Admin Dashboard → 系統設定 → 點擊「一鍵重置 Demo 對話」
- **方式二**：重新執行 `cd apps/api && pnpm seed`

> Demo 對話標題以 `[DEMO]` 前綴區分，重置時不影響真實使用者的對話資料。

---

## 四、啟動服務

### 4.1 使用啟動腳本（推薦）

```bash
# 互動式選單
./scripts/dev-start.sh

# 或直接啟動全部
./scripts/dev-start.sh all
```

### 4.2 手動啟動

```bash
# 後端 API
pnpm --filter @oda-cyber/api dev &

# 前端（三個 React 應用）
pnpm --filter @oda-cyber/admin dev &
pnpm --filter @oda-cyber/chatbot dev &
pnpm --filter @oda-cyber/cleaner dev &

# RAG Service
cd python/rag-service
uv run uvicorn rag_service.api.main:app --host 0.0.0.0 --port 3502 --reload &
```

### 4.3 健康檢查

| 服務 | URL | 預期回應 |
|------|-----|---------|
| NestJS API | `http://localhost:3051/health` | `{"status":"ok"}` |
| RAG Service | `http://localhost:3502/health` | `{"status":"ok"}` |
| Admin Dashboard | `http://localhost:5501` | 登入頁面 |
| Chatbot UI | `http://localhost:5502` | 登入頁面 |
| Cleaner App | `http://localhost:5503` | 登入頁面 |
| PostgreSQL | `docker exec oda-postgres pg_isready -U postgres` | accepting connections |
| Qdrant | `http://localhost:6333/healthz` | `true` |

一鍵健康檢查：

```bash
./scripts/health-check.sh
```

---

## 五、預設帳號

| 角色 | Email | 密碼 | 系統存取範圍 |
|------|-------|------|------------|
| 管理員 | admin@oda-cyber.com | OdaPoc2026! | Admin Dashboard（全功能）+ Cleaner App + Chatbot |
| 資料清洗員 | cleaner@oda-cyber.com | OdaPoc2026! | Cleaner App（編輯、送審） |
| 資料審核員 | reviewer@oda-cyber.com | OdaPoc2026! | Cleaner App（批准、退回、送入 RAG） |
| 資安顧問 | consultant@oda-cyber.com | OdaPoc2026! | Chatbot（新手/一般/顧問三模式） |
| 一般使用者 | user@oda-cyber.com | OdaPoc2026! | Chatbot（新手/一般兩模式） |
| IT 工程師 | ituser@oda-cyber.com | OdaPoc2026! | Chatbot（IT 技術導向回應） |
| 基礎使用者 | basic@oda-cyber.com | OdaPoc2026! | Chatbot（基礎功能） |

---

## 六、展示操作驗證清單

完成建置後，依此清單驗證系統功能：

### 6.1 Admin Dashboard（admin 帳號）

- [ ] 登入 `http://localhost:5501`
- [ ] 首頁：知識庫 Chunks 總數 ≈ 152、標籤雲顯示 5 個標籤
- [ ] 使用者管理：7 個帳號可見，角色下拉可操作
- [ ] 提示詞管理：12 個範本，點擊「測試」可見變數注入彈窗
- [ ] 對話記錄：可見 Demo 對話（含 `[DEMO]` 前綴）
- [ ] 稽核日誌：操作紀錄即時出現，CSV/JSON 匯出可用
- [ ] 系統設定：Demo 模式開關、網路搜尋補充設定可調整

### 6.2 Cleaner App（cleaner + reviewer 帳號）

- [ ] 登入 `http://localhost:5503`（cleaner 帳號）
- [ ] Dashboard：資料管線漏斗、清洗效能、知識庫概況
- [ ] 任務列表：載入正常（無 HTTP 500）
- [ ] 檔案審核：去識別化內容可見，閱讀/編輯切換正常
- [ ] 知識庫：文件列表、切塊數、標籤顯示正確

### 6.3 Chatbot（user + consultant 帳號）

- [ ] 登入 `http://localhost:5502`（user 帳號）
- [ ] 模式切換：新手/一般兩個模式可切換
- [ ] 發送問題：SSE 串流回應正常產生
- [ ] 信心度指示器：顯示綠/黃/紅標籤
- [ ] 回饋按鈕：👍/👎 點擊後狀態改變
- [ ] 登出切換 consultant 帳號
- [ ] 模式切換：新手/一般/顧問三個模式可見
- [ ] 顧問模式提問「資安法修正」：產生四段結構化分析（條款分析、差異比較、ISO 對應、輔導建議）
- [ ] 匯出報告按鈕可見（顧問專屬）

---

## 七、常見問題排除

### Q1：Alembic migration 版本不一致

**症狀**：Cleaner App 任務列表回傳 HTTP 500，錯誤訊息含 `column ... does not exist`

**原因**：`alembic_version` 表的版本號被推進，但 DDL 未實際執行

**修復**：
```bash
# 1. 確認 DB 當前版本
cd python/rag-service
uv run alembic current

# 2. 檢查實際 schema（以 task_files 為例）
docker exec <postgres-container> psql -U postgres -d oda_cyber \
  -c "\d task_files"

# 3. 若版本號超前，回滾並重新執行
docker exec <postgres-container> psql -U postgres -d oda_cyber \
  -c "UPDATE alembic_version SET version_num = '<正確版本>';"
uv run alembic upgrade head
```

### Q2：RAG Service 無法連線

**症狀**：`/health` 回傳 `{"ragService":{"status":"down"}}`

**修復**：
```bash
# 確認 RAG Service 是否啟動
curl http://localhost:3502/health

# 若未啟動
cd python/rag-service
uv run uvicorn rag_service.api.main:app --host 0.0.0.0 --port 3502 --reload
```

### Q3：Chatbot 回應全為「網路補充」

**可能原因**：知識庫未載入種子資料

**修復**：
```bash
# 驗證知識庫是否有資料
curl -s http://localhost:3502/api/v1/rag/stats

# 若 total_points = 0，重新匯入
cd data/seed-knowledge && bash ingest.sh
```

### Q4：SearXNG 連線失敗

**症狀**：網路搜尋補充不觸發

**排查**：
```bash
# 檢查 SearXNG 容器
docker ps | grep searxng

# 確認 .env 中 SEARXNG_URL 與實際 port 一致
# 本機 Docker：SEARXNG_URL=http://localhost:8080
```

### Q5：密碼已過期

**症狀**：登入後被強制導向密碼變更頁面

**原因**：種子帳號建立超過 90 天（PASSWORD_MAX_AGE_DAYS）

**修復**：
```bash
# 重新執行 seed 更新密碼時間戳
cd apps/api && pnpm seed
```

---

## 八、環境變數速查

### 必須設定（STG 獨立值）

| 變數 | 說明 | 產生方式 |
|------|------|---------|
| `DATABASE_URL` | PostgreSQL 連線字串 | 依 STG DB 設定 |
| `RAG_DATABASE_URL` | Python 端 DB 連線（asyncpg） | 依 STG DB 設定 |
| `JWT_SECRET` | JWT 簽章金鑰 | `openssl rand -hex 32` |
| `INTERNAL_API_KEY` | NestJS ↔ FastAPI 內部認證 | `openssl rand -hex 32` |
| `RAG_INTERNAL_API_KEY` | 同 INTERNAL_API_KEY | 同上 |
| `RAG_GOOGLE_API_KEY` | Gemini API 金鑰 | Google AI Studio |

### 可使用預設值

| 變數 | 預設值 | 說明 |
|------|--------|------|
| `LLM_PROVIDER` | gemini | LLM 提供者 |
| `RAG_EMBEDDING_PROVIDER` | gemini | 向量嵌入提供者 |
| `RAG_EMBEDDING_DIMENSION` | 1536 | 向量維度 |
| `RAG_RETRIEVAL_TOP_K` | 5 | 檢索前 K 筆 |
| `RAG_CHUNK_SIZE` | 500 | 文件切塊大小 |
| `RAG_BM25_ENABLED` | true | 混合檢索（BM25 + 向量） |
| `RAG_BM25_TOKENIZER` | jieba | 中文分詞器 |
| `SEARXNG_URL` | http://localhost:8080 | SearXNG 位址 |
| `PASSWORD_MAX_AGE_DAYS` | 90 | 密碼過期天數 |

---

## 九、完整建置流程快速參照

```
┌────────────────────────────────────────────────────┐
│                 STG 建置總覽                        │
├────────────────────────────────────────────────────┤
│                                                    │
│  1. git clone → cd oda-cyber-konsult               │
│  2. cp .env.example .env → 編輯金鑰               │
│  3. pnpm install && cd python && uv sync           │
│  4. Docker: PostgreSQL + Qdrant + SearXNG          │
│  5. Prisma migrate deploy + Alembic upgrade head   │
│  6. pnpm seed（7 帳號 + 12 提示詞 + 3 Demo 對話） │
│  7. bash data/seed-knowledge/ingest.sh             │
│     （11 文件 → 152 chunks 知識庫）                │
│  8. ./scripts/dev-start.sh all                     │
│  9. 健康檢查 + 驗證清單                            │
│                                                    │
│  預計耗時：首次 15-20 分鐘（含 Docker 下載）       │
│  後續啟動：2-3 分鐘                                │
└────────────────────────────────────────────────────┘
```
