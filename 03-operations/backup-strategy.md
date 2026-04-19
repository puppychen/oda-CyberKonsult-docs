# 備份策略 (Backup Strategy)

> **ODA Cyber Konsult**
> 文件類型：運維備份計畫
> 建立日期：2026-04-13

---

## 1. 備份目標（POC 級 RTO/RPO）

| 目標 | 值 | 說明 |
|------|----|----|
| **RTO**（Recovery Time Objective） | 4 小時 | 單機重建 + 資料還原完成時間 |
| **RPO**（Recovery Point Objective） | 24 小時 | 每日備份一次，可容許遺失 24 小時資料 |
| 備份成功率 | ≥ 99% | 連續 30 天以內不超過 1 次失敗 |
| 備份驗證 | 每週 | 從最新備份 dry-run 還原至測試環境 |

> 正式生產（Wave 6）目標：RTO 1h / RPO 1h，需改採 Streaming Replication + Continuous Archiving。

---

## 2. 備份範圍

| 資料來源 | 敏感度 | 備份腳本 | 儲存位置 | 保留期 |
|---------|-------|---------|---------|--------|
| PostgreSQL（`oda_cyber` DB） | 極高 | `scripts/backup-postgres.sh` | `/var/backups/oda-cyber/postgres/` | 30 天 |
| Qdrant（`cyberkonsult` collection） | 中 | `scripts/backup-qdrant.sh` | `/var/backups/oda-cyber/qdrant/` | 28 天 |
| 上傳檔案（`uploads/`） | 極高 | `scripts/backup-uploads.sh` | `/var/backups/oda-cyber/uploads/` | 90 天 |
| 設定檔（`.env.production`） | 極高 | 手動備份 | 密碼管理器 / 保險庫 | 永久 |
| Nginx TLS 憑證 | 高 | Let's Encrypt 自動 | `docker/nginx/certs/` | 由 certbot 管理 |

> **不備份**：`node_modules/`、`.venv/`、Docker 映像檔（可從 registry 或源碼重建）、稽核日誌（在 DB 內，已隨 PostgreSQL 備份）

---

## 3. 備份排程

詳細 crontab 見 `scripts/crontab.example`，摘要如下：

| 頻率 | 時間 | 動作 |
|------|------|------|
| 每日 | 03:00 | PostgreSQL 全量 `pg_dump`（custom format） |
| 每週日 | 04:00 | Qdrant Snapshot 下載 |
| 每日 | 05:00 | Uploads rsync 增量備份（hard-link 節省空間） |
| 每月 1 日 | 02:00 | Docker image prune |
| 每季首月 1 日 | 02:30 | DR 演練提醒信 |

**排程安排原則**：
- 03:00-06:00 為業務低峰時段，備份對線上服務影響最小
- Qdrant 改為每週以降低磁碟成本（向量資料變動頻率低）
- Uploads 每日以防資料遺失（使用者上傳不可重建）

---

## 4. 還原程序

### 4.1 PostgreSQL 還原

```bash
# 從最新備份還原（完整覆蓋）
pg_restore \
  --host=localhost --port=5432 --username=postgres \
  --dbname=oda_cyber \
  --clean --if-exists \
  /var/backups/oda-cyber/postgres/oda_cyber_latest.dump

# 從特定時間點還原
pg_restore --dbname=oda_cyber \
  /var/backups/oda-cyber/postgres/oda_cyber_20260413_030000.dump
```

**還原後驗證**：
```bash
psql -U postgres -d oda_cyber -c "SELECT COUNT(*) FROM users;"
psql -U postgres -d oda_cyber -c "SELECT MAX(created_at) FROM audit_logs;"
```

### 4.2 Qdrant 還原

```bash
# 1. 將 snapshot 檔案上傳至 Qdrant 主機的 snapshots 目錄
SNAPSHOT=cyberkonsult_20260413_040000.snapshot
docker cp "/var/backups/oda-cyber/qdrant/${SNAPSHOT}" \
  oda-qdrant:/qdrant/snapshots/cyberkonsult/${SNAPSHOT}

# 2. 呼叫 Qdrant Recover API
curl -X PUT "http://localhost:6333/collections/cyberkonsult/snapshots/recover" \
  -H "Content-Type: application/json" \
  -d "{\"location\": \"file:///qdrant/snapshots/cyberkonsult/${SNAPSHOT}\"}"

# 3. 驗證
curl -s http://localhost:6333/collections/cyberkonsult | python3 -m json.tool
```

### 4.3 Uploads 還原

```bash
# rsync 從 latest 恢復（dry-run 預覽）
rsync -an --delete /var/backups/oda-cyber/uploads/latest/ ./uploads/

# 確認無誤後實際執行
rsync -a --delete /var/backups/oda-cyber/uploads/latest/ ./uploads/
```

---

## 5. 備份驗證

### 5.1 每週自動驗證（排程）

建議新增測試腳本 `scripts/verify-backup.sh`（未提供，列為 TODO）：
1. 將最新 `oda_cyber_latest.dump` 還原至臨時 DB（`oda_cyber_verify`）
2. 執行 `SELECT COUNT(*) FROM users/conversations/messages` 確認表結構與筆數合理
3. 清理臨時 DB
4. 紀錄驗證結果至 `logs/backup-verify.log`

### 5.2 每季手動 DR 演練

見 `docs/03-operations/dr-checklist.md`。

---

## 6. 異地備份（建議但尚未實作）

POC 階段備份僅在本機；生產環境應額外推送至：

| 目標 | 建議 | 實作階段 |
|------|------|---------|
| GCS / S3 | 每日 cron 推送 | Wave 6 |
| 異地 VM | rsync over SSH | Wave 6 |
| 磁帶 / 冷儲存 | 每季一次 | Wave 7+ |

---

## 7. 備份失敗處理

1. **監控**：`logs/backup-*.log` 內含 ERROR 字串觸發告警（Wave 6 接 Alertmanager）
2. **重試**：cron 失敗後 1 小時自動重試一次
3. **升級**：連續 2 次失敗 → 寄信至 `MAILTO` + 人工介入
4. **根因分析**：檢查磁碟容量、DB 連線、Qdrant 健康狀態

**常見失敗原因**：
- 磁碟空間不足 → `df -h` 確認、擴容或提前清理
- PG 連線失敗 → 檢查 `PGPASSWORD`、`.pgpass` 或 SSL 憑證
- Qdrant API 逾時 → 大 collection 首次快照可能需 10+ 分鐘，增加 curl timeout

---

## 8. 相關文件

| 文件 | 說明 |
|------|------|
| `scripts/backup-postgres.sh` | PG 備份腳本 |
| `scripts/backup-qdrant.sh` | Qdrant 備份腳本 |
| `scripts/backup-uploads.sh` | Uploads 備份腳本 |
| `scripts/crontab.example` | Cron 排程範本 |
| `docs/03-operations/dr-checklist.md` | 災難復原檢查表 |
| `docs/03-operations/runbook.md` §5 | 運維手冊備份章節 |
