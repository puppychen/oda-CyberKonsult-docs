---
audience: both
purpose: runbook
status: draft
owner: ODA Cyber Konsult
updated: 2026-08-05
---

# GCP 部署設置指南（Console 操作版）

## TL;DR

本文件帶您用 **Google Cloud Console 網頁介面**（不需打指令）把 CyberKonsult 部署到 GCP。

| 項目 | 結論 |
|------|------|
| 資料落地 | **全部留在您自己的 GCP 專案內**（Qdrant 自架、Cloud SQL 私有 IP） |
| 部署區域 | `asia-east1`（台灣彰化）——台灣使用者延遲最低 |
| 預估月費 | **約 US$100～110** |
| 需要改程式碼 | **有一項**：BM25／Embedding 快取改為支援 PostgreSQL（由開發方負責，見第 6 章） |
| 完成時間 | 基礎設施約 3～4 小時；含程式碼改動約 5～7 個工作天 |

> **為什麼要改程式碼**：Cloud Run 是「無狀態」服務，容器隨時會被回收重建。目前 BM25 關鍵字索引存在本地 SQLite 檔案，放到 Cloud Run 上會因多個執行個體同時寫入而損毀。這是唯一必須改的地方，其餘（檔案上傳、Qdrant、資料庫）都只需調整設定值。

---

## 部署架構

```
                        使用者瀏覽器
                     ┌───────┴────────┐
                 取得畫面          呼叫 API
                     │            （跨網域，需 CORS）
      ┌──────────────▼───────────────┐      │
      │  Firebase Hosting（含 CDN）   │      │
      │  ├ Chatbot 聊天介面           │      │
      │  ├ Admin 管理後台             │      │
      │  └ Cleaner 清洗審核           │      │
      └──────────────────────────────┘      │
                                             ▼
┌─ 您的 GCP 專案（asia-east1）────────────────────────────────┐
│                                                             │
│  ┌── Cloud Run ─────────────────────────────────────────┐   │
│  │  oda-api（NestJS）    對外開放，靠 JWT 保護           │   │
│  │  oda-rag（FastAPI）   僅限內部                        │   │
│  │  oda-searxng          僅限內部                        │   │
│  └────────┬──────────────────┬──────────────────────────┘   │
│           │ Direct VPC Egress（走內部網路）                  │
│           ▼                  ▼                              │
│  ┌────────────────┐   ┌──────────────────────┐              │
│  │  Cloud SQL     │   │  GCE 虛擬機（無外網）  │              │
│  │  PostgreSQL 17 │   │  └ Qdrant 向量資料庫  │              │
│  │  （私有 IP）    │   │        │              │              │
│  │  ├ 主業務資料表 │   │        ▼              │              │
│  │  ├ 清洗資料表   │   │  持久磁碟 30GB        │              │
│  │  └ BM25 索引 ★ │   │  （每日自動快照）      │              │
│  └────────────────┘   └──────────────────────┘              │
│                                                             │
│  Cloud Storage 儲存桶（掛載進 oda-rag）                       │
│  └ 上傳檔案 / 清洗產出 / 暫存                                 │
│                                                             │
│  Secret Manager（機密）  Artifact Registry（映像檔）          │
└─────────────────────────────────────────────────────────────┘
```

★ = 本次需要改程式碼的部分

---

## 事前準備

### 您需要具備

| 項目 | 說明 |
|------|------|
| Google 帳號 | 需可建立 GCP 專案與綁定帳單 |
| 帳單帳戶 | 已啟用付款方式（新用戶有 US$300 試用額度） |
| 權限 | 專案的「擁有者（Owner）」角色 |
| 本機環境 | 需安裝 Node.js 22+ 與 pnpm（僅用於前端建置與部署） |

### 命名規範（本文件統一使用）

建議照抄，之後每章的截圖與說明才對得上。

| 資源 | 名稱 | 說明 |
|------|------|------|
| 專案 ID | `cyberkonsult` | 可自訂，全球唯一 |
| 區域 | `asia-east1` | 台灣彰化 |
| 可用區 | `asia-east1-b` | 虛擬機所在 |
| 虛擬私有雲 | `oda-vpc` | 內部網路 |
| 子網路（虛擬機用） | `oda-subnet-main` | `10.10.0.0/24` |
| 子網路（Cloud Run 用） | `oda-subnet-run` | `10.10.1.0/26` |
| 資料庫執行個體 | `oda-cyber-db` | Cloud SQL |
| 虛擬機 | `oda-qdrant` | 向量資料庫主機 |
| 儲存桶 | `cyberkonsult-storage` | 需全球唯一 |
| 映像檔倉庫 | `cyber-konsult` | Artifact Registry |

### 進度檢查表

每完成一章請回來打勾，避免遺漏。

- [ ] 第 1 章：專案與 API
- [ ] 第 2 章：虛擬私有雲
- [ ] 第 3 章：Cloud SQL 資料庫
- [ ] 第 4 章：Qdrant 虛擬機
- [ ] 第 5 章：儲存桶與機密
- [ ] 第 6 章：程式碼調整（開發方）
- [ ] 第 7 章 7-1～7-3：容器映像建置（首次手動）
- [ ] 第 8 章：Cloud Run 部署
- [ ] **第 7 章 7-4**：回頭設定自動化建置與部署 ← 順序不可提前
- [ ] 第 9 章：前端部署
- [ ] 第 10 章：驗證與維運

> ⚠️ 第 7-4 章刻意排在第 8 章之後。自動化的最後一步是「更新既有服務的映像」，服務必須先存在才有意義。

---

## 第 1 章：建立專案與啟用 API

### 1-1 建立專案

