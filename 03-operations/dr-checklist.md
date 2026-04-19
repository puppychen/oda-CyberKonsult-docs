# 災難復原檢查表 (Disaster Recovery Checklist)

> **ODA Cyber Konsult**
> 文件類型：DR Runbook（POC 級）
> 建立日期：2026-04-13

---

## 0. 目標與範圍

| 項目 | 值 |
|------|----|
| **RTO**（復原時間目標） | 4 小時 |
| **RPO**（復原點目標） | 24 小時 |
| 適用環境 | 單機 Docker Compose / pm2 部署（POC） |
| 演練頻率 | 每季一次（1/4/7/10 月首週） |
| 文件版本 | 1.0.0 |

> 正式生產（Wave 6）將升級為 RTO 1h / RPO 1h，需配置主從複寫與持續備份。

---

## 1. 緊急聯絡清單

| 角色 | 聯絡方式 | 備援聯絡人 |
|------|---------|-----------|
| 值班 SRE | `<phone>` / `<email>` | `<backup-contact>` |
| 系統管理員 | `<admin>` | `<backup-admin>` |
| 資料庫 DBA | `<dba>` | — |
| 資安主管 | `<security-lead>` | — |
| 雲端供應商支援 | `<gcp/aws-support>` | — |

> **部署時請更新**：填寫實際聯絡資訊並每季檢視。

---

## 2. 5 大災難情境應變

### 情境 A：PostgreSQL 資料庫損毀 / 無法連線

**偵測指標**：
- `curl http://localhost:3051/health/ready` 回傳 `database: down`
- NestJS 日誌出現 `Prisma: Can't reach database server`

**應變步驟**（預估 30-60 分鐘）：

- [ ] **1. 確認故障範圍**
  ```bash
  docker ps | grep postgres               # 容器是否存在
  docker logs oda-postgres --tail 100      # 查看錯誤訊息
  df -h                                    # 磁碟空間
  ```
- [ ] **2. 嘗試重啟**
  ```bash
  docker restart oda-postgres
  sleep 30 && pg_isready -h localhost -p 5432
  ```
- [ ] **3. 若重啟無效 → 還原最新備份**
  ```bash
  # a. 停止依賴 DB 的服務
  docker stop oda-api oda-rag

  # b. 重建 volume（清除損毀資料）
  docker compose -f docker/docker-compose.prod.yml down postgres
  docker volume rm oda-cyber-prod_pg_data

  # c. 啟動新 PG 實例（等待健康）
  docker compose -f docker/docker-compose.prod.yml up -d postgres
  until docker exec oda-postgres pg_isready -U postgres; do sleep 2; done

  # d. 還原最新備份
  docker exec -i oda-postgres pg_restore \
    -U postgres -d oda_cyber --clean --if-exists \
    < /var/backups/oda-cyber/postgres/oda_cyber_latest.dump

  # e. 重啟相關服務
  docker compose -f docker/docker-compose.prod.yml up -d api rag-service
  ```
- [ ] **4. 驗證還原結果**
  ```bash
  curl http://localhost:3051/health/ready
  # 應回傳 {"status":"ready", "checks": {...}}
  ```
- [ ] **5. 記錄事件**：時間、故障原因、資料遺失量、還原耗時 → `logs/incident-YYYYMMDD.md`

---

### 情境 B：Qdrant 向量資料庫損毀

**偵測指標**：
- Chatbot 回答無引用來源、信心度皆顯示 🔴
- `curl http://localhost:6333/healthz` 失敗

**應變步驟**（預估 30-45 分鐘）：

