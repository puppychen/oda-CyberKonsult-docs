# ODA CyberKonsult - 系統啟動與測試驗證指南

> 最後驗證日期：2026-02-24

## 1. 前置條件

| 項目 | 需求 |
|------|------|
| Node.js | 22+（透過 `.nvmrc` 管理） |
| pnpm | 9.15+ |
| Python | 3.12+ |
| uv | Python 套件管理 |
| Docker | PostgreSQL + Qdrant 容器運行中 |

## 2. Docker 容器

確認以下容器正在運行：

```bash
docker ps --format '{{.Names}} {{.Ports}}' | grep -E 'postgres|qdrant'
```

| 容器名稱 | 用途 | Port Mapping |
|----------|------|-------------|
| boodion-database | PostgreSQL | 5234 → 5432 |
| qdrant | 向量資料庫 | 6333-6334 → 6333-6334 |

> 注意：PostgreSQL 對外 port 為 **5234**（非預設 5432），需在 `.env` 中正確設定。

## 3. 安裝依賴

```bash
# TypeScript 依賴
cd /path/to/oda-cyber-konsult
pnpm install

# Python 依賴
cd python
uv sync
```

## 4. 資料庫遷移與種子資料

```bash
# Prisma 遷移 + 生成 Client
cd apps/api
pnpm prisma:migrate
npx prisma generate

# 種子資料（建立預設帳號、提示詞模板等）
pnpm seed

# Alembic 遷移（清洗子系統資料表）
cd ../../python/rag-service
uv run alembic upgrade head
```

## 5. 啟動服務

### 終端 1 — TypeScript 全部服務（Turbo）

```bash
cd /path/to/oda-cyber-konsult
pnpm dev
```

啟動項目：NestJS API (4000) + Admin Dashboard (5173) + Chatbot UI (5174) + Cleaner App (5175)

### 終端 2 — RAG Service

```bash
cd python/rag-service
uv run uvicorn rag_service.api.main:app --reload --host 127.0.0.1 --port 8017
```

> 注意：預設 port 8000 可能被 Parallels NAT 佔用，實際使用 **8017**。
> 確保 `.env` 中 `FASTAPI_BASE_URL=http://localhost:8017` 與啟動 port 一致。

## 6. 服務清單與 Port 對應

| 服務 | Port | 健康檢查 |
|------|------|---------|
| NestJS API | 4000 | `curl http://localhost:4000/health` |
| Admin Dashboard | 5173 | 瀏覽器開啟 `http://localhost:5173` |
| Chatbot UI | 5174 | 瀏覽器開啟 `http://localhost:5174` |
| Cleaner App | 5175 | 瀏覽器開啟 `http://localhost:5175` |
| RAG Service | 8017 | `curl http://localhost:8017/health` |
| Qdrant | 6333 | `curl http://localhost:6333/healthz` |
| PostgreSQL | 5234 | `docker exec boodion-database psql -U postgres -c "SELECT 1"` |

### 健康檢查預期回應

```bash
# NestJS — 應回傳 database: up, ragService: up
curl -s http://localhost:4000/health | python3 -m json.tool

# RAG Service
curl -s http://localhost:8017/health
# {"status":"ok","service":"CyberKonsult RAG Service"}

# Qdrant
curl -s http://localhost:6333/healthz
# healthz check passed
```

## 7. 測試帳號

| 姓名 | Email | 密碼 | 角色 | 可登入應用 |
|------|-------|------|------|-----------|
| System Admin | admin@oda-cyber.com | OdaPoc2026! | admin | Admin + Chatbot + Cleaner |
| Data Cleaner | cleaner@oda-cyber.com | OdaPoc2026! | data_cleaner | Chatbot + Cleaner |
| Demo Consultant | consultant@oda-cyber.com | OdaPoc2026! | consultant | Chatbot |
| Demo User | user@oda-cyber.com | OdaPoc2026! | user | Chatbot |

> 帳號定義於 `apps/api/prisma/seed.ts`，密碼使用 bcrypt 12 rounds 雜湊。

## 8. 測試案例

### TC-1：Admin Dashboard (`http://localhost:5173`)

| # | 測試項目 | 預期結果 | 驗證狀態 |
|---|---------|---------|---------|
| 1.1 | 登入頁載入 | 顯示「ODA CyberKonsult / 管理後台登入」 | PASS |
| 1.2 | admin 帳號登入 | 右上角顯示「System Admin」，進入首頁 | PASS |
| 1.3 | 首頁功能 | 四步驟流程（上傳→選規則→清洗→完成）+ 檔案拖放上傳區 | PASS |
| 1.4 | 規則管理 `/rules` | 顯示 3 個規則設定檔（醫療研究用/學術研究用/完全合規用） | PASS |
| 1.5 | 任務中心 `/tasks` | 顯示清洗任務列表，含狀態、進度條、分頁 | PASS |
| 1.6 | 使用者管理 `/users` | 顯示 5 個帳號，含角色下拉、狀態切換 | PASS |
| 1.7 | 提示詞管理 `/prompts` | 顯示 6 筆 seed 模板（3 角色 x 2-3 模式） | PASS |
| 1.8 | 資料來源 `/datasources` | Google Drive 連線設定表單 + 同步狀態 | PASS |
| 1.9 | 非 admin 登入被拒 | consultant 帳號顯示「此帳號無管理員權限」 | PASS |

