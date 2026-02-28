# 運維手冊 (Runbook)

> **ODA Cyber Konsult - 資安助手 RAG 系統**
>
> 文件版本：1.0.0
> 建立日期：2026-03-01
> 文件類型：運維手冊（Operations Runbook）

---

## 1. 服務拓撲與埠位

```
使用者 ──→ [Nginx/Caddy :80/443]
               │
               ├──→ NestJS API (:4000)
               │         │
               │         ├──→ RAG Service / FastAPI (:8000)
               │         │         │
               │         │         ├──→ Qdrant (:6333/6334)
               │         │         └──→ PostgreSQL (:5432)
               │         │
               │         ├──→ Gemini / OpenAI API (HTTPS)
               │         ├──→ SearXNG (:8080, Docker 內部)
               │         └──→ PostgreSQL (:5432)
               │
               ├──→ Admin Dashboard (靜態, :5173 dev)
               ├──→ Chatbot UI (靜態, :5174 dev)
               └──→ Cleaner App (靜態, :5175 dev)
```

### 服務清單

| 服務 | Port | 技術 | 管理方式 | 關鍵性 |
|------|------|------|---------|--------|
| NestJS API | 4000 | NestJS 11 | PM2 / Docker | 核心 |
| RAG Service | 8000 | FastAPI | Gunicorn + Uvicorn / Docker | 核心 |
| PostgreSQL | 5432 | PostgreSQL 17 | Docker / Cloud SQL | 核心 |
| Qdrant | 6333/6334 | Qdrant | Docker | 核心 |
| SearXNG | 8080 | SearXNG | Docker（僅內部） | 輔助 |
| Admin Dashboard | 5173 (dev) | React + Vite | Nginx 靜態 | 管理 |
| Chatbot UI | 5174 (dev) | React + Vite | Nginx 靜態 | 使用者面向 |
| Cleaner App | 5175 (dev) | React + Vite | Nginx 靜態 | 管理 |

---

## 2. 健康檢查端點

### 2.1 NestJS API

| 端點 | 方法 | 說明 | 預期回應 |
|------|------|------|---------|
| `GET /health` | HTTP | 綜合健康檢查（DB + RAG） | `{"status":"healthy"}` |
| `GET /health/live` | HTTP | 存活探針（Kubernetes） | `{"status":"ok"}` |
| `GET /health/ready` | HTTP | 就緒探針（DB + RAG 連線） | `{"status":"ready"}` |

```bash
# 快速檢查
curl -s http://localhost:4000/health | python3 -m json.tool

# 預期回應
# {
#   "status": "healthy",
#   "checks": {
#     "database": { "status": "up" },
#     "ragService": { "status": "up" }
#   }
# }
```

### 2.2 RAG Service (FastAPI)

```bash
curl -s http://localhost:8000/health
# {"status":"ok","service":"CyberKonsult RAG Service"}
```

### 2.3 Qdrant

```bash
curl -s http://localhost:6333/healthz
# healthz check passed
```

### 2.4 PostgreSQL

```bash
docker exec <container_name> psql -U postgres -c "SELECT 1"
# 或使用 pg_isready
pg_isready -h localhost -p 5432
```

### 2.5 Kubernetes 探針設定

```yaml
livenessProbe:
  httpGet:
    path: /health/live
    port: 4000
  initialDelaySeconds: 10
  periodSeconds: 30

readinessProbe:
  httpGet:
    path: /health/ready
    port: 4000
  initialDelaySeconds: 5
  periodSeconds: 10
```

---

## 3. 常見故障排除

### 3.1 NestJS API 無回應

**症狀**：`curl http://localhost:4000/health` 無回應或超時

**排查步驟**：
1. 檢查程序是否存在：`lsof -i :4000` 或 `pm2 status`
2. 檢查日誌：`pm2 logs oda-api --lines 50` 或 `docker logs <container>`
3. 檢查 DB 連線：確認 `DATABASE_URL` 環境變數正確
4. 檢查記憶體：`free -m`，NestJS 建議至少 512MB

**解決方案**：
```bash
# 重啟服務
pm2 restart oda-api
# 或 Docker
docker restart oda-api
```

### 3.2 RAG Service 連線失敗

**症狀**：`/health` 回傳 `ragService: { status: "down" }`

**排查步驟**：
1. 確認 RAG Service 運行中：`lsof -i :8000`
2. 確認 `FASTAPI_BASE_URL` 環境變數指向正確位址
3. 檢查 RAG Service 日誌：查看是否有 Qdrant 連線錯誤
4. 確認 Qdrant 運行中：`curl http://localhost:6333/healthz`

**解決方案**：
```bash
# 重啟 RAG Service
cd python/rag-service
uv run uvicorn rag_service.api.main:app --reload --host 127.0.0.1 --port 8000

# 重啟 Qdrant
docker restart qdrant
```

### 3.3 PostgreSQL 連線拒絕

**症狀**：`Prisma: Can't reach database server`

