---
audience: human-primary
purpose: guide
status: approved
owner: ODA Cyber Konsult
---

# ODA CyberKonsult - 系統啟動與測試驗證指南

> 最後驗證日期：2026-02-24
> 最後更新日期：2026-07-18

## 1. 前置條件

| 項目 | 需求 |
|------|------|
| Node.js | 22+（透過 `.nvmrc` 管理） |
| pnpm | 9.15+ |
| Python | 3.12+ |
| uv | Python 套件管理 |
| PostgreSQL | `.env` 的 `DATABASE_URL`／`RAG_DATABASE_URL` 可連線；可使用本機服務或既有容器 |
| Docker | Qdrant + SearXNG 容器運行中 |

## 2. Docker 容器

確認以下容器正在運行：

```bash
docker ps --format '{{.Names}} {{.Ports}}' | grep -E 'postgres|qdrant'
```

| 容器名稱 | 用途 | Port Mapping |
|----------|------|-------------|
| boodion-database | PostgreSQL | 5234 → 5432 |
| oda-qdrant | 向量資料庫 | 6333-6334 → 6333-6334 |

> 目前本機 `.env` 使用 `localhost:5234/oda_cyber`，實際主機、Port、帳號與資料庫名稱一律以 `.env` 為準；腳本不得假設固定為 5432。

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
pnpm exec prisma generate

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

啟動項目：NestJS API (3051) + Admin Dashboard (5501) + Chatbot UI (5502) + Cleaner App (5503)

### 終端 2 — RAG Service

```bash
cd python/rag-service
uv run uvicorn rag_service.api.main:app --reload --host 127.0.0.1 --port 3502
```

> 若本機 3502 已被占用，可改用其他 Port；`FASTAPI_BASE_URL` 必須同步指向實際位置。

## 6. 服務清單與 Port 對應

| 服務 | Port | 健康檢查 |
|------|------|---------|
| NestJS API | 3051 | `curl http://localhost:3051/health` |
| Admin Dashboard | 5501 | 瀏覽器開啟 `http://localhost:5501` |
| Chatbot UI | 5502 | 瀏覽器開啟 `http://localhost:5502` |
| Cleaner App | 5503 | 瀏覽器開啟 `http://localhost:5503` |
| RAG Service | 3502 | `curl http://localhost:3502/health` |
| Qdrant | 6333 | `curl http://localhost:6333/healthz` |
| PostgreSQL | 由 `.env` 決定（目前 5234） | `bash scripts/health-check.sh` |

### 健康檢查預期回應

```bash
# NestJS — 應回傳 database: up, ragService: up
curl -s http://localhost:3051/health | uv run --project python/rag-service python -m json.tool

# RAG Service
curl -s http://localhost:3502/health
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
| Data Reviewer | reviewer@oda-cyber.com | OdaPoc2026! | data_reviewer | Chatbot + Cleaner |
| Demo User | user@oda-cyber.com | OdaPoc2026! | user | Chatbot |
| Basic User | basic@oda-cyber.com | OdaPoc2026! | basic_user | Chatbot |

> 帳號定義於 `apps/api/prisma/seed.ts`，密碼使用 bcrypt 12 rounds 雜湊。

## 8. 測試案例

### TC-1：Admin Dashboard (`http://localhost:5501`)

| # | 測試項目 | 預期結果 | 驗證狀態 |
|---|---------|---------|---------|
| 1.1 | 登入頁載入 | 顯示「ODA CyberKonsult / 管理後台登入」 | PASS |
| 1.2 | admin 帳號登入 | 右上角顯示「System Admin」，進入首頁 | PASS |
| 1.3 | 首頁功能 | 四步驟流程（上傳→選規則→清洗→完成）+ 檔案拖放上傳區 | PASS |
| 1.4 | 首頁清洗流程 | 自動套用固定規則（`get_default_rules()`），無需手動選擇規則 | PASS |
| 1.5 | 任務中心 `/tasks` | 顯示清洗任務列表，含狀態、進度條、分頁 | PASS |
| 1.6 | 使用者管理 `/users` | 顯示 5 個帳號，含角色下拉、狀態切換 | PASS |
| 1.7 | 提示詞管理 `/prompts` | 顯示 6 筆 seed 模板（3 角色 x 2-3 模式） | PASS |
| 1.8 | 資料來源 `/datasources` | Google Drive 連線設定表單 + 同步狀態 | PASS |
| 1.9 | 非 admin 登入被拒 | consultant 帳號顯示「此帳號無管理員權限」 | PASS |