- [ ] **1. 確認故障**：`docker logs oda-qdrant --tail 100`
- [ ] **2. 嘗試重啟**：`docker restart oda-qdrant && sleep 20`
- [ ] **3. 若無效 → 從 snapshot 還原**
  ```bash
  # a. 停止 Qdrant
  docker stop oda-qdrant

  # b. 取最近 snapshot
  ls -t /var/backups/oda-cyber/qdrant/*.snapshot | head -1
  SNAPSHOT=$(ls -t /var/backups/oda-cyber/qdrant/*.snapshot | head -1)

  # c. 清空現有 volume 並啟動
  docker volume rm oda-cyber-prod_qdrant_data
  docker compose -f docker/docker-compose.prod.yml up -d qdrant
  until curl -sf http://localhost:6333/healthz; do sleep 2; done

  # d. 將 snapshot 拷貝進容器並建立 collection
  docker cp "$SNAPSHOT" oda-qdrant:/qdrant/snapshots/cyberkonsult/
  SNAPNAME=$(basename "$SNAPSHOT")
  curl -X PUT "http://localhost:6333/collections/cyberkonsult/snapshots/recover" \
    -H "Content-Type: application/json" \
    -d "{\"location\": \"file:///qdrant/snapshots/cyberkonsult/${SNAPNAME}\"}"
  ```
- [ ] **4. 驗證向量數**
  ```bash
  curl -s http://localhost:6333/collections/cyberkonsult | grep -E 'status|vectors_count'
  ```
- [ ] **5. 若 snapshot 也遺失 → 重建知識庫**
  - 重新執行 `./scripts/dev-start.sh --seed-demo`
  - 或透過 Cleaner App 重新送入已清洗檔案

---

### 情境 C：上傳檔案遺失（uploads/）

**偵測指標**：
- 使用者回報下載清洗結果 404
- Cleaner App 檔案列表無法顯示內容

**應變步驟**（預估 15-30 分鐘）：

- [ ] **1. 確認範圍**
  ```bash
  ls -la ./uploads/
  du -sh /var/backups/oda-cyber/uploads/latest/
  ```
- [ ] **2. 從 rsync 備份恢復**
  ```bash
  rsync -an --delete /var/backups/oda-cyber/uploads/latest/ ./uploads/  # dry-run
  rsync -a --delete /var/backups/oda-cyber/uploads/latest/ ./uploads/   # actual
  ```
- [ ] **3. 權限校正**：`chown -R oda:oda ./uploads/`
- [ ] **4. 驗證**：隨機下載 1-2 個任務的清洗結果

---

### 情境 D：整機當機 / 硬體失效

**偵測指標**：所有服務無回應、SSH 連不上

**應變步驟**（預估 2-4 小時，視備機準備情況）：

- [ ] **1. 準備新主機**（雲端 VM / 備援實體機）
  - OS：Ubuntu 22.04 LTS 以上
  - 資源：4 CPU / 8GB RAM / 100GB SSD 最低
- [ ] **2. 安裝 runtime**
  ```bash
  # Docker + Docker Compose
  curl -fsSL https://get.docker.com | sh
  sudo systemctl enable --now docker
  # 或 VM 模式：Node 22 + pnpm + Python 3.12 + uv + Nginx
  ```
- [ ] **3. 取回源碼 + 備份**
  ```bash
  git clone <repo-url> /opt/oda-cyber-konsult
  scp backup-server:/var/backups/oda-cyber/** /var/backups/oda-cyber/
  ```
- [ ] **4. 還原 `.env.production`**（從密碼管理器）
- [ ] **5. 執行還原序列**：依序執行情境 A、B、C 的還原步驟
- [ ] **6. DNS 切換**：將域名指向新主機 IP
- [ ] **7. 端對端驗證**：執行 `scripts/health-check.sh` 全綠

---

### 情境 E：安全事件（JWT 外洩 / 入侵痕跡）

**偵測指標**：稽核日誌異常、未授權存取、敏感資料被下載

**應變步驟**（時間不限，以安全為優先）：

- [ ] **1. 立即隔離**
  ```bash
  # 切斷對外網路（Nginx 層）
  docker stop oda-nginx
  ```
- [ ] **2. 保存現場**
  ```bash
  # 保存當前日誌
  docker compose -f docker/docker-compose.prod.yml logs > incident-$(date +%Y%m%d).log

  # 建立即時備份（後續鑑識用）
  ./scripts/backup-postgres.sh /tmp/incident-backup/
  ```
- [ ] **3. 查閱稽核日誌**
  ```bash
  docker exec oda-postgres psql -U postgres -d oda_cyber \
    -c "SELECT timestamp, user_id, action, ip_address FROM audit_logs
        WHERE timestamp > NOW() - INTERVAL '48 hours'
        ORDER BY timestamp DESC LIMIT 100"
  ```