**排查步驟**：
1. 確認 Docker 容器運行：`docker ps | grep postgres`
2. 確認 port mapping：`docker port <container_name>`
3. 確認 `DATABASE_URL` 中的 host/port/password 正確
4. 測試連線：`psql -h localhost -p 5432 -U postgres -d oda_cyber`

**解決方案**：
```bash
# 啟動 PostgreSQL
docker start <container_name>

# 檢查 Prisma migration
cd apps/api && npx prisma migrate status
```

### 3.4 SSE 串流中斷

**症狀**：Chatbot 回覆中途停止，無 `done` 事件

**排查步驟**：
1. 檢查 NestJS 日誌是否有 LLM API 超時錯誤
2. 確認 LLM API Key 有效且配額充足
3. 檢查 Nginx/反向代理的超時設定（需 > 30 秒）

**解決方案**：
```nginx
# Nginx 設定加大 proxy 超時
proxy_read_timeout 120s;
proxy_send_timeout 120s;
```

### 3.5 清洗任務卡在 processing

**症狀**：任務進度不更新，一直在 processing 狀態

**排查步驟**：
1. 檢查 RAG Service 日誌：是否有 Presidio/spaCy 錯誤
2. 確認 WebSocket 連線正常
3. 檢查檔案是否可讀取（磁碟空間、檔案權限）

**解決方案**：
```bash
# 檢查磁碟空間
df -h

# 檢查上傳目錄權限
ls -la uploads/

# 手動取消卡住的任務
curl -X DELETE http://localhost:4000/api/v1/tasks/<task_id> \
  -H "Authorization: Bearer <token>"
```

### 3.6 Qdrant 向量搜尋無結果

**症狀**：RAG 查詢無引用來源，LLM 回答為通用內容

**排查步驟**：
1. 確認知識庫非空：`curl http://localhost:6333/collections`
2. 檢查 collection 的 vector count
3. 測試直接搜尋：透過 Cleaner App 知識庫 → 語意搜尋 Tab
4. 檢查 embedding model 是否可用

**解決方案**：
```bash
# 查看 collection 資訊
curl -s http://localhost:6333/collections/cyberkonsult | python3 -m json.tool

# 確認向量數量
# 如果 vectors_count = 0，需要重新 ingest 資料
```

---

## 4. 日誌查看

### 4.1 NestJS API

```bash
# PM2 環境
pm2 logs oda-api --lines 100
pm2 logs oda-api --err --lines 50  # 僅錯誤

# Docker 環境
docker logs oda-api --tail 100 -f

# 開發環境（stdout）
# 直接查看終端輸出
```

### 4.2 RAG Service (FastAPI)

```bash
# Docker 環境
docker logs oda-rag --tail 100 -f

# 開發環境
# RAG_DEBUG=true 啟用詳細日誌
RAG_DEBUG=true uv run uvicorn rag_service.api.main:app --reload
```

### 4.3 稽核日誌查詢

稽核日誌儲存於 PostgreSQL `audit_logs` 表：

```bash
# 透過 API 查詢（需 Admin JWT）
curl "http://localhost:4000/api/audit-logs?limit=20" \
  -H "Authorization: Bearer <admin_token>"

# 匯出 CSV
curl "http://localhost:4000/api/audit-logs/export?format=csv&start_date=2026-03-01" \
  -H "Authorization: Bearer <admin_token>" -o audit_logs.csv

# 直接 SQL 查詢
psql -h localhost -p 5432 -U postgres -d oda_cyber \
  -c "SELECT timestamp, action, user_id, details FROM audit_logs ORDER BY timestamp DESC LIMIT 20"
```

---

## 5. 備份與還原

### 5.1 PostgreSQL 備份

```bash
# 全量備份
pg_dump -h localhost -p 5432 -U postgres -d oda_cyber -F c -f backup_$(date +%Y%m%d).dump

# 還原
pg_restore -h localhost -p 5432 -U postgres -d oda_cyber -c backup_20260301.dump

# 僅備份特定表（如清洗資料）
pg_dump -h localhost -p 5432 -U postgres -d oda_cyber \
  -t files -t tasks -t task_files -t rule_profiles -t anonymization_rules \
  -F c -f cleaning_backup_$(date +%Y%m%d).dump
```

### 5.2 Qdrant 備份

```bash
# 建立 Snapshot
curl -X POST "http://localhost:6333/collections/cyberkonsult/snapshots"
# 回傳 snapshot 檔名

# 列出 Snapshots
curl "http://localhost:6333/collections/cyberkonsult/snapshots"

# 下載 Snapshot
curl "http://localhost:6333/collections/cyberkonsult/snapshots/<snapshot_name>" \
  -o qdrant_snapshot_$(date +%Y%m%d).snapshot

# 從 Snapshot 還原
curl -X PUT "http://localhost:6333/collections/cyberkonsult/snapshots/recover" \
  -H "Content-Type: application/json" \
  -d '{"location": "file:///qdrant/snapshots/<snapshot_name>"}'
```

### 5.3 備份排程建議

