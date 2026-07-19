---
audience: ai-primary
purpose: spec
status: approved
owner: ODA Cyber Konsult
---

# 資料去識別化系統 API 文件

> 本文件涵蓋 RAG 資料去識別化子系統的完整 API 規格，包含檔案上傳、去識別化任務、結果下載及 WebSocket 即時通知。去識別化規則由系統內建固定規則 `get_default_rules()` 提供，不可動態配置。
>
> **使用範圍**：本 API 僅供系統管理員使用，用於處理即將匯入 RAG 知識庫的訓練資料。

## 目錄

- [通用說明](#通用說明)
- [Upload API - 檔案上傳](#upload-api---檔案上傳)
- [Clean API - 清洗操作](#clean-api---清洗操作)
- [Task API - 任務管理](#task-api---任務管理)
- [Review API - 審核流程（Maker-Checker）](#review-api---審核流程maker-checker)
- [Knowledge Base API - 知識庫文件管理](#knowledge-base-api---知識庫文件管理)
- [Download API - 結果下載](#download-api---結果下載)
- [WebSocket API - 即時通知](#websocket-api---即時通知)
- [實體類型列表](#實體類型列表)
- [去識別化策略列表](#去識別化策略列表)
- [共用型別定義](#共用型別定義)

## 通用說明

### Base URL

```
http://localhost:3051/api/v1
```

### 回應格式

所有 REST API 回應均使用 `ApiResponse<T>` 包裝：

```json
{
  "success": true,
  "data": { ... },
  "message": "操作成功",
  "timestamp": "2025-01-15T10:30:00.000Z"
}
```

### 錯誤回應

```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "file_ids 不得為空"
  },
  "timestamp": "2025-01-15T10:30:00.000Z"
}
```

### 常見 HTTP 狀態碼

| 狀態碼 | 說明 |
|--------|------|
| 200 | 請求成功 |
| 201 | 資源建立成功 |
| 400 | 請求參數錯誤 |
| 404 | 資源不存在 |
| 409 | 資源狀態衝突，操作未執行 |
| 413 | 檔案大小超出限制 |
| 422 | 資料驗證失敗 |
| 503 | 相依服務不可用，無法安全完成操作 |
| 500 | 伺服器內部錯誤 |

---

## Upload API - 檔案上傳

### 上傳檔案

上傳一或多個待清洗檔案。支援 PDF、Word、Excel、PowerPoint、CSV、純文字、Markdown、JSON、HTML 與 ZIP 格式。

- **方法**: `POST`
- **路徑**: `/api/v1/upload`
- **Content-Type**: `multipart/form-data`
- **欄位名稱**: `files`
- **上傳檔大小限制**: 每個檔案不超過 50 MB，且同一批原始檔案合計不超過 50 MB（ZIP 檔本身亦同）
- **代理封裝限制**: NestJS 接收最多 51 MB multipart request，預留約 1 MB 給欄位名稱、檔名與 boundary；使用者檔案額度仍為 50 MB
- **ZIP 解壓後總量限制**: 依 RAG 服務 `max_zip_total_size_mb` 設定，預設 200 MB

Admin 上傳區會先檢查副檔名、單檔與整批 50 MB 上限；不符合時不送出 API，並在上傳區持續顯示中文原因。API 回傳的英文錯誤代碼或訊息會依白名單規則轉成中文，包括登入失效、權限不足、格式不支援、檔案過大，以及 ZIP 損毀、加密、解壓容量、壓縮比例、路徑安全與檔案數量限制。未知訊息一律改用該操作的中文預設說明，不直接顯示內部服務、位址或例外內容。

若上傳在更新登入狀態後仍回傳 401，Admin 會清除本機登入資料、切回登入畫面，並持續顯示「登入已失效，請重新登入」；不以重新載入頁面清除錯誤原因。

ZIP 可能以 HTTP 200 回傳部分處理結果：若 `files` 為空，Admin 停留在上傳步驟、保留已選檔案並顯示 `errors` 中文原因；若至少一個檔案成功，則進入分類步驟，並以「部分檔案未匯入」警告列出 `errors`。因此 `success: true` 只代表上傳請求已完成，不代表 ZIP 內每個項目都成功匯入。

#### 回應結構 - `FileUploadResponse[]`

| 欄位 | 型別 | 說明 |
|------|------|------|
| `id` | `UUID` | 檔案唯一識別碼 |
| `filename` | `string` | 儲存後的檔案名稱 |
| `size` | `number` | 檔案大小（bytes） |
| `type` | `string` | 檔案 MIME 類型 |
| `uploaded_at` | `ISO 8601` | 上傳時間 |

#### 回應範例

```json
{
  "success": true,
  "data": [
    {
      "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "filename": "a1b2c3d4_report.pdf",
      "size": 1048576,
      "type": "application/pdf",
      "uploaded_at": "2025-01-15T10:30:00.000Z"
    }
  ]
}
```

#### curl 範例

```bash
curl -X POST http://localhost:3051/api/v1/upload \
  -F "files=@/path/to/report.pdf" \
  -F "files=@/path/to/data.csv"
```

---

### 列出所有上傳檔案

預設列出系統中尚未邏輯刪除的上傳檔案，支援分頁、類型、審核狀態與檔名篩選。當 `pipeline_status=deleted` 時，資料集切換為只列出已邏輯刪除的檔案，不會混入有效資料。

- **方法**: `GET`
- **路徑**: `/api/v1/files`
- **權限**: `admin`, `data_cleaner`, `data_reviewer`

#### 查詢參數

| 參數 | 型別 | 預設值 | 說明 |
|------|------|--------|------|
| `page` | `integer` | 1 | 頁碼（最小 1） |
| `limit` | `integer` | 20 | 每頁筆數（1-100） |
| `file_type` | `string` | - | 篩選檔案類型（pdf/docx/xlsx/csv/txt/json/html/markdown） |
| `search` | `string` | - | 搜尋檔案名稱（模糊比對） |
| `pipeline_status` | `string` | - | 審核狀態：`unprocessed`、`processing`、`failed`、`pending_submission`、`pending_review`、`approved`、`ingested`、`rejected`、`deleted`；`deleted` 只查邏輯刪除資料，其他值以外的輸入回傳 HTTP 422 |

#### 回應結構 - `FileListResponse`

| 欄位 | 型別 | 說明 |
|------|------|------|
| `items` | `FileListItem[]` | 檔案清單 |
| `total` | `integer` | 符合條件的總筆數 |
| `page` | `integer` | 目前頁碼 |
| `limit` | `integer` | 每頁筆數 |

#### FileListItem 結構

| 欄位 | 型別 | 說明 |
|------|------|------|
| `id` | `UUID` | 檔案唯一識別碼 |
| `original_name` | `string` | 原始檔案名稱 |
| `file_type` | `string` | 檔案類型 |
| `file_size` | `integer` | 檔案大小（bytes） |
| `uploaded_at` | `ISO 8601` | 上傳時間 |
| `task_count` | `integer` | 關聯清洗任務數 |
| `pipeline_status` | `string` | 有效資料依最新關聯任務計算；邏輯刪除資料固定為 `deleted` |
| `latest_task_id` | `UUID \| null` | 最新關聯任務 ID |
| `latest_task_name` | `string \| null` | 與 `latest_task_id` 相同任務的名稱；舊任務可能為空值 |
| `can_delete` | `boolean` | 僅代表資料庫狀態符合刪除入口；執行前仍會檢查知識庫 |
| `delete_blocked_reason` | `string \| null` | 資料庫狀態不允許刪除時的中文原因 |

#### 審核狀態判定

來源列表以任務工作流呈現檔案目前位於哪個階段。若同一檔案關聯多筆任務，先依 `tasks.created_at DESC, tasks.id DESC` 選出唯一最新任務，再依下表判定；逐檔 `task_files.review_status` 只表示該檔案的審核結果，不會覆寫來源列表狀態。

| 條件 | `pipeline_status` | Cleaner 顯示 |
|------|-------------------|--------------|
| `files.deleted_at` 有值 | `deleted` | 已刪除 |
| 沒有關聯任務，或最新任務尚未開始／已取消 | `unprocessed` | 未處理 |
| 最新任務 `status=processing` | `processing` | 清洗中 |
| 最新任務 `status=failed` | `failed` | 清洗失敗 |
| 最新任務 `status=completed` 且 `approval_status=pending` | `pending_submission` | 未送審 |
| 最新任務 `approval_status=review_requested` | `pending_review` | 待審核 |
| 最新任務 `approval_status=rejected` | `rejected` | 已退回 |
| 最新任務 `approval_status=approved` | `approved` | 已批准 |
| 最新任務 `approval_status=ingested` | `ingested` | 已入庫 |

查詢依序執行有效／已刪除資料集選擇、檔案類型、檔名搜尋與審核狀態篩選，完成後才計算 `total` 並套用 `limit`／`offset`。結果固定以 `files.uploaded_at DESC, files.id DESC` 排序，確保相同上傳時間仍可穩定換頁；`total` 不會隨頁碼改變。

#### 回應範例

```json
{
  "items": [
    {
      "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "original_name": "report_2026.pdf",
      "file_type": "pdf",
      "file_size": 1048576,
      "uploaded_at": "2026-02-12T10:30:00",
      "task_count": 2,
      "pipeline_status": "pending_review",
      "latest_task_id": "b2c3d4e5-f6a7-8901-bcde-f12345678901",
      "latest_task_name": "ISO 27001 文件清洗審查",
      "can_delete": true,
      "delete_blocked_reason": null
    }
  ],
  "total": 42,
  "page": 1,
  "limit": 20
}
```

刪除資料的 `can_delete` 固定為 `false`，`delete_blocked_reason` 為「此來源資料已刪除」。若有關聯任務，`latest_task_id` 供 Cleaner 開啟任務歷史，`latest_task_name` 由同一筆任務取得；沒有關聯任務時兩者皆為 `null`。Cleaner 的任務名稱欄優先顯示名稱，舊任務名稱為空值或空白時顯示「任務 {ID 前 8 碼}」，沒有任務時顯示 `-`；操作欄不再重複顯示「未處理」，而以 `-` 搭配 tooltip 說明。

有效資料只有最新狀態為 `pending_submission`（未送審）或 `pending_review`（待審核）時才可能顯示刪除入口；若逐檔狀態已是 `rejected`，仍須先由任務退回流程處理。`can_delete=true` 只代表資料庫前置條件成立，刪除確認時仍會重新檢查全部關聯任務與 Qdrant。

API 時間維持 UTC 契約。Cleaner 顯示時以共用格式函式明確轉為 `Asia/Taipei`；舊資料若缺少 `Z` 或 offset，按 UTC 解讀，避免依賴使用者裝置時區。

---

### 分析待審核來源資料的刪除影響

- **方法**：`GET`
- **路徑**：`/api/v1/files/{file_id}/deletion-impact`
- **權限**：僅 `admin`
- **用途**：確認視窗開啟前，重新檢查所有關聯任務及 Qdrant 知識庫。

系統不是只看來源列表顯示的「待審核」。同一檔案可能關聯多筆任務，因此任一關聯任務仍在清洗、已批准或已入庫時，`can_delete` 都會是 `false`。Qdrant 無法連線時回傳 HTTP 503，不會假設資料尚未入庫。

| 欄位 | 型別 | 說明 |
|---|---|---|
| `can_delete` | `boolean` | 完整檢查後是否仍可刪除 |
| `blocked_reasons` | `string[]` | 中文阻擋原因 |
| `linked_tasks` | `SourceFileDeletionTaskImpact[]` | 受影響任務與刪除後有效檔案數 |
| `retained_original_files` | `integer` | 會保留的原始檔數 |
| `retained_cleaned_outputs` | `integer` | 會保留的清洗結果數 |
| `qdrant_chunks` | `integer` | 以 `file_id` 精確辨識的知識庫切塊數；必須為 0 才能刪除 |

### 邏輯刪除待審核來源資料

- **方法**：`DELETE`
- **路徑**：`/api/v1/files/{file_id}`
- **權限**：僅 `admin`
- **Content-Type**：`application/json`

```json
{
  "reason": "重複上傳"
}
```

NestJS 會以目前 JWT 使用者覆寫 `deleter_id`，不接受用戶端指定刪除者。`reason` 去除前後空白後須為 1～200 字。

刪除端點會鎖定 `File`、所有關聯 `Task` 與 `TaskFile`，再次執行影響分析；狀態在確認後若已改變，回傳 HTTP 409 且不寫入刪除標記。成功時在同一個 PostgreSQL 交易中完成以下操作：

1. 寫入既有 `files.deleted_at`、`deleted_by`、`deletion_reason`。
2. 保留原始檔、清洗結果、`TaskFile` 關聯與歷史欄位，不刪除實體檔案。
3. 排除已刪除檔案後，批次重算每筆關聯任務的檔案數、完成數與實體數。
4. 任務若沒有其他有效檔案，改為 `status=cancelled`、`approval_status=pending`；仍有有效檔案則維持原工作流狀態。
5. 寫入 `cleaning_audit_logs`，動作為 `source_file_soft_deleted`；NestJS 操作紀錄動作為 `SOURCE_FILE_DELETE`。

此功能使用既有軟刪除欄位，**不需要 Alembic 或 Prisma migration**。本次不提供還原介面，也不沿用法規版本的刪除／還原端點；已批准、已入庫資料及知識庫切塊都不在可刪除範圍。

---

### 取得檔案資訊

根據檔案 ID 查詢已上傳檔案的詳細資訊。

- **方法**: `GET`
- **路徑**: `/api/v1/upload/{file_id}`
- **路徑參數**: `file_id` (UUID) - 檔案唯一識別碼

#### 回應結構 - `FileInfo`

| 欄位 | 型別 | 說明 |
|------|------|------|
| `id` | `UUID` | 檔案唯一識別碼 |
| `filename` | `string` | 儲存後的檔案名稱 |
| `original_name` | `string` | 原始檔案名稱 |
| `size` | `number` | 檔案大小（bytes） |
| `type` | `string` | 檔案 MIME 類型 |
| `uploaded_at` | `ISO 8601` | 上傳時間 |

#### 回應範例

```json
{
  "success": true,
  "data": {
    "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "filename": "a1b2c3d4_report.pdf",
    "original_name": "Q1_財務報告.pdf",
    "size": 1048576,
    "type": "application/pdf",
    "uploaded_at": "2025-01-15T10:30:00.000Z"
  }
}
```

#### curl 範例

```bash
curl http://localhost:3051/api/v1/upload/a1b2c3d4-e5f6-7890-abcd-ef1234567890
```

---

### 更新法規／知識分類與附加資料

- **方法**：`PATCH`
- **路徑**：`/api/v1/upload/{fileId}/metadata`
- **權限**：`admin`
- **用途**：上傳後、開始清洗前，逐檔設定整批共用的法規／知識類型。

| 欄位 | 必填 | 說明 |
|---|---:|---|
| `regulation_type` | 清洗前必須有值 | `general`、`law`、`regulation`、`iso_standard`、`nist`、`cis`、`sop`、`case`；`other` 僅相容舊 API。 |
| `authority` | 否 | 發布機關。 |
| `source_url` | 否 | 官方來源網址。 |
| `version_date`、`effective_date` | 否 | 法規版本與施行日期。 |
| `status` | 否 | `current`、`deprecated`、`draft`。 |

此 PATCH 仍允許 partial update；真正的必要條件由 `POST /clean` 保證，因為上傳與分類是兩個連續步驟。清洗任務建立後不得再變更 `regulation_type`；若嘗試變更，回傳 HTTP 409：

```json
{
  "detail": {
    "code": "REGULATION_TYPE_LOCKED",
    "message": "Regulation/knowledge type cannot change after cleaning has started",
    "file_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
  }
}
```

PATCH 與建立清洗任務會鎖定同一筆檔案資料，確保併發時只會得到「先完成分類再建立任務」或「任務建立後拒絕改分類」兩種結果。

---

## Clean API - 清洗操作

### 啟動清洗任務

**前置條件**：`file_ids` 指向的每個檔案都必須已有 `regulation_type`。任一檔案未分類時，回傳 HTTP 400，且不得建立 Task、TaskFile 或提交背景佇列。

```json
{
  "detail": {
    "code": "REGULATION_TYPE_REQUIRED",
    "message": "Every file must have a regulation/knowledge type before cleaning",
    "file_ids": ["a1b2c3d4-e5f6-7890-abcd-ef1234567890"]
  }
}
```

提交檔案進行 PII（個人可識別資訊）偵測與去識別化處理。系統使用內建固定規則，無需指定。

- **方法**: `POST`
- **路徑**: `/api/v1/clean`
- **Content-Type**: `application/json`

#### 請求結構

| 欄位 | 型別 | 必填 | 說明 |
|------|------|------|------|
| `file_ids` | `UUID[]` | 是 | 待清洗檔案 ID 列表 |
| `task_name` | `string` | 是 | 任務名稱；去除前後空白後須為 1～100 字，允許重複 |

#### 請求範例

```json
{
  "file_ids": [
    "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "b2c3d4e5-f6a7-8901-bcde-f12345678901"
  ],
  "task_name": "114 年度資訊資產盤點表清洗"
}
```

#### 回應結構

| 欄位 | 型別 | 說明 |
|------|------|------|
| `task_id` | `UUID` | 任務唯一識別碼 |
| `status` | `string` | 任務狀態（`pending`） |
| `message` | `string` | 操作訊息 |

#### 回應範例

```json
{
  "success": true,
  "data": {
    "task_id": "c3d4e5f6-a7b8-9012-cdef-123456789012",
    "status": "pending",
    "message": "清洗任務已建立，共 2 個檔案排入處理佇列"
  }
}
```

#### curl 範例

```bash
curl -X POST http://localhost:3051/api/v1/clean \
  -H "Content-Type: application/json" \
  -d '{"file_ids": ["a1b2c3d4-e5f6-7890-abcd-ef1234567890"], "task_name": "114 年度資訊資產盤點表清洗"}'
```

---

### 預覽清洗結果

針對單一檔案取樣預覽清洗效果，不會建立正式任務。系統使用內建固定規則進行預覽，適合在正式執行前確認清洗結果。

- **方法**: `POST`
- **路徑**: `/api/v1/clean/preview`
- **Content-Type**: `application/json`

#### 請求結構

| 欄位 | 型別 | 必填 | 說明 |
|------|------|------|------|
| `file_id` | `UUID` | 是 | 預覽檔案 ID |
| `sample_size` | `int` | 否 | 取樣字元數（預設 1000） |

#### 請求範例

```json
{
  "file_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "sample_size": 500
}
```

#### 回應結構

| 欄位 | 型別 | 說明 |
|------|------|------|
| `original` | `string` | 原始文字取樣 |
| `anonymized` | `string` | 去識別化後文字 |
| `entities` | `DetectedEntity[]` | 偵測到的實體列表 |
| `stats` | `object` | 統計資訊 |

#### `DetectedEntity` 結構

| 欄位 | 型別 | 說明 |
|------|------|------|
| `entity_type` | `string` | 實體類型 |
| `text` | `string` | 原始文字片段 |
| `start` | `int` | 起始位置 |
| `end` | `int` | 結束位置 |
| `score` | `float` | 辨識信心分數（0-1） |

#### 回應範例

```json
{
  "success": true,
  "data": {
    "original": "客戶王小明（身分證 A123456789）於 2025-01-10 來電...",
    "anonymized": "客戶陳大華（身分證 **********）於 <DATE_TIME> 來電...",
    "entities": [
      {
        "entity_type": "PERSON",
        "text": "王小明",
        "start": 2,
        "end": 5,
        "score": 0.95
      },
      {
        "entity_type": "TW_ID",
        "text": "A123456789",
        "start": 11,
        "end": 21,
        "score": 0.99
      }
    ],
    "stats": {
      "total_entities": 3,
      "by_type": {
        "PERSON": 1,
        "TW_ID": 1,
        "DATE_TIME": 1
      }
    }
  }
}
```

#### curl 範例

```bash
curl -X POST http://localhost:3051/api/v1/clean/preview \
  -H "Content-Type: application/json" \
  -d '{
    "file_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "sample_size": 500
  }'
```

---

### 取得清洗結果詳情

取得已完成清洗任務的完整結果，包含各檔案的處理詳情與實體統計。

- **方法**: `GET`
- **路徑**: `/api/v1/clean/{task_id}/result`
- **路徑參數**: `task_id` (UUID) - 任務唯一識別碼

#### 回應結構 - `TaskResultResponse`

| 欄位 | 型別 | 說明 |
|------|------|------|
| `task_id` | `UUID` | 任務 ID |
| `task_name` | `string \| null` | 任務名稱；舊任務可能為 `null` |
| `status` | `string` | 任務狀態 |
| `total_files` | `int` | 檔案總數 |
| `completed_files` | `int` | 已完成檔案數 |
| `total_entities_found` | `int` | 偵測到的實體總數 |
| `files` | `FileResultDetail[]` | 各檔案處理詳情 |
| `created_at` | `ISO 8601` | 任務建立時間 |
| `completed_at` | `ISO 8601 \| null` | 任務完成時間 |

#### `FileResultDetail` 結構

| 欄位 | 型別 | 說明 |
|------|------|------|
| `file_id` | `UUID` | 檔案 ID |
| `filename` | `string` | 檔案名稱 |
| `status` | `string` | 處理狀態（`completed` / `failed`） |
| `entities_found` | `int` | 該檔案偵測到的實體數 |
| `entity_breakdown` | `object` | 依類型的實體數統計 |
| `error` | `string \| null` | 錯誤訊息（僅失敗時） |

#### 回應範例

```json
{
  "success": true,
  "data": {
    "task_id": "c3d4e5f6-a7b8-9012-cdef-123456789012",
    "status": "completed",
    "total_files": 2,
    "completed_files": 2,
    "total_entities_found": 47,
    "files": [
      {
        "file_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
        "filename": "Q1_財務報告.pdf",
        "status": "completed",
        "entities_found": 32,
        "entity_breakdown": {
          "PERSON": 12,
          "TW_ID": 5,
          "EMAIL_ADDRESS": 8,
          "PHONE_NUMBER": 7
        },
        "error": null
      },
      {
        "file_id": "b2c3d4e5-f6a7-8901-bcde-f12345678901",
        "filename": "客戶清單.xlsx",
        "status": "completed",
        "entities_found": 15,
        "entity_breakdown": {
          "PERSON": 10,
          "TW_PHONE": 5
        },
        "error": null
      }
    ],
    "created_at": "2025-01-15T10:30:00.000Z",
    "completed_at": "2025-01-15T10:32:15.000Z"
  }
}
```

#### curl 範例

```bash
curl http://localhost:3051/api/v1/clean/c3d4e5f6-a7b8-9012-cdef-123456789012/result
```

---

## Task API - 任務管理

### 列出所有任務

取得清洗任務列表，支援分頁。依建立時間倒序排列。

- **方法**: `GET`
- **路徑**: `/api/v1/tasks`
- **查詢參數**:

| 參數 | 型別 | 必填 | 預設 | 說明 |
|------|------|------|------|------|
| `limit` | `int` | 否 | 20 | 每頁筆數（最大 100） |
| `offset` | `int` | 否 | 0 | 跳過筆數 |

#### 回應結構

| 欄位 | 型別 | 說明 |
|------|------|------|
| `tasks` | `TaskResponse[]` | 任務列表 |
| `total` | `int` | 總任務數 |

#### `TaskResponse` 結構

| 欄位 | 型別 | 說明 |
|------|------|------|
| `task_id` | `UUID` | 任務 ID |
| `status` | `string` | 任務狀態（見下方說明） |
| `total_files` | `int` | 檔案總數 |
| `completed_files` | `int` | 已完成檔案數 |
| `progress` | `float` | 進度百分比（0-100） |
| `created_at` | `ISO 8601` | 建立時間，必須包含 UTC offset（`+00:00` 或 `Z`） |
| `updated_at` | `ISO 8601` | 最後更新時間 |

#### 任務狀態值

| 狀態 | 說明 |
|------|------|
| `pending` | 等待處理 |
| `processing` | 處理中 |
| `completed` | 已完成 |
| `failed` | 處理失敗 |
| `cancelled` | 已取消 |

#### 回應範例

```json
{
  "success": true,
  "data": {
    "tasks": [
      {
        "task_id": "c3d4e5f6-a7b8-9012-cdef-123456789012",
        "task_name": "114 年度資訊資產盤點表清洗",
        "status": "completed",
        "total_files": 2,
        "completed_files": 2,
        "progress": 100.0,
        "created_at": "2025-01-15T10:30:00.000Z",
        "updated_at": "2025-01-15T10:32:15.000Z"
      }
    ],
    "total": 1
  }
}
```

#### curl 範例

```bash
curl "http://localhost:3051/api/v1/tasks?limit=10&offset=0"
```

---

### 取得任務狀態

根據任務 ID 查詢單一任務的即時狀態。

- **方法**: `GET`
- **路徑**: `/api/v1/tasks/{task_id}`
- **路徑參數**: `task_id` (UUID) - 任務唯一識別碼

#### 回應結構 - `TaskResponse`

同[列出所有任務](#列出所有任務)中的 `TaskResponse` 結構。

#### 回應範例

```json
{
  "success": true,
  "data": {
    "task_id": "c3d4e5f6-a7b8-9012-cdef-123456789012",
    "status": "processing",
    "total_files": 3,
    "completed_files": 1,
    "progress": 33.3,
    "created_at": "2025-01-15T10:30:00.000Z",
    "updated_at": "2025-01-15T10:31:05.000Z"
  }
}
```

#### curl 範例

```bash
curl http://localhost:3051/api/v1/tasks/c3d4e5f6-a7b8-9012-cdef-123456789012
```

---

### 修改任務名稱

- **方法**: `PATCH`
- **路徑**: `/api/v1/tasks/{task_id}/name`
- **權限**: 僅 `admin`

```json
{
  "task_name": "114 年度資訊資產盤點表清洗"
}
```

名稱會先去除前後空白，結果必須為 1～100 字。所有任務狀態均可修改；最後成功請求為準。成功回傳最新 `TaskResponse`，並由 NestJS 寫入 `TASK_NAME_UPDATE` 稽核事件。

---

### 取消任務

取消尚未完成的清洗任務。僅 `pending` 或 `processing` 狀態的任務可被取消。

- **方法**: `DELETE`
- **路徑**: `/api/v1/tasks/{task_id}`
- **路徑參數**: `task_id` (UUID) - 任務唯一識別碼

#### 回應範例

```json
{
  "success": true,
  "data": {
    "task_id": "c3d4e5f6-a7b8-9012-cdef-123456789012",
    "status": "cancelled",
    "message": "任務已取消"
  }
}
```

#### curl 範例

```bash
curl -X DELETE http://localhost:3051/api/v1/tasks/c3d4e5f6-a7b8-9012-cdef-123456789012
```

---

## Review API - 審核流程（Maker-Checker）

本區段涵蓋資料去識別化審核流程的 API，實作 Maker-Checker 職責分離原則。

### 角色定義

| 角色 | 說明 | 可執行操作 |
|------|------|-----------|
| `data_cleaner` | 清洗人員 | 編輯內容、更新標籤、送審 |
| `data_reviewer` | 審查人員 | 審核檔案、批准/退回任務、送入 RAG |
| `admin` | 管理員 | 所有操作（但送審後不能自己批准） |

### 審批狀態流轉

```
approval_status:
  pending ──→ review_requested ──→ approved ──→ ingested
     ↑              │
     │              ▼
     └─── rejected (cleaner 可重新編輯後再送審)

  approved ──送入失敗／補償完成──→ approved (可重試)
```

任務與檔案使用不同層級的審核狀態，介面名稱不可混用：

| 資料層級 | 欄位 | 狀態 | 介面名稱 | 對編輯權限的影響 |
|---------|------|------|----------|------------------|
| 檔案 | `task_files.review_status` | `approved` | 通過 | 不單獨改變凍結狀態 |
| 檔案 | `task_files.review_status` | `rejected` | 未通過 | 不單獨解除凍結；可保留備註供後續修改 |
| 任務 | `tasks.approval_status` | `review_requested` | 待審核 | 全部有效檔案凍結 |
| 任務 | `tasks.approval_status` | `rejected` | 已退回 | 解除凍結，可修改後重新送審 |
| 任務 | `tasks.approval_status` | `approved` / `ingested` | 已批准／已送入 | 維持凍結，不可再編輯 |

逐檔「未通過」與「退回任務」分開處理，可讓多檔任務批准合格檔案並排除未通過檔案，同時避免審核期間內容被修改。

批准任務時，所有已完成且未刪除的檔案都必須完成逐檔審核，並且至少有一個檔案為「通過」。全部檔案皆為「未通過」時不得批准；通過與未通過混合時可批准，但送入 RAG 只處理通過檔案。

送入 RAG 採任務級全成或全退。任一檔案、Qdrant 或 PostgreSQL 提交失敗時，系統會 rollback 本次資料庫版本異動、以 Qdrant 伺服器端來源 filter 刪除本次全部 Qdrant/BM25 資料（不受 10,000 筆限制）、還原本次調整的舊法規 Qdrant 有效期限、記錄 `task_ingest_failed`，並維持 `approval_status='approved'`、`ingested_at=NULL` 供重新嘗試；全部通過檔案成功後才改為 `ingested`。若資料庫 commit 回應不明，系統會先重新讀取任務狀態；已確實提交為 `ingested` 時不會誤刪有效向量。

法規版本以 `effective_from` 排序，缺值時依序採 `effective_date`、`version_date`、目前日期。系統會依同一法規的完整 current／deprecated 歷程重建相鄰且不重疊的有效區間；只有生效日較新的版本會取代現行版，較舊或同日資料保留為歷史版。自動入庫與人工取代都同步 PostgreSQL 與 Qdrant 的 `effective_to`，失敗時反向還原；vector、hierarchical 與 hybrid BM25 檢索套用相同生效日、失效日、來源及標籤條件。

Cleaner 在送審、批准、退回或送入成功後，會使目前任務詳情、全部審批狀態的任務列表及相關分析快取失效。任務列表每次進入頁面及瀏覽器視窗重新取得焦點時也會重新查詢，避免顯示操作前的審批狀態。

### Maker-Checker 規則

- 送審者（`submitted_by`）不能是批准者（`approver_id`）
- 送審者不能是檔案審核者（`reviewer_id`）
- 所有操作者身份由伺服器 JWT 注入，不接受前端傳入

### 端點列表

| 端點 | 方法 | 角色 | 說明 |
|------|------|------|------|
| `/api/v1/review/tags` | GET | all | 取得標籤列表 |
| `/api/v1/review/{taskId}` | GET | all | 取得審核任務 |
| `/api/v1/review/{taskId}/files/{fileId}/content` | GET | all | 取得檔案內容 |
| `/api/v1/review/{taskId}/files/{fileId}/content` | PUT | cleaner, admin | 更新編輯內容 |
| `/api/v1/review/{taskId}/files/{fileId}/tags` | PUT | cleaner, admin | 更新標籤 |
| `/api/v1/review/{taskId}/files/{fileId}/quality-checks` | PUT | cleaner, reviewer, admin | 依任務階段更新品質檢查 |
| `/api/v1/review/{taskId}/submit` | POST | cleaner, admin | 送審 |
| `/api/v1/review/{taskId}/files/{fileId}/status` | PUT | reviewer, admin | 審核檔案 |
| `/api/v1/review/{taskId}/approve` | POST | reviewer, admin | 批准任務 |
| `/api/v1/review/{taskId}/reject` | POST | reviewer, admin | 退回任務 |
| `/api/v1/review/{taskId}/ingest` | POST | reviewer, admin | 送入 RAG |

### GET 取得審核任務

- **路徑**: `/api/v1/review/{taskId}`
- **角色**: `admin`、`data_cleaner`、`data_reviewer`

NestJS 代理會保留 FastAPI 的 `submitted_by`、`approved_by` 原始 UUID，並以一次批次查詢補上顯示人員摘要。摘要只包含 `id` 與 `name`，不回傳 Email；若舊資料對應的使用者已不存在，摘要為 `null`，Cleaner 以「未知使用者（ID 前 8 碼）」顯示並在 tooltip 保留完整 ID。

| 欄位 | 型別 | 說明 |
|------|------|------|
| `submitted_by` | `UUID \| null` | 原始送審者 ID，供稽核與 Maker-Checker 判斷 |
| `submitted_by_user` | `{ id: UUID, name: string \| null } \| null` | 送審者顯示摘要，不含 Email |
| `approved_by` | `UUID \| null` | 原始批准者 ID |
| `approved_by_user` | `{ id: UUID, name: string \| null } \| null` | 批准者顯示摘要，不含 Email |
| `files[].is_deleted` | `boolean` | 是否已由來源資料功能邏輯刪除；為 `true` 時只保留歷史顯示，不可再審核或入庫 |

Cleaner API client 會將上述 snake_case 欄位轉為 `submittedByUser`、`approvedByUser` 等 camelCase 欄位供 React 使用。

### PUT 更新人工編輯內容與重新偵測

- **路徑**: `/api/v1/review/{taskId}/files/{fileId}/content`
- **角色**: `admin`、`data_cleaner`
- **前置條件**: `approval_status in ('pending', 'rejected')`，且來源資料未刪除

#### 請求結構

```json
{
  "edited_content": "人工確認後的內容"
}
```

`edited_content` 不得為空字串。NestJS 會以目前 JWT 使用者覆寫 `editor_id`，不接受前端指定編輯者。

#### 成功回應

```json
{
  "status": "ok",
  "file_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "entity_count": 0,
  "total_entities_found": 4
}
```

#### 行為

1. 寫入資料庫前，以既有 `PIIDetector` 嚴格模式重新偵測完整 `edited_content`。
2. 鎖定任務並重新確認審批狀態，避免偵測期間任務已送審或批准。
3. 在同一交易更新 `edited_content`、`entities_json`、檔案 `entity_count` 與任務 `total_entities_found`。
4. 任務總數以所有未刪除關聯檔案的最新 `entity_count` 重新加總，不只套用前端顯示值。
5. 稽核紀錄 `file_content_edited` 會保存修改前後文字長度、修改前後偵測數及新的任務總數，不保存編輯內容。
6. 偵測器失敗時回傳 HTTP 503，內容與計數均不寫入；真正未偵測到敏感資料時則正常回傳 HTTP 200 與 `entity_count: 0`。

Cleaner 儲存成功後會重新查詢目前檔案與任務，並使任務列表及清洗分析快取失效，因此 PII 摘要、檔案「偵測敏感數」及任務「偵測敏感總數」會同步更新。

此功能沿用既有欄位，不需要 Alembic 或 Prisma migration，也不自動批次修改歷史任務；既有已編輯資料需再次儲存才會重新偵測。

### POST 送審任務

- **路徑**: `/api/v1/review/{taskId}/submit`
- **角色**: `admin`, `data_cleaner`
- **前置條件**: `task.status == 'completed'` 且 `approval_status in ('pending', 'rejected')`

#### 請求結構

| 欄位 | 型別 | 必填 | 說明 |
|------|------|------|------|
| `note` | `string` | 否 | 送審備註 |

#### 行為

- 設定 `approval_status = 'review_requested'`，記錄 `submitted_by` / `submitted_at`
- 若從 `rejected` 重新送審：重設所有 completed file 的 `review_status` 為 `pending`
- 清除先前批准資訊（`approved_by`, `approved_at`）

### POST 退回任務

- **路徑**: `/api/v1/review/{taskId}/reject`
- **角色**: `admin`, `data_reviewer`
- **前置條件**: `approval_status == 'review_requested'`
- **Maker-Checker**: `reviewer_id != task.submitted_by`（否則 403）

#### 請求結構

| 欄位 | 型別 | 必填 | 說明 |
|------|------|------|------|
| `note` | `string` | 是 | 退回原因（必填） |

#### 行為

- 設定 `approval_status = 'rejected'`
- 保留 file-level review notes（供 cleaner 查看退回原因）

逐檔標記「未通過」只會更新 `task_files.review_status`；任務仍維持 `review_requested`。只有呼叫本端點成功後，任務才會改為 `rejected`，並出現在 Cleaner 任務列表的「已退回」頁籤。

### 狀態凍結規則

送審後（`review_requested`、`approved`、`ingested`）禁止以下操作：
- 更新檔案編輯內容（PUT content）
- 更新檔案標籤（PUT tags）

品質檢查依角色與任務階段控制：

| 角色 | 可修改條件 |
|------|--------------|
| `data_cleaner` | 任務為 `pending` 或 `rejected` |
| `data_reviewer` | 任務為 `review_requested` 且檔案仍為 `pending` |
| `admin` | 上述兩種階段 |

`approved` 與 `ingested` 任務的品質檢查一律唯讀。NestJS 從 JWT 注入操作者角色，前端不能自行指定。

`task_files.review_status = 'rejected'` 不會自行解除凍結。審核人員必須呼叫「退回任務」端點，將 `tasks.approval_status` 改為 `rejected`，清洗人員才可重新編輯。

---

## Knowledge Base API - 知識庫文件管理

### 列出分組文件

- **方法**: `GET`
- **路徑**: `/api/v1/knowledge-base/documents/grouped`
- **權限**: `admin`、`data_cleaner`、`data_reviewer`

每個來源文件回傳一筆資料。系統先組合來源類型、標籤、關鍵字篩選，再套用排序，最後才分頁；因此 `total` 是所有條件套用後的文件數。

#### 查詢參數

| 參數 | 型別 | 預設值 | 說明 |
|------|------|--------|------|
| `page` | `integer` | `1` | 頁碼，最小值 1 |
| `limit` | `integer` | `20` | 每頁筆數，1～100 |
| `source_type` | `string` | - | 來源類型：`cleaning`、`gdrive`、`docs` |
| `tag` | `string` | - | 標籤完全比對 |
| `keyword` | `string` | - | 不分大小寫搜尋文件名、完整來源或任一標籤；最長 200 字 |
| `sort_by` | `string` | `indexed_at` | `display_name`、`indexed_at`、`chunk_count` |
| `sort_order` | `string` | `desc` | `asc` 或 `desc` |

#### 回應項目

| 欄位 | 型別 | 說明 |
|------|------|------|
| `source` | `string` | Qdrant 完整來源識別字串 |
| `display_name` | `string` | 使用者可辨識的文件名 |
| `source_type` | `string` | 來源類型 |
| `chunk_count` | `integer` | 文件切塊數 |
| `tags` | `string[]` | 文件標籤 |
| `indexed_at` | `ISO 8601 \| null` | 知識庫匯入時間（UTC）；無法可靠回推的舊文件為 `null` |

`indexed_at` 的來源優先序如下：

1. 清洗匯入：`tasks.ingested_at`。
2. Google Drive：`gdrive_sync_files.last_synced_at`；同步服務新增與更新時皆明確寫入 UTC，不依賴 PostgreSQL session timezone。
3. 其他新匯入文件：Qdrant chunk payload 的 `indexed_at`。
4. 舊資料沒有上述時間：回傳 `null`，Cleaner 顯示「—」，不以檔名或目前時間推測。

日期排序時，無日期文件固定排在有日期文件之後；同值以文件名維持穩定排序。Cleaner 顯示時固定轉為 `Asia/Taipei`（UTC+8）。

---

## Download API - 結果下載

### 下載所有清洗結果（ZIP）

將任務中所有已清洗檔案打包為 ZIP 下載。

- **方法**: `GET`
- **路徑**: `/api/v1/download/{task_id}`
- **權限**: `admin`、`data_cleaner`，必須帶入 Bearer Token
- **路徑參數**: `task_id` (UUID) - 任務 ID
- **回應格式**: `application/zip`
- **Content-Disposition**: `attachment; filename="task_{task_id}.zip"`

Admin 的「下載清洗結果」會用目前登入者的 Bearer Token 呼叫 API，收到 Blob 後才觸發瀏覽器下載；不會導向 Cleaner。此檔案是清洗完成的工作產物，不代表已通過 Maker-Checker 審核；「前往審核」為另一個獨立操作。

#### curl 範例

```bash
curl -o cleaned_result.zip \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  http://localhost:3051/api/v1/download/c3d4e5f6-a7b8-9012-cdef-123456789012
```

---

### 下載單一清洗檔案

下載任務中特定檔案的清洗結果。

- **方法**: `GET`
- **路徑**: `/api/v1/download/{task_id}/{file_id}`
- **權限**: `admin`、`data_cleaner`，必須帶入 Bearer Token
- **路徑參數**:
  - `task_id` (UUID) - 任務 ID
  - `file_id` (UUID) - 檔案 ID
- **回應格式**: 依原始檔案類型而定

#### curl 範例

```bash
curl -o cleaned_report.pdf \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  http://localhost:3051/api/v1/download/c3d4e5f6-a7b8-9012-cdef-123456789012/a1b2c3d4-e5f6-7890-abcd-ef1234567890
```

---

### 下載清洗報告

下載任務的完整 JSON 清洗報告，包含所有偵測到的實體與統計資料。

- **方法**: `GET`
- **路徑**: `/api/v1/download/{task_id}/report`
- **權限**: `admin`、`data_cleaner`，必須帶入 Bearer Token
- **路徑參數**: `task_id` (UUID) - 任務 ID
- **回應格式**: `application/json`
- **Content-Disposition**: `attachment; filename="report_{task_id}.json"`

#### 報告 JSON 結構

```json
{
  "task_id": "c3d4e5f6-a7b8-9012-cdef-123456789012",
  "generated_at": "2025-01-15T12:00:00.000Z",
  "summary": {
    "total_files": 2,
    "total_entities": 47,
    "processing_time_ms": 13500
  },
  "files": [
    {
      "file_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "filename": "Q1_財務報告.pdf",
      "entities": [
        {
          "entity_type": "PERSON",
          "original": "王小明",
          "replacement": "陳大華",
          "strategy": "pseudonymize",
          "position": { "start": 15, "end": 18 },
          "score": 0.95
        }
      ]
    }
  ]
}
```

#### curl 範例

```bash
curl -o report.json \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  http://localhost:3051/api/v1/download/c3d4e5f6-a7b8-9012-cdef-123456789012/report
```

---

## WebSocket API - 即時通知

透過 WebSocket 即時接收任務狀態變更與處理進度，免去輪詢的開銷。

### 連線方式

#### 訂閱特定任務

```
ws://localhost:3051/ws/tasks/{task_id}
```

僅接收指定任務的狀態更新。

#### 訂閱所有任務

```
ws://localhost:3051/ws/all
```

接收所有進行中任務的狀態更新。

### 訊息格式

所有 WebSocket 訊息均為 JSON 格式：

```json
{
  "type": "task_update",
  "task_id": "c3d4e5f6-a7b8-9012-cdef-123456789012",
  "status": "processing",
  "progress": 66.7,
  "error": null
}
```

#### 訊息欄位

| 欄位 | 型別 | 說明 |
|------|------|------|
| `type` | `string` | 訊息類型，固定為 `task_update` |
| `task_id` | `string` | 任務 ID |
| `status` | `string \| undefined` | 任務狀態（狀態變更時才帶入） |
| `progress` | `float \| undefined` | 進度百分比（進度更新時帶入） |
| `error` | `string \| undefined` | 錯誤訊息（失敗時帶入） |

### 訊息範例

#### 任務開始處理

```json
{
  "type": "task_update",
  "task_id": "c3d4e5f6-a7b8-9012-cdef-123456789012",
  "status": "processing",
  "progress": 0
}
```

#### 進度更新

```json
{
  "type": "task_update",
  "task_id": "c3d4e5f6-a7b8-9012-cdef-123456789012",
  "progress": 66.7
}
```

#### 任務完成

```json
{
  "type": "task_update",
  "task_id": "c3d4e5f6-a7b8-9012-cdef-123456789012",
  "status": "completed",
  "progress": 100
}
```

#### 任務失敗

```json
{
  "type": "task_update",
  "task_id": "c3d4e5f6-a7b8-9012-cdef-123456789012",
  "status": "failed",
  "error": "檔案解析失敗：不支援的格式"
}
```

### JavaScript 連線範例

```javascript
const ws = new WebSocket('ws://localhost:3051/ws/tasks/c3d4e5f6-a7b8-9012-cdef-123456789012');

ws.onmessage = (event) => {
  const data = JSON.parse(event.data);
  console.log(`Task ${data.task_id}: ${data.status ?? ''} ${data.progress ?? ''}%`);

  if (data.status === 'completed') {
    ws.close();
  }
};

ws.onerror = (error) => {
  console.error('WebSocket error:', error);
};
```

---

## 實體類型列表

系統支援以下 PII 實體類型的自動偵測：

| 實體類型 | 說明 | 範例 |
|----------|------|------|
| `PERSON` | 人名 | 王小明、John Smith |
| `TW_ID` | 台灣身分證字號 | A123456789 |
| `TW_PHONE` | 台灣手機號碼 | 0912-345-678 |
| `TW_UNIFIED_BUSINESS_NO` | 台灣統一編號 | 12345678 |
| `TW_ADDRESS` | 台灣地址 | 台北市中正區忠孝東路一段100號 |
| `EMAIL_ADDRESS` | 電子郵件地址 | user@example.com |
| `PHONE_NUMBER` | 一般電話號碼 | +886-2-1234-5678 |
| `CREDIT_CARD` | 信用卡號碼 | 4111-1111-1111-1111 |
| `IBAN_CODE` | 國際銀行帳號 | DE89370400440532013000 |
| `IP_ADDRESS` | IP 位址 | 192.168.1.1 |
| `DATE_TIME` | 日期與時間 | 2025-01-15、民國114年 |
| `LOCATION` | 地址或地點 | 台北市信義區信義路五段 |
| `ORGANIZATION` | 組織或公司名稱 | 台灣積體電路製造股份有限公司 |
| `API_KEY` | API 金鑰 | sk-abc123def456... |
| `ACCESS_TOKEN` | 存取權杖 | Bearer eyJhbGciOiJ... |
| `PRIVATE_KEY` | 私密金鑰 | -----BEGIN RSA PRIVATE KEY----- |
| `AWS_ACCESS_KEY` | AWS 存取金鑰 | AKIAIOSFODNN7EXAMPLE |
| `AZURE_KEY` | Azure 金鑰 | DefaultEndpointsProtocol=https... |
| `GCP_KEY` | GCP 金鑰 | AIzaSyA1234567890... |
| `URL` | 網址 | https://example.com/path |

---

## 去識別化策略列表

各實體類型可搭配以下去識別化策略使用：

| 策略 | 說明 | 處理方式範例 |
|------|------|-------------|
| `mask` | 完全遮罩 | `王小明` → `***`、`A123456789` → `**********` |
| `partial_mask` | 部分遮罩 | `0912345678` → `0912******`、`A123456789` → `A*******89` |
| `pseudonymize` | 假名替換 | `王小明` → `[Person_1]`（使用一致性對映） |
| `generalize` | 泛化處理 | `台北市信義區` → `台北市`、`25歲` → `20-30歲` |
| `keep_labeled` | 保留並標注 | `王小明` → `[PERSON: 王小明]` |
| `encrypt` | 加密處理 | `A123456789` → `enc:a3f8b2c1...`（可逆） |

### 策略參數

#### `mask` 策略參數

| 參數 | 型別 | 預設 | 說明 |
|------|------|------|------|
| `mask_char` | `string` | `*` | 遮罩字元 |

#### `partial_mask` 策略參數

| 參數 | 型別 | 預設 | 說明 |
|------|------|------|------|
| `keep_first` | `int` | `1` | 保留前幾個字元 |
| `keep_last` | `int` | `1` | 保留後幾個字元 |
| `mask_char` | `string` | `*` | 遮罩字元 |
| `mode` | `string` | `default` | 特殊模式（見下方說明） |

##### `mode` 參數說明

| mode 值 | 行為 | 範例 |
|---------|------|------|
| `default` | 保留前 N 後 M 字元，中間遮罩 | `A123456789` → `A*******89` |
| `chinese_name` | 保留姓氏（第一字），其餘以 ○ 替換 | `王大明` → `王○○` |
| `ip_octet` | 以 `.` 分割 IP，遮蔽第 3 段 | `10.10.200.10` → `10.10.X.10` |
| `tw_address` | 保留縣市區，遮蔽路段門牌 | `台北市中正區忠孝東路100號` → `台北市中正區***` |

#### `generalize` 策略參數

| 參數 | 型別 | 預設 | 說明 |
|------|------|------|------|
| `level` | `int` | `1` | 泛化層級（數值越高越模糊） |

#### `encrypt` 策略參數

| 參數 | 型別 | 預設 | 說明 |
|------|------|------|------|
| `algorithm` | `string` | `aes-256-gcm` | 加密演算法 |

---

## 共用型別定義

以下型別定義位於 `packages/shared-types/` 中，前後端共用：

```typescript
// 通用 API 回應包裝
interface ApiResponse<T> {
  success: boolean;
  data?: T;
  error?: {
    code: string;
    message: string;
  };
  message?: string;
  timestamp: string;
}

// 檔案上傳回應
interface FileUploadResponse {
  id: string;
  filename: string;
  size: number;
  type: string;
  uploaded_at: string;
}

// 檔案資訊
interface FileInfo {
  id: string;
  filename: string;
  original_name: string;
  size: number;
  type: string;
  uploaded_at: string;
}

// 實體規則
interface EntityRuleSchema {
  entity_type: string;
  strategy: string;
  params?: Record<string, unknown>;
}

// 任務回應
interface TaskResponse {
  task_id: string;
  task_name: string | null;
  status: 'pending' | 'processing' | 'completed' | 'failed' | 'cancelled';
  total_files: number;
  completed_files: number;
  progress: number;
  created_at: string;
  updated_at: string;
}

// WebSocket 訊息
interface TaskUpdateMessage {
  type: 'task_update';
  task_id: string;
  status?: string;
  progress?: number;
  error?: string;
}
```