### TC-2：Chatbot UI (`http://localhost:5502`)

| # | 測試項目 | 預期結果 | 驗證狀態 |
|---|---------|---------|---------|
| 2.1 | 登入頁載入 | 顯示「CyberKonsult 資安助手 / 登入以開始對話」 | PASS |
| 2.2 | user 帳號登入 | 右上角顯示「Demo User」，模式切換：新手/一般 | PASS |
| 2.3 | Demo 對話可見 | 左側顯示 [DEMO] 開頭的對話記錄 | PASS |
| 2.4 | 發送新訊息 | 需 RAG + LLM API Key 可用（未測試） | SKIP |
| 2.5 | consultant 登入 | 顯示「Demo Consultant」，模式：新手/一般/顧問，不同 Demo 對話 | PASS |
| 2.6 | 註冊功能 | 登入頁底部「還沒有帳號？註冊」連結可見 | PASS |

### TC-3：Cleaner App (`http://localhost:5503`)

| # | 測試項目 | 預期結果 | 驗證狀態 |
|---|---------|---------|---------|
| 3.1 | 登入頁載入 | 顯示「ODA 資料清洗管理系統 / 清洗管理介面登入」 | PASS |
| 3.2 | cleaner 帳號登入 | 右上角顯示「Data Cleaner」，進入 Dashboard | PASS |
| 3.3 | Dashboard 總覽 | 資料管線視覺化 + 清洗效能 + 知識庫概況 + 操作記錄 | PASS |
| 3.4 | 來源資料 `/files` | 檔案列表在檔名後顯示最新任務名稱，含類型與審核狀態篩選；舊任務未命名時顯示 ID 前 8 碼，無任務時顯示 `-`，狀態包含「未送審」 | PASS |
| 3.4a | 審核狀態分頁 | 選擇任一審核狀態後，`total` 為全部符合資料數；換頁不會混入其他狀態或重複資料 | PASS |
| 3.4b | 任務／逐檔狀態分層 | 任務尚未送審但逐檔標記未通過時，來源列表仍顯示「未送審」，不誤顯示「已退回」 | PASS |
| 3.5 | 任務列表 `/tasks` | 任務列表，含審批狀態 Tab（待審核/已批准/已退回/已送入） | PASS |
| 3.6 | 知識庫 `/knowledge-base` | 文件管理 + 語意搜尋 + 來源分析，向量總數 71 | PASS |
| 3.7 | user 登入被拒 | user 帳號顯示「此帳號無清洗管理權限」 | PASS |

#### 來源資料篩選 PostgreSQL 行為測試

```bash
cd python
RUN_POSTGRES_INTEGRATION=1 uv run --project rag-service \
  pytest rag-service/tests/test_source_file_filter_postgres_integration.py -q
```

測試會建立 25 筆唯一命名資料，驗證篩選發生在計數與分頁之前；另驗證完整 SQL 狀態矩陣、來源狀態採任務工作流而非逐檔審核結果，以及任務同時間時以 `id DESC` 決定最新任務，並確認 `latest_task_name`、`latest_task_id` 與審核狀態來自同一筆任務。所有 fixture 位於外層交易內，測試結束時 rollback；執行環境必須確認 `RAG_DATABASE_URL` 指向預期的開發資料庫。

#### 並行登入鎖定 PostgreSQL 行為測試

```bash
pnpm --filter @oda-cyber/api test:auth-lock-integration
```

前提為 NestJS API 與 PostgreSQL 已啟動，`.env` 的 `DATABASE_URL` 指向同一資料庫。測試建立一次性帳號並同時送出 5 次錯誤登入，驗證 HTTP 皆為 401、`login_attempts=5` 且 `locked_until` 位於未來；帳號會在結束時刪除。第 6 次請求可能先被登入限流回覆 429，因此 423 鎖定回應由 `auth.service.spec.ts` 獨立驗證。

#### RAG 入庫補償與法規版本測試

```bash
cd python
uv run pytest rag-service/tests/test_review_integration.py \
  rag-service/tests/test_regulation_versioning.py \
  rag-service/tests/test_regulations_integration.py \
  rag-service/tests/test_qdrant_indexed_at.py \
  rag-service/tests/test_retriever.py -q
```