### TC-2：Chatbot UI (`http://localhost:5174`)

| # | 測試項目 | 預期結果 | 驗證狀態 |
|---|---------|---------|---------|
| 2.1 | 登入頁載入 | 顯示「CyberKonsult 資安助手 / 登入以開始對話」 | PASS |
| 2.2 | user 帳號登入 | 右上角顯示「Demo User」，模式切換：新手/一般 | PASS |
| 2.3 | Demo 對話可見 | 左側顯示 [DEMO] 開頭的對話記錄 | PASS |
| 2.4 | 發送新訊息 | 需 RAG + LLM API Key 可用（未測試） | SKIP |
| 2.5 | consultant 登入 | 顯示「Demo Consultant」，模式：新手/一般/顧問，不同 Demo 對話 | PASS |
| 2.6 | 註冊功能 | 登入頁底部「還沒有帳號？註冊」連結可見 | PASS |

### TC-3：Cleaner App (`http://localhost:5175`)

| # | 測試項目 | 預期結果 | 驗證狀態 |
|---|---------|---------|---------|
| 3.1 | 登入頁載入 | 顯示「ODA 資料清洗管理系統 / 清洗管理介面登入」 | PASS |
| 3.2 | cleaner 帳號登入 | 右上角顯示「Data Cleaner」，進入 Dashboard | PASS |
| 3.3 | Dashboard 總覽 | 資料管線視覺化 + 清洗效能 + 知識庫概況 + 操作記錄 | PASS |
| 3.4 | 來源資料 `/files` | 檔案列表，含類型/管線狀態篩選 | PASS |
| 3.5 | 任務列表 `/tasks` | 任務列表，含審批狀態 Tab（待審核/已批准/已退回/已送入） | PASS |
| 3.6 | 知識庫 `/knowledge-base` | 文件管理 + 語意搜尋 + 來源分析，向量總數 71 | PASS |
| 3.7 | user 登入被拒 | user 帳號顯示「此帳號無清洗管理權限」 | PASS |

### TC-4：角色權限交叉驗證

| # | 測試項目 | 預期結果 | 驗證狀態 |
|---|---------|---------|---------|
| 4.1 | consultant → Admin Dashboard | 被拒：「此帳號無管理員權限」 | PASS |
| 4.2 | user → Cleaner App | 被拒：「此帳號無清洗管理權限」 | PASS |
| 4.3 | admin → 全部應用 | Admin、Chatbot、Cleaner 皆可登入 | PASS |

## 9. 角色權限對照表

| 應用 | admin | data_cleaner | consultant | user |
|------|-------|-------------|------------|------|
| Admin Dashboard (5173) | O | X | X | X |
| Chatbot UI (5174) | O | O | O | O |
| Cleaner App (5175) | O | O | X | X |

- **Admin Dashboard**：僅 `admin` 角色
- **Cleaner App**：`admin` + `data_cleaner` 角色
- **Chatbot UI**：所有角色皆可登入

## 10. Chatbot 模式差異

| 模式 | 可用角色 | topK | 溫度 | 最大 Token | Rerank |
|------|---------|------|------|-----------|--------|
| 新手 | 全部 | 3 | 0.3 | 1024 | - |
| 一般（標準） | 全部 | 5 | 0.1 | 2048 | - |
| 顧問（專家） | consultant, admin | 8 | 0.05 | 4096 | O |

## 11. 常見問題排除

### Port 衝突

```bash
# 查看 port 佔用
lsof -i :5173
lsof -i :8000

# 常見衝突：port 8000 被 Parallels NAT (prl_naptd) 佔用
# 解法：RAG Service 改用其他 port（如 8017），並更新 .env FASTAPI_BASE_URL
```

### Docker 容器未啟動

```bash
# 啟動 PostgreSQL
docker start boodion-database

# 啟動 Qdrant
docker start qdrant
```

### Prisma Client 未生成

```bash
cd apps/api && npx prisma generate
```

### 登出功能異常（Cleaner App）

Cleaner App 的登出按鈕可能偶爾未正確清除 token。手動清除方式：
- 開啟瀏覽器 DevTools → Application → Local Storage → 清除對應項目
- 或直接導航至 `/login`

### NestJS Health 端點

```bash
# 正確路徑（無 /api 前綴）
curl http://localhost:4000/health

# 錯誤：curl http://localhost:4000/api/health → 404
```