- [ ] **4. 輪換所有密鑰**（重要）
  - `JWT_SECRET`（重啟後所有 token 失效）
  - `INTERNAL_API_TOKEN`
  - 資料庫密碼
  - 所有管理員帳號強制改密
  ```bash
  openssl rand -base64 64  # for JWT_SECRET
  ```
- [ ] **5. 核對完整性**
  - 驗證 Prisma/Alembic migration 版本未被動手腳
  - 對照 `pg_dump` 與當前 DB schema 差異
  - 檢查 `uploads/` 是否有未知檔案
- [ ] **6. 依《個資法》通報**
  - 若確認 PII 外洩 → 72 小時內通報個資主管機關
  - 通知受影響使用者
- [ ] **7. 事後改進**
  - 更新 threat-model.md
  - 加強對應 mitigation（M-029 CORS/CSP、M-020 LLM 配額監控、2FA）
  - 更新測試用例避免同類事件

---

## 3. 通用還原前檢查

執行任何還原操作前，請先確認：

- [ ] **A. 磁碟空間**：`df -h /var/backups /var/lib/docker /opt/oda-cyber-konsult` 皆有 > 20% 餘裕
- [ ] **B. 備份完整性**：最新備份非零大小且 `pg_restore --list` 可解析
- [ ] **C. 聯絡利害關係人**：告知預計停機時間（ETA）
- [ ] **D. 關閉寫入流量**：停止 API 或設維護頁，避免還原過程資料不一致

---

## 4. 還原後驗證清單

還原任一服務後必須通過：

- [ ] `curl http://localhost:3051/health` → `healthy`
- [ ] `curl http://localhost:3051/health/ready` → `ready`（DB + RAG 均 up）
- [ ] `curl http://localhost:3502/health` → `ok`
- [ ] `curl http://localhost:6333/collections/cyberkonsult` 回傳 vector count > 0
- [ ] Admin 登入成功 + 使用者列表可載入
- [ ] Chatbot 送出一個查詢取得回應 + 含引用來源
- [ ] Cleaner App 任務列表可顯示
- [ ] 稽核日誌最新時間戳與事故時間符合
- [ ] 執行 `./scripts/health-check.sh`（若存在）全綠

---

## 5. 演練計畫

| 週期 | 演練項目 | 紀錄位置 |
|------|---------|---------|
| 每季 | 情境 A 或 B 擇一完整演練 | `logs/dr-drill-YYYYQN.md` |
| 每半年 | 情境 D 完整跨機還原 | `logs/dr-drill-fullover.md` |
| 每年 | 情境 E 安全事件紅隊演練 | `logs/security-drill.md` |

**演練紀錄應包含**：
- 實際耗時 vs RTO 目標
- 遇到的障礙與解決方法
- 改進建議（更新腳本/文件/監控）
- 驗證清單逐項打勾證據

---

## 6. 已知限制（POC 階段）

| 限制 | 影響 | 解決階段 |
|------|------|---------|
| 單機部署，無冗餘 | 硬體故障將造成服務中斷 | Wave 6（K8s） |
| 備份僅本地存放 | 機房災難可能連備份一起遺失 | Wave 6（GCS/S3） |
| 無自動 failover | 需人工介入切換備援 | Wave 6（HA Pair） |
| 無即時監控告警 | 故障發現靠使用者回報 | Wave 6（Prometheus） |
| JWT_SECRET 輪換需停機 | 線上使用者被強制登出 | Wave 7（優雅輪換） |
| RTO 4h / RPO 24h | 資料最多可能遺失 24h | Wave 6（RPO < 1h） |

---

## 7. 相關文件

| 文件 | 說明 |
|------|------|
| `docs/03-operations/backup-strategy.md` | 備份排程與策略 |
| `docs/03-operations/runbook.md` | 運維手冊（故障排除） |
| `docs/01-specs/threat-model.md` | 威脅模型（§6 事件應變） |
| `docs/01-specs/system-audit-2026-04-13.md` | 系統稽核報告（Phase 5 HA） |
| `scripts/backup-*.sh` | 備份腳本 |
| `.env.production.example` | 生產環境變數範本 |