1. 開啟 [Google Cloud Console](https://console.cloud.google.com)
2. 點畫面頂端的**專案選擇器**（網址列下方的下拉選單）
3. 點右上角「**新增專案**（New Project）」
4. 填寫後按「**建立**」：

| 欄位 | 填入值 |
|------|--------|
| 專案名稱 | `ODA Cyber Konsult` |
| 專案 ID | `cyberkonsult`（記下來，後面會一直用到） |
| 位置 | 選您的組織，或「無機構」 |

5. 建立後，**確認畫面頂端已切換到這個新專案**

### 1-2 綁定帳單帳戶

1. 左側 ☰ 導覽選單 → 「**帳單**（Billing）」
2. 若顯示「此專案沒有帳單帳戶」→ 點「**連結帳單帳戶**」
3. 選擇您的帳單帳戶 → 「**設定帳戶**」

> ⚠️ 沒有綁定帳單，後續所有服務都無法建立。

### 1-3 啟用必要的 API

需要啟用 9 個 API。逐一操作較慢，建議用批次方式：

1. 左側 ☰ → 「**API 和服務**（APIs & Services）」→「**已啟用的 API 和服務**」
2. 點上方「**+ 啟用 API 和服務**」
3. 在搜尋框逐一輸入下列名稱，點進去後按「**啟用**」：

| # | API 名稱 | 用途 |
|---|---------|------|
| 1 | Cloud Run Admin API | 部署容器服務 |
| 2 | Cloud SQL Admin API | 建立資料庫 |
| 3 | Compute Engine API | 建立 Qdrant 虛擬機 |
| 4 | Artifact Registry API | 存放容器映像檔 |
| 5 | Cloud Build API | 建置容器映像檔 |
| 6 | Secret Manager API | 存放機密設定 |
| 7 | Service Networking API | Cloud SQL 私有連線 |
| 8 | Cloud Storage API | 檔案儲存 |
| 9 | Cloud Logging API | 記錄與監控 |

> 💡 每個 API 啟用需 30～60 秒，可同時開多個分頁加速。

### 1-4 驗證

左側 ☰ →「API 和服務」→「已啟用的 API 和服務」，確認上表 9 項都在清單中。

---

## 第 2 章：建立虛擬私有雲

> **為什麼需要**：讓 Cloud Run、資料庫、Qdrant 在**私有網路內互通**，資料庫與 Qdrant 完全不對外開放，只有 Cloud Run 進得去。

### 2-1 建立虛擬私有雲網路

1. 左側 ☰ →「**VPC 網路**」→「**VPC 網路**」
2. 點上方「**建立 VPC 網路**」
3. 填寫：

| 欄位 | 填入值 |
|------|--------|
| 名稱 | `oda-vpc` |
| 子網路建立模式 | 選「**自訂**（Custom）」 |

4. 在「**新的子網路**」區塊填第一個子網路：

| 欄位 | 填入值 |
|------|--------|
| 名稱 | `oda-subnet-main` |
| 區域 | `asia-east1` |
| IPv4 範圍 | `10.10.0.0/24` |
| 專用 Google 存取權 | **開啟（On）** |

5. 點「**新增子網路**」，填第二個（給 Cloud Run 用）：

| 欄位 | 填入值 |
|------|--------|
| 名稱 | `oda-subnet-run` |
| 區域 | `asia-east1` |
| IPv4 範圍 | `10.10.1.0/26` |
| 專用 Google 存取權 | **開啟（On）** |

6. 其餘保持預設，點「**建立**」

> ⚠️ `oda-subnet-run` 的網段**必須是 /26 或更大**，這是 Cloud Run Direct VPC egress 的硬性要求。

### 2-2 建立防火牆規則

允許 Cloud Run 存取 Qdrant 的 6333 連接埠。

1. 左側 ☰ →「**VPC 網路**」→「**防火牆**」
2. 點「**建立防火牆規則**」
3. 填寫：

| 欄位 | 填入值 |
|------|--------|
| 名稱 | `allow-internal-qdrant` |
| 網路 | `oda-vpc` |
| 優先順序 | `1000` |
| 流量方向 | **輸入（Ingress）** |
| 目標 | 「**指定的目標標記**」 |
| 目標標記 | `qdrant` |
| 來源篩選器 | 「**IPv4 範圍**」 |
| 來源 IPv4 範圍 | `10.10.0.0/16` |
| 通訊協定和通訊埠 | 勾「**指定的通訊協定和通訊埠**」→ 勾 **TCP** → 填 `6333` |

4. 點「**建立**」

> 這條規則的意思：**只有** `10.10.x.x` 內部網段（也就是您的 Cloud Run 與虛擬機）能連 Qdrant，外部網際網路完全打不到。

### 2-3 保留私有服務連線範圍

Cloud SQL 使用私有 IP 時需要這一步。

> ⚠️ **這個功能不在左側選單裡**。必須先點進 `oda-vpc` 這個網路的詳細頁，上方分頁列才會出現「私人服務連線」。停在網路清單頁是找不到的。

1. 左側 ☰ →「**虛擬私有雲網路**」→「**虛擬私有雲網路**」（清單頁）
2. **點網路名稱 `oda-vpc`** 進入詳細頁（不是勾選前面的核取方塊）
3. 切到上方分頁列的「**私人服務連線**」
4. 子分頁「**已分配給服務的 IP 範圍**」→ 點「**分配 IP 範圍**」
5. 填寫：

| 欄位 | 填入值 |
|------|--------|
| 名稱 | `google-managed-services-oda-vpc` |
| IP 範圍 | 選「**自動**」，前置長度填 `16` |

6. 點「**分配**」
7. 切到子分頁「**與服務的私人連線**」→ 點「**建立連線**」
8. 「已指派的分配」勾選剛才建立的那筆 → 點「**連線**」（背景建立 VPC peering，需 1～3 分鐘）

> 💡 **Console 介面若又改版，改用等價指令**（`gcloud` 參數比 UI 穩定得多）：
>
> ```bash
> # 對應步驟 4~6：分配 IP 範圍
> gcloud compute addresses create google-managed-services-oda-vpc \
>   --global \
>   --purpose=VPC_PEERING \
>   --prefix-length=16 \
>   --network=oda-vpc \
>   --project=<你的專案 ID>
>
> # 對應步驟 7~8：建立私人連線
> gcloud services vpc-peerings connect \
>   --service=servicenetworking.googleapis.com \
>   --ranges=google-managed-services-oda-vpc \
>   --network=oda-vpc \
>   --project=<你的專案 ID>
> ```
>
> 前提是第 1-3 章的 **Service Networking API** 已啟用，否則第二道指令會直接失敗。

### 2-4 驗證

- VPC 網路清單中有 `oda-vpc`，底下有 2 個子網路
- 防火牆清單中有 `allow-internal-qdrant`
- 進入 `oda-vpc` 詳細頁 →「私人服務連線」分頁 →「與服務的私人連線」子分頁，狀態顯示為「**已連線**」

  或用指令確認（有輸出 `servicenetworking-googleapis-com` 即成功）：

  ```bash
  gcloud services vpc-peerings list --network=oda-vpc --project=<你的專案 ID>
  ```

---

## 第 3 章：建立 Cloud SQL 資料庫

### 3-1 建立執行個體

1. 左側 ☰ →「**SQL**」→ 點「**建立執行個體**」
2. 選「**PostgreSQL**」
3. 填寫基本設定：

| 欄位 | 填入值 |
|------|--------|
| 執行個體 ID | `oda-cyber-db` |
| 密碼 | 點「**產生**」自動產生強密碼，**務必立刻複製存好** |
| 資料庫版本 | `PostgreSQL 17`（若無此選項用 16） |
| Cloud SQL 版本 | **Enterprise** |
| 預設設定 | 選「**沙箱／開發**（Sandbox）」 |
| 區域 | `asia-east1`（台灣） |
| 可用區可用性 | **單一可用區**（POC 用；正式營運改「多可用區」） |

4. 展開「**自訂執行個體**」，調整：

| 區塊 | 設定 |
|------|------|
| 機器設定 | 共用核心 → **1 vCPU、1.7 GB**（db-g1-small） |
| 儲存空間 | SSD、**20 GB**、勾「**啟用自動增加儲存空間**」 |
| **連線** | ⚠️ 勾選「**私人 IP**」；**取消勾選「公開 IP」** |
| 網路（私人 IP 下方） | 選 `oda-vpc` |
| 資料備份 | 勾「**自動備份**」，時間設 `03:00`，保留 `7` 天 |
| 資料保護 | 勾「**啟用刪除防護**」 |

5. 點「**建立執行個體**」（需等 5～10 分鐘）

> ⚠️ **一定要取消「公開 IP」**。這是確保資料庫不對外曝露的關鍵設定。

### 3-2 建立資料庫與使用者

執行個體建好後：

1. 點進 `oda-cyber-db` → 左側「**資料庫**」→「**建立資料庫**」

| 欄位 | 填入值 |
|------|--------|
| 資料庫名稱 | `oda_cyber` |

2. 左側「**使用者**」→「**新增使用者帳戶**」

| 欄位 | 填入值 |
|------|--------|
| 使用者名稱 | `oda_app` |
| 密碼 | 產生強密碼並**存好** |

### 3-3 記下私有 IP

在執行個體「**總覽**」頁面找到「**私人 IP 位址**」（形如 `10.20.x.x`），**記下來**，第 5 章與第 8 章會用到。

### 3-4 驗證

| 檢查項 | 預期結果 |
|--------|---------|
| 執行個體狀態 | 綠色勾勾「可執行」 |
| 連線設定 | 只有私人 IP，**沒有**公開 IP |
| 資料庫清單 | 有 `oda_cyber` |
| 使用者清單 | 有 `oda_app` |

---

## 第 4 章：建立 Qdrant 虛擬機

> **為什麼 Qdrant 不放 Cloud Run**：Qdrant 是有狀態的資料庫，需要常駐記憶體索引與持久磁碟。Cloud Run 的容器隨時會被回收、磁碟不保留，放上去資料會遺失。

### 4-1 建立資料磁碟

先建立**獨立的資料磁碟**（與開機碟分開，虛擬機重建時資料不會消失）。

1. 左側 ☰ →「**Compute Engine**」→「**磁碟**」→「**建立磁碟**」
2. 填寫：

| 欄位 | 填入值 |
|------|--------|
| 名稱 | `oda-qdrant-data` |
| 類型 | **區域性**（Zonal） |
| 區域／可用區 | `asia-east1` / `asia-east1-b` |
| 磁碟來源類型 | **空白磁碟** |
| 磁碟類型 | **平衡永久磁碟**（Balanced persistent disk） |
| 大小 | `30` GB |

3. 點「**建立**」

### 4-2 建立虛擬機

1. 左側 ☰ →「**Compute Engine**」→「**VM 執行個體**」→「**建立執行個體**」
2. 基本設定：

| 欄位 | 填入值 |
|------|--------|
| 名稱 | `oda-qdrant` |
| 區域／可用區 | `asia-east1` / `asia-east1-b` |
| 機器系列 | **E2** |
| 機器類型 | **e2-medium**（2 vCPU、4 GB） |

3. 「**開機磁碟**」→ 點「**變更**」：

| 欄位 | 填入值 |
|------|--------|
| 作業系統 | **Container-Optimized OS** |
| 版本 | 最新的 stable |
| 開機磁碟類型 | 平衡永久磁碟 |
| 大小 | `20` GB |

4. 展開「**進階選項**」→「**磁碟**」→ 點「**附加現有磁碟**」→ 選 `oda-qdrant-data`，模式選「**讀取／寫入**」

5. 展開「**進階選項**」→「**網路**」：

| 欄位 | 填入值 |
|------|--------|
| 網路標記 | `qdrant`（對應第 2-2 章的防火牆規則） |
| 網路介面 → 網路 | `oda-vpc` |
| 網路介面 → 子網路 | `oda-subnet-main` |
| 網路介面 → 外部 IPv4 位址 | **臨時**（暫時保留，設定完成後移除） |

6. 展開「**進階選項**」→「**管理**」→ 在「**自動化**（啟動指令碼）」欄位貼上：

```bash
#!/bin/bash
# 格式化並掛載資料磁碟（僅首次執行時格式化）
DISK=/dev/disk/by-id/google-persistent-disk-1
MOUNT=/mnt/qdrant
mkdir -p $MOUNT
if ! blkid $DISK; then
  mkfs.ext4 -m 0 -F -E lazy_itable_init=0,lazy_journal_init=0,discard $DISK
fi
mount -o discard,defaults $DISK $MOUNT
mkdir -p $MOUNT/storage

# 啟動 Qdrant 容器（開機自動重啟）
docker run -d \
  --name qdrant \
  --restart always \
  -p 6333:6333 -p 6334:6334 \
  -v $MOUNT/storage:/qdrant/storage \
  qdrant/qdrant:latest
```

7. 點「**建立**」

### 4-3 確認 Qdrant 正常運作

1. 在 VM 執行個體清單，點 `oda-qdrant` 右側的「**SSH**」按鈕（開啟瀏覽器終端機）
2. 等 1～2 分鐘讓啟動指令碼跑完，然後輸入：

```bash
docker ps
curl http://localhost:6333/healthz
```

3. 預期看到 `qdrant` 容器在執行中，健康檢查回應正常
4. **記下虛擬機的內部 IP**（形如 `10.10.0.x`），第 8 章要用

### 4-4 移除外部 IP（強化安全）

映像檔已下載完成，可移除對外連線能力。

1. VM 執行個體清單 → 點 `oda-qdrant` →「**編輯**」
2. 找到「**網路介面**」→ 展開 → 「**外部 IPv4 位址**」改為「**無**」
3. 點「**儲存**」

> 移除後虛擬機無法連外網（也無法更新 Qdrant 版本）。日後要更新時，暫時加回外部 IP 即可。

### 4-5 設定每日自動快照

1. 左側 ☰ →「**Compute Engine**」→「**快照**」→「**快照排程**」分頁 →「**建立快照排程**」
2. 填寫：

| 欄位 | 填入值 |
|------|--------|
| 名稱 | `qdrant-daily-backup` |
| 區域 | `asia-east1` |
| 排程頻率 | **每天**，時間 `03:00`（台北時間） |
| 自動刪除快照時間 | `14` 天後 |
| 來源磁碟刪除政策 | **保留快照** |

3. 建立後，回到「**磁碟**」→ 點 `oda-qdrant-data` →「**編輯**」→「**快照排程**」選 `qdrant-daily-backup` → 儲存

### 4-6 驗證

| 檢查項 | 預期結果 |
|--------|---------|
| 虛擬機狀態 | 執行中，**無外部 IP** |
| SSH 內執行 `docker ps` | 看到 qdrant 容器 |
| 資料磁碟 | 已附加、已掛載於 `/mnt/qdrant` |
| 快照排程 | 已套用至 `oda-qdrant-data` |

---

## 第 5 章：建立儲存桶與機密

### 5-1 建立 Cloud Storage 儲存桶

用來存放上傳的檔案與清洗產出。

1. 左側 ☰ →「**Cloud Storage**」→「**值區**（Buckets）」→「**建立**」
2. 填寫：

| 欄位 | 填入值 |
|------|--------|
| 名稱 | `cyberkonsult-storage`（需全球唯一，可加後綴） |
| 位置類型 | **地區**（Region） |
| 位置 | `asia-east1` |
| 儲存空間級別 | **Standard** |
| 公開存取權 | 勾「**禁止公開存取**」 |
| 存取權控管 | **統一** |
| 保護工具 | 「**軟刪除政策**」保留 7 天 |

3. 點「**建立**」
4. 進入儲存桶，建立 3 個資料夾：`uploads`、`output`、`gdrive_temp`

### 5-2 建立機密

1. 左側 ☰ →「**Secret Manager**」→「**建立密鑰**」
2. 依下表逐一建立（每建立一個要重複這個步驟）：

| 密鑰名稱 | 內容 | 說明 |
|---------|------|------|
| `database-url` | `postgresql://oda_app:<密碼>@<CloudSQL私有IP>:5432/oda_cyber?schema=public` | NestJS 用 |
| `rag-database-url` | `postgresql+asyncpg://oda_app:<密碼>@<CloudSQL私有IP>:5432/oda_cyber` | RAG 服務用 |
| `jwt-secret` | 64 字元以上隨機字串 | 登入權杖簽章 |
| `internal-api-key` | 32 字元以上隨機字串 | NestJS ↔ RAG 內部認證 |
| `rag-internal-api-key` | 同上（**與上面填一樣的值**） | 同上 |
| `google-api-key` | 您的 Gemini API 金鑰 | LLM 與向量化 |
| `openai-api-key` | 您的 OpenAI API 金鑰（若使用） | 備援 LLM |
| `gdrive-encryption-key` | 現有 `.env` 中的 Fernet 金鑰 | Google 雲端硬碟憑證加密 |

> 💡 **產生隨機字串**：可用 Console 右上角的 Cloud Shell 執行 `openssl rand -base64 48`，或用任何密碼產生器。

建立時的欄位：

| 欄位 | 設定 |
|------|------|
| 名稱 | 照上表 |
| 密鑰值 | 貼上內容 |
| 複製政策 | **自動** |

### 5-3 建立服務帳戶

為 Cloud Run 服務建立專屬身分（最小權限原則）。

1. 左側 ☰ →「**IAM 與管理**」→「**服務帳戶**」→「**建立服務帳戶**」
2. 建立 2 個：

| 服務帳戶名稱 | ID | 用途 |
|-------------|-----|------|
| `ODA API Service` | `oda-api-sa` | NestJS |
| `ODA RAG Service` | `oda-rag-sa` | FastAPI |

3. 建立時在「**授予這個服務帳戶專案存取權**」步驟加入角色：

| 服務帳戶 | 需要的角色 |
|---------|-----------|
| `oda-api-sa` | Secret Manager 密鑰存取者、Cloud SQL 用戶端、Cloud Run 叫用者 |
| `oda-rag-sa` | Secret Manager 密鑰存取者、Cloud SQL 用戶端、**Storage 物件使用者** |

> `oda-rag-sa` 需要 Storage 權限，因為它要讀寫掛載的儲存桶。

### 5-4 驗證

| 檢查項 | 預期結果 |
|--------|---------|
| 儲存桶 | 已建立，含 3 個資料夾，禁止公開存取 |
| Secret Manager | 8 個密鑰（若不用 OpenAI 則 7 個） |
| 服務帳戶 | 2 個，角色已授予 |

---

## 第 6 章：程式碼調整（開發方負責）

> ✅ **本章已於 2026-08-03 完成**，以下保留說明供理解架構之用。您不需在 Console 操作。
> 驗證結果：Python 測試 972 項全數通過；兩種後端在真實 PostgreSQL 上行為一致（7 項一致性測試通過）。

### 6-1 為什麼需要改

| 元件 | 目前做法 | 在 Cloud Run 上的問題 |
|------|---------|---------------------|
| BM25 關鍵字索引 | 本地 SQLite 檔案 | 多個執行個體同時寫入會**損毀索引** |
| Embedding 快取 | 本地 SQLite 檔案 | 同上；且 Cloud Run 檔案系統佔用記憶體 |

Google 官方文件明載：Cloud Storage 掛載**不提供檔案鎖定機制**，而 SQLite 依賴檔案鎖定來保護交易完整性。

### 6-2 改法：雙後端設計

```
        BM25Store（共用介面）
        ├── SQLite 實作     ← 本機開發與測試（維持現狀）
        └── PostgreSQL 實作 ← Cloud Run 使用（新增）

     以環境變數切換：RAG_BM25_BACKEND=sqlite | postgres
```

| 環境 | 使用的後端 | 影響 |
|------|-----------|------|
| 本機開發 | SQLite（預設） | **與現在完全相同**，離線可開發 |
| 既有測試 | SQLite | **810 個測試一行都不用改** |
| Cloud Run | PostgreSQL | 多執行個體安全 |

### 6-3 實際異動（已完成）

| 檔案 | 異動 |
|------|------|
| `indexing/bm25_base.py` | **新增** 共用介面（Protocol） |
| `indexing/bm25_store_pg.py` | **新增** PostgreSQL 實作（psycopg2 連線池） |
| `indexing/bm25_factory.py` | **新增** 後端選擇工廠 |
| `cache/embedding_cache_pg.py` | **新增** PostgreSQL 快取 |
| `cache/factory.py` | **新增** 後端選擇工廠 |
| `alembic/versions/013_add_bm25_index_tables.py` | **新增** 建表 migration |
| `tests/test_bm25_contract.py` | **新增** 雙後端一致性測試 |
| `tests/test_backend_factory.py` | **新增** 工廠與預設值測試 |
| `config.py` | 新增 `bm25_backend`、`cache_backend`（預設 `sqlite`） |
| `api/main.py`、`cli/{ingest,query,gdrive}.py` | 改用工廠建立 |
| `indexing/bm25_store.py`、`cache/embedding_cache.py` | **一行未改**（Protocol 為結構型別） |

> 部署前必須先執行 `alembic upgrade head` 建立 4 張新表，否則服務啟動時會明確報錯提示。

### 6-4 檔案儲存不需改程式碼

上傳與清洗產出的檔案路徑本來就由環境變數控制，部署時改指向掛載點即可：

| 環境變數 | 本機值 | Cloud Run 值 |
|---------|--------|-------------|
| `RAG_UPLOAD_DIR` | `data/uploads` | `/mnt/storage/uploads` |
| `RAG_OUTPUT_DIR` | `data/output` | `/mnt/storage/output` |
| `RAG_GDRIVE_TEMP_DIR` | `data/gdrive_temp` | `/mnt/storage/gdrive_temp` |

### 6-5 完成判準

- [ ] 本機執行 `./scripts/dev-start.sh` 行為與改動前完全相同
- [ ] Python 測試全數通過（原有 810 項 + 新增一致性測試）
- [ ] 設定 `RAG_BM25_BACKEND=postgres` 後可連 PostgreSQL 正常檢索

---

## 第 7 章：建置容器映像

### 7-1 建立映像檔倉庫

1. 左側 ☰ →「**Artifact Registry**」→「**建立存放區**」
2. 填寫：

| 欄位 | 填入值 |
|------|--------|
| 名稱 | `cyber-konsult` |
| 格式 | **Docker** |
| 模式 | **標準** |
| 位置類型 | **地區** |
| 地區 | `asia-east1` |

3. 點「**建立**」

### 7-2 首次建置映像（手動）

> **為什麼首次要手動**：第 8 章需要先有映像才能建立 Cloud Run 服務。自動化建置放在第 7-4 章，**必須等第 8 章的服務建好之後**才設定——順序顛倒會出問題，原因見 7-4 開頭說明。

Console 沒有「上傳本機程式碼」的按鈕，最簡單的方式是用畫面右上角的 **Cloud Shell**（瀏覽器內建終端機，不需在本機安裝任何東西）。

1. 點 Console 右上角的「**啟用 Cloud Shell**」圖示（`>_` 符號）
2. 等待終端機開啟後，將專案程式碼複製進來：

```bash
git clone https://github.com/puppychen/oda-CyberKonsult.git
cd oda-CyberKonsult
```

3. 執行下列 3 行：

```bash
gcloud builds submit --tag asia-east1-docker.pkg.dev/cyberkonsult/cyber-konsult/oda-api:v1 --file apps/api/Dockerfile .

gcloud builds submit --tag asia-east1-docker.pkg.dev/cyberkonsult/cyber-konsult/oda-rag:v1 --file python/rag-service/Dockerfile .

gcloud builds submit --tag asia-east1-docker.pkg.dev/cyberkonsult/cyber-konsult/oda-searxng:v1 docker/searxng/
```

> 💡 前兩行結尾的 `.` 是關鍵——它把**整個專案根目錄**當作建置來源。這兩個 Dockerfile 需要根目錄的 `pnpm-lock.yaml`、`packages/`、`contracts/` 等檔案，若只指定 Dockerfile 所在目錄會建置失敗。第 3 行的 SearXNG 則相反，只需要它自己的目錄。

### 7-3 驗證

左側 ☰ →「Artifact Registry」→ 點 `cyber-konsult`，應看到 3 個映像檔各有 `v1` 標籤。

### 7-4 設定自動化建置與部署

> ⚠️ **請先完成第 8 章，再回來執行本節。**
>
> 自動化流程的最後一步是「更新 Cloud Run 服務的映像」，它只會換映像、**不會建立服務設定**。若服務還不存在就啟用自動化，系統會建出一個沒有環境變數、沒有密鑰、沒有虛擬私有雲連線的空服務，等於白做一次還要砍掉重來。

設定完成後，日常發版只需要打一個 git tag，系統就會自動建置並更新對應服務。

#### 步驟一：授權 GitHub

1. 左側 ☰ →「**Cloud Build**」→「**觸發條件**」
2. 點「**連結存放區**」→ 來源選「**GitHub**」
3. 依畫面完成 GitHub 授權（會安裝 Google Cloud Build 應用程式），選擇 `oda-CyberKonsult` 存放區

#### 步驟二：授予部署權限

自動部署需要讓 Cloud Build 有權更新 Cloud Run 服務。

1. 左側 ☰ →「**IAM 與管理**」→「**IAM**」
2. 找到 Cloud Build 使用的服務帳戶（名稱含 `cloudbuild` 或專案編號的預設帳戶）
3. 點編輯（鉛筆圖示），新增以下兩個角色：

| 角色 | 用途 |
|------|------|
| **Cloud Run Admin** | 更新 Cloud Run 服務的映像 |
| **服務帳戶使用者**（Service Account User） | 以服務本身的身分執行部署 |

#### 步驟三：建立三個觸發器

「**觸發條件**」→「**建立觸發條件**」，依下表建立 3 個。三者的「事件」皆選「**推送新標記**」，「設定」皆選「**Cloud Build 設定檔**」：

| 觸發器名稱 | 標記（正規表示式） | 設定檔位置 | 對應服務 |
|-----------|------------------|-----------|---------|
| `deploy-api` | `^api-v[0-9]+\.[0-9]+\.[0-9]+$` | `/cloudbuild.api.yaml` | `oda-api` |
| `deploy-rag` | `^rag-v[0-9]+\.[0-9]+\.[0-9]+$` | `/cloudbuild.rag.yaml` | `oda-rag` |
| `deploy-searxng` | `^searxng-v[0-9]+\.[0-9]+\.[0-9]+$` | `/cloudbuild.searxng.yaml` | `oda-searxng` |

### 7-5 日常發版

三個服務**各自獨立發版**，改了哪個就發哪個。有兩種發版方式，依需不需要版本號選用。

#### 方式一：日常建置（免版本號）

固定使用服務名稱作為標籤，每次重新指向最新的 commit。**映像會以 commit 代碼標記**，例如 `oda-api:a1b2c3d`，可追溯到確切的原始碼版本，不需要自己維護版本號。

```bash
# 以更新後端 API 為例（RAG 換成 oda-rag、SearXNG 換成 oda-searxng）
git push origin :refs/tags/oda-api   # 1. 先刪除遠端的舊標籤
git tag -d oda-api                   # 2. 刪除本機的舊標籤
git tag oda-api                      # 3. 在目前的 commit 重新打標籤
git push origin oda-api              # 4. 推送，隨即觸發建置
```

> ⚠️ **順序不可顛倒**。遠端的舊標籤還在時，推送同名標籤會被拒絕（Git 不允許標籤指向不同 commit）。第 1 步的 `:refs/tags/` 寫法就是「刪除遠端標籤」的意思。

> 💡 刪除標籤**不會**觸發建置，只有第 4 步的推送會。

#### 方式二：正式發布（帶版本號）

需要對外標示版本時使用。**映像會以版本號標記**，例如 `oda-api:1.0.1`。

```bash
git tag api-v1.0.1
git push origin api-v1.0.1
```

> 💡 前綴 `api-v` 會被自動去掉，所以 `api-v1.0.1` 產生的映像是 `oda-api:1.0.1`。前綴的作用是決定「這個標籤要觸發哪一個服務」。

---

兩種方式推送後，系統都會自動完成三件事：**建置映像 → 存入 Artifact Registry → 更新對應的 Cloud Run 服務**。到「Cloud Build」→「記錄」可看進度，約需 8～15 分鐘。

每個映像除了上述主要標籤外，還會額外標記 `latest` 與 commit 代碼，方便回溯。

> ⚠️ **標籤必須符合觸發器的規則**，否則不會有任何動作，也不會有錯誤通知。允許的寫法只有 `oda-api` 與 `api-v1.0.1` 這兩種格式；打成 `api-v1.0`（少一段數字）或 `apiv1.0.1`（少了連字號）都不會觸發。推送後請到「Cloud Build」→「記錄」確認有新的建置出現。

> ⚠️ **標籤指向哪個 commit，就用那個版本的設定檔建置**。若剛修改過 `cloudbuild.*.yaml`，必須先把修改推送到主分支，再打標籤——否則系統讀到的仍是舊設定。

> 📌 **想要只建映像、不自動上線**：把設定檔中最後一個名為 `deploy` 的步驟整段刪除或註解掉即可。之後改為到 Cloud Run 手動選擇映像版本部署。

---

## 第 8 章：部署 Cloud Run 服務

> **部署順序很重要**：先 RAG 服務 → 再 NestJS API（因為 API 需要填入 RAG 的網址）。

> 📌 **本章只需要做一次**。這裡建立的環境變數、密鑰、虛擬私有雲連線、磁碟區掛接等設定會長期保留；日後改程式碼只是換一個新映像，這些設定不會被動到。設定完成後請回到第 7-4 章啟用自動化，之後就不必再進 Cloud Run 手動操作。

### 8-1 部署 RAG 服務

1. 左側 ☰ →「**Cloud Run**」→「**部署容器**」→「**服務**」
2. **容器映像檔網址**：點「選取」→ 從 Artifact Registry 選 `oda-rag:v1`
3. 基本設定：

| 欄位 | 填入值 |
|------|--------|
| 服務名稱 | `oda-rag` |
| 區域 | `asia-east1` |
| 驗證 | **允許未經驗證的叫用**（靠內部網路限制保護） |
| 輸入 | **內部**（Internal） |

4. 展開「**容器、磁碟區、網路、安全性**」

**「容器」分頁：**

| 欄位 | 填入值 |
|------|--------|
| 容器通訊埠 | `3502` |
| 記憶體 | `2 GiB` |
| CPU | `1` |
| **CPU 分配** | **一律分配 CPU**（重要！背景清洗任務需要） |
| 執行要求逾時 | `900` 秒 |
| 執行個體數量下限 | `1`（避免冷啟動中斷背景任務） |
| 執行個體數量上限 | `1`（POC 階段；WebSocket 廣播需單一執行個體） |

**「變數和密鑰」分頁** — 環境變數：

| 名稱 | 值 |
|------|-----|
| `RAG_QDRANT_HOST` | 第 4-3 章記下的虛擬機內部 IP |
| `RAG_QDRANT_PORT` | `6333` |
| `RAG_QDRANT_COLLECTION` | `oda_documents` |
| `RAG_BM25_BACKEND` | `postgres` |
| `RAG_UPLOAD_DIR` | `/mnt/storage/uploads` |
| `RAG_OUTPUT_DIR` | `/mnt/storage/output` |
| `RAG_GDRIVE_TEMP_DIR` | `/mnt/storage/gdrive_temp` |
| `RAG_DEBUG` | `false` |
| `RAG_API_PORT` | `3502` |
| `RAG_EMBEDDING_PROVIDER` | `gemini` |
| `RAG_EMBEDDING_DIMENSION` | `1536` |
| `SEARXNG_URL` | 暫留空，第 8-3 章回來補 |

**同一分頁** — 參照密鑰（點「參照密鑰」按鈕）：

| 環境變數名稱 | 選擇的密鑰 | 版本 |
|------------|-----------|------|
| `RAG_DATABASE_URL` | `rag-database-url` | 最新 |
| `RAG_GOOGLE_API_KEY` | `google-api-key` | 最新 |
| `RAG_INTERNAL_API_KEY` | `rag-internal-api-key` | 最新 |
| `RAG_GDRIVE_ENCRYPTION_KEY` | `gdrive-encryption-key` | 最新 |

**「磁碟區」分頁：**

1. 點「**掛接磁碟區**」（Mount volume）
2. 磁碟區類型選「**Cloud Storage 值區**」

| 欄位 | 填入值 |
|------|--------|
| 磁碟區名稱 | `storage` |
| 值區 | `cyberkonsult-storage` |
| 唯讀 | **不勾**（需要寫入） |

3. 切到「**容器**」分頁 →「**磁碟區掛接**」→ 點「**掛接磁碟區**」

| 欄位 | 填入值 |
|------|--------|
| 磁碟區名稱 | `storage` |
| 掛接路徑 | `/mnt/storage` |

**「網路」分頁：**

1. 勾選「**連線至虛擬私有雲以進行輸出流量**」
2. 選「**直接將流量傳送至虛擬私有雲**」（Send traffic directly to a VPC）

| 欄位 | 填入值 |
|------|--------|
| 網路 | `oda-vpc` |
| 子網路 | `oda-subnet-run` |
| 流量轉送 | **僅將要求轉送至私人 IP** |

**「安全性」分頁：**

| 欄位 | 填入值 |
|------|--------|
| 服務帳戶 | `oda-rag-sa` |

5. 點「**建立**」（約需 2～3 分鐘）
6. 部署完成後**記下服務網址**（形如 `https://oda-rag-xxxxx.asia-east1.run.app`）

### 8-2 部署 NestJS API

同樣流程，差異如下：

1. 容器映像檔選 `oda-api:v1`

| 欄位 | 填入值 |
|------|--------|
| 服務名稱 | `oda-api` |
| 驗證 | **允許未經驗證的叫用**（前端需存取，靠 JWT 保護） |
| 輸入 | **全部**（All） |
| 容器通訊埠 | `3051` |
| 記憶體 | `512 MiB` |
| CPU | `1` |
| CPU 分配 | 僅在處理要求時分配 |
| 執行個體數量下限 | `0`（可縮至零省錢） |
| 執行個體數量上限 | `3` |

**環境變數：**

| 名稱 | 值 |
|------|-----|
| `NODE_ENV` | `production` |
| `PORT` | `3051` |
| `FASTAPI_BASE_URL` | 第 8-1 章記下的 RAG 服務網址 |
| `CORS_ORIGINS` | 暫留空，第 9 章回來補 |
| `PASSWORD_MAX_AGE_DAYS` | `90` |
| `LLM_PROVIDER` | `gemini` |
| `LLM_MODEL_GEMINI` | `gemini-2.0-flash` |

**參照密鑰：**

| 環境變數名稱 | 密鑰 |
|------------|------|
| `DATABASE_URL` | `database-url` |
| `JWT_SECRET` | `jwt-secret` |
| `INTERNAL_API_KEY` | `internal-api-key` |
| `GOOGLE_API_KEY` | `google-api-key` |

**網路分頁**：同 8-1（Direct VPC egress，選 `oda-vpc` / `oda-subnet-run`）
**安全性分頁**：服務帳戶選 `oda-api-sa`

點「建立」，完成後**記下服務網址**。

### 8-3 部署 SearXNG

| 欄位 | 填入值 |
|------|--------|
| 容器映像檔 | `oda-searxng:v1` |
| 服務名稱 | `oda-searxng` |
| 驗證 | 允許未經驗證的叫用 |
| 輸入 | **內部** |
| 容器通訊埠 | `8080` |
| 記憶體 | `512 MiB` |
| 執行個體數量下限 | `0` |
| 執行個體數量上限 | `2` |

部署完成後，**回頭補上前兩個服務的環境變數**：

1. 進入 `oda-rag` →「編輯並部署新修訂版本」→ 把 `SEARXNG_URL` 填入 SearXNG 的服務網址 → 部署

### 8-4 執行資料庫遷移

資料表尚未建立，需執行一次遷移。用 Cloud Shell：

```bash
# 在 Cloud Shell 中，於專案目錄執行
cd apps/api
export DATABASE_URL="<第 5-2 章的 database-url 內容>"
pnpm install
pnpm prisma migrate deploy
pnpm seed          # 建立預設帳號與提示詞範本
```

> ⚠️ Cloud Shell 需能連到 Cloud SQL 私有 IP。若連不上，改用「Cloud SQL Studio」（Console 內建的查詢介面）手動執行，或暫時開啟公開 IP 並限制來源 IP，完成後關閉。

### 8-5 驗證

| 檢查項 | 做法 | 預期結果 |
|--------|------|---------|
| API 健康檢查 | 瀏覽器開 `<oda-api網址>/health` | 回傳正常狀態 |
| RAG 服務 | Cloud Run 記錄檔無錯誤 | 啟動成功訊息 |
| 資料庫連線 | Cloud SQL Studio 查 `users` 資料表 | 有 7 筆預設帳號 |
| Qdrant 連線 | RAG 記錄檔 | 無連線逾時錯誤 |

### 8-6 後續更新方式

三個服務都建好並驗證通過後，**回到第 7-4 章設定自動化**。之後更新程式碼只需要打一個 git tag，不必再手動進 Cloud Run。

兩種更新方式的差別：

| 情境 | 做法 |
|------|------|
| **改了程式碼**（換新版本） | 打 tag（第 7-5 章），系統自動建置並更新映像 |
| **改了設定**（環境變數、密鑰、記憶體、執行個體數量） | 進 Cloud Run →「編輯並部署新修訂版本」手動調整 |

> 💡 自動部署只會更換映像，不會覆寫你在本章設定的環境變數與密鑰，兩者互不干擾。

> ⚠️ **服務被誤刪時不能只靠 tag 救回**。自動部署預期服務已存在，若 Cloud Run 服務被刪除，重新打 tag 只會建出一個沒有任何設定的空服務——必須回到本章重新完整設定一次。

**若新版本有問題需要退回**：進 Cloud Run → 該服務 →「修訂版本」分頁 → 選擇前一個正常的版本 →「管理流量」把 100% 流量切回去。這比重新建置快得多。

---

## 第 9 章：部署前端

三個前端（Admin、Chatbot、Cleaner）部署到 Firebase Hosting。

### 9-1 確認 Firebase 專案

本專案的 Firebase 已啟用，專案 ID 為 `cyberkonsult`，可直接跳至 9-2。

**若需在新環境重建**：

1. 開啟 [Firebase Console](https://console.firebase.google.com)
2. 點「**新增專案**」
3. ⚠️ **重要**：在「輸入專案名稱」時，**選擇既有的 GCP 專案**（下拉選單會列出）。務必選既有專案，不要新建——Firebase 與 GCP 必須是同一個專案，Hosting 才能與其他 GCP 資源共用權限與帳單
4. 依畫面指示完成（Google Analytics 可略過）
5. 左側「**建構**」→「**Hosting**」→ 點「**開始使用**」

### 9-2 在 Cloud Shell 部署

1. 回到 GCP Console，開啟 **Cloud Shell**
2. 在專案目錄執行：

```bash
npm install -g firebase-tools
firebase login --no-localhost     # 依畫面指示完成授權
firebase use cyberkonsult
```

3. **設定檔已在專案裡，不需手動建立**：

| 檔案 | 作用 |
|------|------|
| `firebase.json` | 三個站台的 Hosting 設定（public 目錄、SPA 轉址、快取標頭） |
| `.firebaserc` | 專案關聯與站台代號綁定 |
| `.env.production`（專案根目錄） | **三個前端共用的正式環境變數** |

> 💡 **環境變數只有一個來源**。三個前端的 `vite.config.ts` 都設定 `envDir` 指向專案根目錄，因此建置時會讀取根目錄的 `.env.production`，各應用目錄下不放任何 `.env`。只有 `VITE_` 開頭的變數會被編譯進瀏覽器，其餘欄位不會外洩到前端。

> ⚠️ **`.env.production` 必須列出全部 `VITE_` 變數**。Vite 會先載入 `.env`（開發用的 localhost 值）再以 `.env.production` 覆蓋，漏掉任何一個變數就會沿用開發值，而且不會有任何警告。

> 💡 **前端直接呼叫後端網域**，不透過 Hosting 轉送。`firebase.json` 的轉址規則只負責單頁應用的路由（所有路徑都交給 `index.html`）。因此**後端必須將前端網址加入 CORS 白名單**，見第 9-3 章。靜態資源（`assets/**`）設為長期快取、其餘路徑設為不快取，確保改版後使用者立即拿到新版。

> ⚠️ **若前端網址與預設不同**，請先修改 `.env.production` 中的 `VITE_ADMIN_URL` / `VITE_CLEANER_URL`。這兩個值控制 Admin 與 Cleaner 之間的跳轉，填錯會導致跳轉連到錯誤位址。

4. 在 Firebase Console →「Hosting」**新增 2 個網站**：`cyberkonsult-admin`、`cyberkonsult-cleaner`

> 💡 `cyberkonsult` 是專案的預設站台，建立專案時就自動存在，不需另外新增；本文件將它配置給 Chatbot（面向終端使用者，網址最簡潔）。

5. 在 Cloud Shell 綁定站台代號（**只需執行一次**）：

```bash
firebase target:apply hosting chatbot cyberkonsult
firebase target:apply hosting admin   cyberkonsult-admin
firebase target:apply hosting cleaner cyberkonsult-cleaner
```

6. 建置與部署。**三個站台各自獨立**，可以只更新其中一個：

```bash
# ---- Chatbot ----
pnpm --filter @oda-cyber/chatbot build
firebase deploy --only hosting:chatbot

# ---- Admin ----
pnpm --filter @oda-cyber/admin build
firebase deploy --only hosting:admin

# ---- Cleaner ----
pnpm --filter @oda-cyber/cleaner build
firebase deploy --only hosting:cleaner
```

需要三個一次全部更新時：

```bash
pnpm --filter "@oda-cyber/chatbot" --filter "@oda-cyber/admin" --filter "@oda-cyber/cleaner" build
firebase deploy --only hosting
```

> 💡 建置指令**不需要**再手動帶 `VITE_API_BASE_URL=`。Vite 在 production 模式會自動讀取專案根目錄的 `.env.production`。

> 📌 **前端與後端部署互相獨立**：前端以上述指令部署，後端三個服務走各自的 Cloud Build 設定檔（專案根目錄的 `cloudbuild.api.yaml`、`cloudbuild.rag.yaml`、`cloudbuild.searxng.yaml`，設定方式見各檔開頭註解）。只改前端畫面時不需要動後端；只改後端時，前端也不必重新部署。

### 9-3 回填 CORS 設定

前端直接呼叫後端網域，因此**後端必須放行這三個前端網址**，否則瀏覽器會擋下所有 API 請求——畫面打得開，但登入與所有功能都會失敗。

**設定兩個環境變數**（皆為逗號分隔，結尾不可有斜線）：

```bash
CORS_ORIGINS=https://cyberkonsult.web.app,https://cyberkonsult-admin.web.app,https://cyberkonsult-cleaner.web.app

# WebSocket（清洗任務進度推播）。僅 Admin 與 Cleaner 使用，Chatbot 不需要。
WS_CORS_ORIGINS=https://cyberkonsult-admin.web.app,https://cyberkonsult-cleaner.web.app
```

**設定位置依後端的部署方式而定：**

| 後端部署方式 | 設定位置 | 生效方式 |
|-------------|---------|---------|
| Cloud Run | 服務 →「編輯並部署新修訂版本」→ 環境變數 | 部署新修訂版本 |
| 以 `scripts/dev-start.sh` 啟動 | **該機器的 `.env`** | 重新啟動服務 |
| Docker Compose | `docker-compose.prod.yml` 的 `environment` 或 `env_file` | `docker compose up -d` |

> ⚠️ **`scripts/dev-start.sh` 只會載入 `.env`**（見其 `load_env_file()`），**不會讀 `.env.production`**。以該腳本啟動服務時，設定必須寫在 `.env`，寫在 `.env.production` 不會生效。

> ⚠️ **改完必須重新啟動服務**。後端在程序啟動時讀取環境變數（`apps/api/src/config/cors.config.ts`），執行中修改設定檔不會生效。

**驗證是否生效**（不需開瀏覽器）：

```bash
curl -sI -X OPTIONS https://<你的後端網域>/api/v1/auth/login \
  -H "Origin: https://cyberkonsult-admin.web.app" \
  -H "Access-Control-Request-Method: POST" | grep -i access-control-allow-origin
```

有回傳 `access-control-allow-origin` 那一行即表示放行成功；沒有任何輸出就是還沒生效。

### 9-4 驗證

| 檢查項 | 預期結果 |
|--------|---------|
| 開啟 Chatbot 網址 | 顯示登入畫面 |
| 用預設帳號登入 | 成功進入聊天介面 |
| 送出一個問題 | 有串流回覆（可能需先匯入知識庫） |
| 瀏覽器開發者工具 | 無 CORS 錯誤 |

---

## 第 10 章：驗證與維運

### 10-1 整體功能驗收

| # | 情境 | 操作 | 預期結果 |
|---|------|------|---------|
| 1 | 登入 | 用 `admin@oda-cyber.com` 登入 Admin | 進入管理後台 |
| 2 | 聊天 | Chatbot 提問「什麼是釣魚信件」 | 逐字串流回覆 |
| 3 | 知識庫檢索 | 同上，觀察回覆下方 | 顯示引用來源與信心度標籤 |
| 4 | 檔案上傳 | Cleaner 上傳測試檔案 | 上傳成功，出現在清單 |
| 5 | 清洗流程 | 啟動清洗任務 | 進度更新，產出可下載 |
| 6 | 稽核紀錄 | Admin 查看稽核日誌 | 有上述操作的紀錄 |

### 10-2 匯入知識庫資料

現有種子資料需匯入 Qdrant。用 Cloud Shell：

```bash
cd python/rag-service
# 設定連線至 Cloud Run 的 RAG 服務，或直接在 Cloud Shell 連 Qdrant 內部 IP
# 詳細指令請洽開發方（需帶 X-Internal-Token）
```

> 開發方會另外提供匯入腳本與操作說明。

### 10-3 設定監控告警

1. 左側 ☰ →「**Monitoring**」→「**快訊**」→「**建立政策**」
2. 建議至少設定 3 條：

| 告警名稱 | 條件 | 通知門檻 |
|---------|------|---------|
| Cloud Run 錯誤率過高 | 5xx 回應比例 | 連續 5 分鐘 > 5% |
| Cloud SQL 連線數過高 | 連線數 | > 80% 上限 |
| Qdrant 虛擬機停止 | 執行個體正常運作時間 | 中斷 > 2 分鐘 |

3. 通知管道設定您的電子郵件

### 10-4 成本控管

1. 左側 ☰ →「**帳單**」→「**預算與快訊**」→「**建立預算**」

| 欄位 | 建議值 |
|------|--------|
| 預算金額 | US$150／月 |
| 快訊門檻 | 50%、90%、100% |

### 10-5 月費明細（預估）

| 項目 | 規格 | 月費（US$） |
|------|------|-----------|
| Cloud SQL | db-g1-small + 20GB SSD | ~28 |
| GCE 虛擬機 | e2-medium + 30GB 磁碟 + 快照 | ~32 |
| Cloud Run（RAG） | 1 vCPU / 2GB，常駐 1 執行個體 | ~35 |
| Cloud Run（API + SearXNG） | 可縮至零 | ~5 |
| Cloud Storage | Standard，用量小 | ~1 |
| Artifact Registry | 3 個映像檔 | ~1 |
| Secret Manager | 8 個密鑰 | <1 |
| Firebase Hosting | 免費額度內 | 0 |
| **合計** | | **~103** |

**省錢調整**：若可接受背景清洗任務延遲，將 `oda-rag` 的「執行個體數量下限」改為 `0`、CPU 分配改為「僅在處理要求時」，可省約 US$30／月。

### 10-6 日常維運

| 工作 | 頻率 | 做法 |
|------|------|------|
| 檢查備份 | 每週 | Cloud SQL →「備份」分頁；Compute Engine →「快照」 |
| 檢視錯誤記錄 | 每週 | Cloud Run → 各服務 →「記錄」分頁 |
| 更新 Qdrant | 每季 | 暫時加回外部 IP → SSH → `docker pull` + 重啟容器 → 移除外部 IP |
| 更新應用程式 | 依需求 | 重新執行第 7 章建置（改用 `v2` 標籤）→ Cloud Run 部署新修訂版本 |
| 檢視成本 | 每月 | 帳單 →「報表」 |

### 10-7 疑難排解

| 症狀 | 可能原因 | 檢查方向 |
|------|---------|---------|
| Cloud Run 顯示啟動失敗 | 環境變數缺漏、密鑰權限不足 | 看「記錄」分頁的錯誤訊息；確認服務帳戶有 Secret Manager 存取權 |
| API 連不到資料庫 | Direct VPC egress 未設定 | 檢查「網路」分頁是否已選 `oda-vpc` / `oda-subnet-run` |
| RAG 連不到 Qdrant | 防火牆或 IP 錯誤 | 確認防火牆規則的目標標記為 `qdrant`；確認填的是**內部** IP |
| 前端呼叫 API 失敗 | CORS 未放行，或 API 網址錯誤 | 先用 9-3 章的 `curl` 驗證 CORS；再確認 `.env.production` 的 `VITE_API_BASE_URL` 與實際後端網域相符（改動後需重新建置前端） |
| 檔案上傳失敗 | 儲存桶掛載或權限 | 確認「磁碟區」分頁掛載路徑為 `/mnt/storage`；確認服務帳戶有 Storage 物件使用者角色 |
| 清洗任務卡住不動 | CPU 被節流 | 確認 `oda-rag` 的 CPU 分配為「**一律分配**」且執行個體下限為 `1` |

### 10-8 已知待辦

| # | 項目 | 負責方 | 狀態 |
|---|------|--------|------|
| 1 | BM25／Embedding 雙後端改造 | 開發方 | ✅ **2026-08-03 完成**（見第 6 章） |
| 2 | `docker/searxng/Dockerfile` | 開發方 | ✅ **2026-08-03 完成**（設定檔本就存在，補上 Cloud Run 用的 Dockerfile 並修正 limiter 設定） |
| 3 | 知識庫匯入腳本 | 開發方 | 📋 待辦，見 10-2 |
| 4 | 自訂網域與憑證 | 依需求 | 📋 目前使用 Firebase 預設網址 |
| 5 | BM25 重新索引留下舊詞條 | 開發方 | ✅ **2026-08-03 修正**：`add_document` 於寫入前先清除該 `doc_id` 的舊詞條，兩後端同步修正。已加回歸測試 `TestReindexDropsStaleTerms`（撤掉修正即失敗，確認非偽測試） |

---

## 附錄：升級到正式營運

本文件為 POC 規格。日後升級為正式營運時，**架構不需重建**，只需調整下列參數：

| 項目 | POC 設定 | 正式營運設定 |
|------|---------|-------------|
| Cloud SQL 可用性 | 單一可用區 | **多可用區**（自動容錯移轉） |
| Cloud SQL 規格 | db-g1-small | db-custom-2-7680 或以上 |
| Cloud Run（API）下限 | 0 | **1**（消除冷啟動） |
| Cloud Run（RAG）上限 | 1 | 3 以上（需先將 WebSocket 改為 Pub/Sub 廣播） |
| 背景清洗任務 | in-process | 改用 Cloud Tasks + Cloud Run Jobs |
| Qdrant | 單機 | 多節點叢集或改用 Qdrant Hybrid Cloud |
| 監控 | 基本告警 | 完整 SLI／SLO 與值班流程 |
| 前端網域 | Firebase 預設 | 自訂網域 + 憑證 |

---

## 相關文件

| 文件 | 路徑 |
|------|------|
| 本機開發環境設定 | `docs/03-operations/dev-environment-setup.md` |
| 既有 Docker 部署指南 | `docs/03-operations/deployment.md` |
| 維運手冊 | `docs/03-operations/runbook.md` |
| 備份策略 | `docs/03-operations/backup-strategy.md` |
| 災害復原檢查表 | `docs/03-operations/dr-checklist.md` |