測試涵蓋部分檔案失敗、資料庫 commit 成功／失敗的不明回應、Qdrant/BM25 完整來源補償、任務維持 `approved` 可重試、歷史回補時間軸、人工取代跨儲存同步，以及 vector／hierarchical／hybrid BM25 法規有效期傳遞與過濾。

### TC-3.5：Maker-Checker 審核流程（Cleaner App）

| # | 測試項目 | 預期結果 | 驗證狀態 |
|---|---------|---------|---------|
| 3.8 | cleaner 送審任務 | 以 cleaner 登入 → 開啟已完成清洗任務 → 按「送審」→ 任務狀態變更為 `review_requested` | PASS |
| 3.9 | reviewer 批准任務 | 以 reviewer 登入 → 開啟已送審任務 → 按「批准」→ 任務狀態變更為 `approved` | PASS |
| 3.10 | reviewer 退回任務 | 以 reviewer 登入 → 開啟已送審任務 → 按「退回任務」→ 填入理由 → 任務狀態變為 `rejected` 並解除內容凍結 | PASS |
| 3.11 | reviewer 將檔案標記為未通過 | 逐檔按「未通過」→ 檔案狀態變為 `rejected`，任務仍為 `review_requested` 且內容維持凍結 | PASS |
| 3.12 | 自審驗證（職責分離） | 以 cleaner 送審 → 同一 cleaner 嘗試批准 → API 回傳 403 Forbidden | PASS |
| 3.13 | 凍結驗證 | 送審後 → 以 cleaner 嘗試編輯檔案 → API 回傳 400（檔案已凍結） | PASS |
| 3.14 | reviewer 可登入 Cleaner App | 以 reviewer 登入 → 顯示「Data Reviewer」→ 可查看任務列表 | PASS |
| 3.15 | 品質檢查編輯權限 | cleaner 僅能於送審前／任務退回後修改；reviewer 僅能於審核中修改待審檔案；已批准或已入庫一律唯讀 | PASS |
| 3.16 | 混合逐檔審核結果 | 任務同時有「通過」與「未通過」檔案時可批准；送入 RAG 只處理通過檔案 | PASS |
| 3.17 | 全部檔案未通過 | 任務沒有任何通過檔案時，前端禁止批准且 API 回傳 400 | PASS |
| 3.18 | 任務狀態即時更新 | 送審、批准、退回、送入成功後，任務詳情與所有列表頁籤重新查詢；切到「已退回」以 `rejected` 篩選；重新進入列表或視窗聚焦時必定重查，背景重查期間保留舊列並顯示載入狀態 | PASS |

### TC-4：角色權限交叉驗證

| # | 測試項目 | 預期結果 | 驗證狀態 |
|---|---------|---------|---------|
| 4.1 | consultant → Admin Dashboard | 被拒：「此帳號無管理員權限」 | PASS |
| 4.2 | user → Cleaner App | 被拒：「此帳號無清洗管理權限」 | PASS |
| 4.3 | admin → 全部應用 | Admin、Chatbot、Cleaner 皆可登入 | PASS |

## 9. 角色權限對照表

| 應用 | admin | data_cleaner | data_reviewer | consultant | user |
|------|-------|-------------|---------------|------------|------|
| Admin Dashboard (5501) | O | X | X | X | X |
| Chatbot UI (5502) | O | O | O | O | O |
| Cleaner App (5503) | O | O | O | X | X |

- **Admin Dashboard**：僅 `admin` 角色
- **Cleaner App**：`admin` + `data_cleaner` + `data_reviewer` 角色（Maker-Checker 審核流程）
- **Chatbot UI**：所有角色皆可登入

### Maker-Checker 角色權限細分（Cleaner App）

| 操作 | admin | data_cleaner | data_reviewer |
|------|-------|-------------|---------------|
| 檢視任務/檔案 | O | O | O |
| 編輯檔案內容 | O | O | X |
| 修改品質檢查 | O（送審前／退回後／審核中待審檔案） | O（送審前／退回後） | O（審核中待審檔案） |
| 送審任務 | O | O | X |
| 逐檔標記通過／未通過 | O | X | O |
| 批准任務 | O | X | O |
| 退回任務 | O | X | O |
| 送入 RAG | O | X | O |

> **職責分離**：送審者（submitted_by）不得為同一任務的審批者（approved_by），由伺服器端強制驗證。

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
lsof -i :5501
lsof -i :3502

# 常見衝突：port 3502 被 Parallels NAT (prl_naptd) 佔用
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
curl http://localhost:3051/health

# 錯誤：curl http://localhost:3051/api/health → 404
```