| 資料 | 備份方式 | 頻率 | 保留期限 |
|------|----------|------|---------|
| PostgreSQL | `pg_dump` / Cloud SQL 自動 | 每日 | 30 天 |
| Qdrant Snapshots | REST API | 每週 | 4 週 |
| 上傳檔案 | rsync / Cloud Storage | 每日 | 90 天 |
| .env 設定 | 加密保存於密碼管理器 | 變更時 | 永久 |

---

## 6. 安全事件應變

### 6.1 JWT Token 洩漏

**嚴重性**：高

**應變步驟**：
1. **立即**：更換 `JWT_SECRET` 環境變數（所有已發行 token 立即失效）
2. **重啟** NestJS API 服務以套用新 secret
3. **通知**受影響使用者重新登入
4. **查閱**稽核日誌，確認洩漏期間是否有異常操作
5. **記錄**事件至安全日誌

```bash
# 1. 產生新 JWT_SECRET
openssl rand -base64 64

# 2. 更新 .env 中的 JWT_SECRET

# 3. 重啟服務
pm2 restart oda-api

# 4. 查閱異常操作
curl "http://localhost:4000/api/audit-logs?start_date=<leak_time>" \
  -H "Authorization: Bearer <new_admin_token>"
```

### 6.2 資料外洩（PII 暴露）

**嚴重性**：極高

**應變步驟**：
1. **隔離**：立即停止對外服務，阻斷存取
2. **評估**：確認外洩範圍（哪些檔案、哪些 PII 類型）
3. **通知**：依《個人資料保護法》通知主管機關與當事人
4. **修復**：
   - 檢查去識別化規則是否有漏洞
   - 重新執行受影響檔案的去識別化
   - 更新知識庫（刪除含 PII 的向量）
5. **事後**：更新安全政策與測試案例

```bash
# 刪除含 PII 的知識庫向量（透過 Cleaner App 或 API）
curl -X DELETE "http://localhost:8000/api/v1/rag/documents/by-source/<source>" \
  -H "X-Internal-Token: <token>"
```

### 6.3 暴力破解攻擊

**嚴重性**：中

**偵測指標**：
- 稽核日誌中大量 `user.login` 失敗記錄
- 同一 IP 在短時間內多次嘗試

**應變步驟**：
1. **確認** ThrottlerModule (60 req/min) 是否生效
2. **檢查**帳號鎖定機制是否正常（5 次失敗 → 15 分鐘鎖定）
3. **封鎖**來源 IP（Nginx / 防火牆層）
4. **加強**：考慮引入 CAPTCHA 或 2FA

---

## 7. 定期維護清單

### 7.1 每日

- [ ] 檢查 `/health` 端點回應正常
- [ ] 查看錯誤日誌是否有異常
- [ ] 確認 PostgreSQL 自動備份完成

### 7.2 每週

- [ ] 建立 Qdrant Snapshot
- [ ] 查看磁碟使用量（上傳檔案目錄）
- [ ] 查看稽核日誌是否有異常操作
- [ ] 檢查 SSL 憑證到期日

### 7.3 每月

- [ ] 更新相依套件（安全性修補）
- [ ] 清理過期的暫存檔案
- [ ] 檢視 Rate Limiting 設定是否需調整
- [ ] 覆核使用者帳號（停用不活躍帳號）

### 7.4 每季

- [ ] 密碼政策覆核（90 天過期已由系統強制）
- [ ] 資料庫效能調校（VACUUM、REINDEX）
- [ ] 安全性滲透測試
- [ ] 災難復原演練

---

## 8. 環境變數參考

> 完整設定請參閱 [setup/deployment.md](../setup/deployment.md)

| 變數 | 說明 | 關鍵性 |
|------|------|--------|
| `JWT_SECRET` | JWT 簽署金鑰（≥64 字元） | 核心安全 |
| `DATABASE_URL` | PostgreSQL 連線字串 | 核心 |
| `RAG_DATABASE_URL` | Python 端 PostgreSQL 連線（asyncpg） | 核心 |
| `FASTAPI_BASE_URL` | RAG Service 位址 | 核心 |
| `RAG_QDRANT_HOST` | Qdrant 主機位址 | 核心 |
| `RAG_GOOGLE_API_KEY` | Gemini API Key | LLM |
| `RAG_DEBUG` | 詳細日誌開關（正式環境設 false） | 除錯 |
| `INTERNAL_API_TOKEN` | NestJS↔FastAPI 內部認證 Token | 安全 |

---

## 相關文件

| 文件 | 說明 | 路徑 |
|------|------|------|
| 部署指南 | 編譯、部署方式、Nginx 設定 | [setup/deployment.md](../setup/deployment.md) |
| 開發環境 | 開發環境完整說明 | [setup/development.md](../setup/development.md) |
| 測試指南 | 服務啟動與測試驗證 | [testing/test-guide.md](../testing/test-guide.md) |
| Health API | 健康檢查端點規格 | [api/health-api.md](../api/health-api.md) |
| 系統架構 | 服務拓撲與通訊 | [architecture/system-overview.md](../architecture/system-overview.md) |

---

> **文件結束**
>
> 本文件為 ODA Cyber Konsult 的運維手冊。
> 部署相關細節請參閱 `deployment.md`，架構細節請參閱 `system-overview.md`。
