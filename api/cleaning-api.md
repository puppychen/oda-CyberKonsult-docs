# 資料去識別化系統 API 文件

> 本文件涵蓋 RAG 資料去識別化子系統的完整 API 規格，包含檔案上傳、去識別化任務、規則管理、結果下載及 WebSocket 即時通知。
>
> **使用範圍**：本 API 僅供系統管理員使用，用於處理即將匯入 RAG 知識庫的訓練資料。

## 目錄

- [通用說明](#通用說明)
- [Upload API - 檔案上傳](#upload-api---檔案上傳)
- [Clean API - 清洗操作](#clean-api---清洗操作)
- [Task API - 任務管理](#task-api---任務管理)
- [Rule API - 規則設定](#rule-api---規則設定)
- [Download API - 結果下載](#download-api---結果下載)
- [WebSocket API - 即時通知](#websocket-api---即時通知)
- [實體類型列表](#實體類型列表)
- [去識別化策略列表](#去識別化策略列表)
- [共用型別定義](#共用型別定義)

---

## 通用說明

### Base URL

```
http://localhost:4000/api/v1
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
| 413 | 檔案大小超出限制 |
| 422 | 資料驗證失敗 |
| 500 | 伺服器內部錯誤 |

---

## Upload API - 檔案上傳

### 上傳檔案

上傳一或多個待清洗檔案。支援 PDF、DOCX、XLSX、CSV、TXT 格式。

- **方法**: `POST`
- **路徑**: `/api/v1/upload`
- **Content-Type**: `multipart/form-data`
- **欄位名稱**: `files`
- **檔案大小限制**: 單檔 50MB

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
curl -X POST http://localhost:4000/api/v1/upload \
  -F "files=@/path/to/report.pdf" \
  -F "files=@/path/to/data.csv"
```

---

### 列出所有上傳檔案

列出系統中所有上傳的檔案，支援分頁、類型篩選與檔名搜尋。

- **方法**: `GET`
- **路徑**: `/api/v1/files`
- **權限**: `admin`, `data_cleaner`

#### 查詢參數

| 參數 | 型別 | 預設值 | 說明 |
|------|------|--------|------|
| `page` | `integer` | 1 | 頁碼（最小 1） |
| `limit` | `integer` | 20 | 每頁筆數（1-100） |
| `file_type` | `string` | - | 篩選檔案類型（pdf/docx/xlsx/csv/txt/json/html/markdown） |
| `search` | `string` | - | 搜尋檔案名稱（模糊比對） |

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
      "task_count": 2
    }
  ],
  "total": 42,
  "page": 1,
  "limit": 20
}
```

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
curl http://localhost:4000/api/v1/upload/a1b2c3d4-e5f6-7890-abcd-ef1234567890
```

---

## Clean API - 清洗操作

### 啟動清洗任務

提交檔案進行 PII（個人可識別資訊）偵測與去識別化處理。可指定規則設定檔或自訂規則。

- **方法**: `POST`
- **路徑**: `/api/v1/clean`
- **Content-Type**: `application/json`

#### 請求結構

| 欄位 | 型別 | 必填 | 說明 |
|------|------|------|------|
| `file_ids` | `UUID[]` | 是 | 待清洗檔案 ID 列表 |
| `profile_id` | `UUID` | 否 | 規則設定檔 ID（與 rules 擇一使用） |
| `rules` | `EntityRuleSchema[]` | 否 | 自訂去識別化規則列表（與 profile_id 擇一使用） |

#### `EntityRuleSchema` 結構

| 欄位 | 型別 | 必填 | 說明 |
|------|------|------|------|
| `entity_type` | `string` | 是 | 實體類型，參見[實體類型列表](#實體類型列表) |
| `strategy` | `string` | 是 | 去識別化策略，參見[去識別化策略列表](#去識別化策略列表) |
| `params` | `object` | 否 | 策略額外參數 |

#### 請求範例

```json
{
  "file_ids": [
    "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "b2c3d4e5-f6a7-8901-bcde-f12345678901"
  ],
  "rules": [
    {
      "entity_type": "PERSON",
      "strategy": "pseudonymize",
      "params": {}
    },
    {
      "entity_type": "TW_ID",
      "strategy": "mask",
      "params": { "mask_char": "*" }
    },
    {
      "entity_type": "EMAIL_ADDRESS",
      "strategy": "partial_mask",
      "params": { "visible_chars": 3 }
    }
  ]
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
curl -X POST http://localhost:4000/api/v1/clean \
  -H "Content-Type: application/json" \
  -d '{
    "file_ids": ["a1b2c3d4-e5f6-7890-abcd-ef1234567890"],
    "rules": [
      { "entity_type": "PERSON", "strategy": "pseudonymize" },
      { "entity_type": "TW_ID", "strategy": "mask" }
    ]
  }'
```

---

### 預覽清洗結果

針對單一檔案取樣預覽清洗效果，不會建立正式任務。適合在正式執行前確認規則是否符合預期。

- **方法**: `POST`
- **路徑**: `/api/v1/clean/preview`
- **Content-Type**: `application/json`

#### 請求結構

| 欄位 | 型別 | 必填 | 說明 |
|------|------|------|------|
| `file_id` | `UUID` | 是 | 預覽檔案 ID |
| `rules` | `EntityRuleSchema[]` | 否 | 自訂去識別化規則（未指定則使用預設） |
| `sample_size` | `int` | 否 | 取樣字元數（預設 1000） |

#### 請求範例

```json
{
  "file_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "rules": [
    { "entity_type": "PERSON", "strategy": "pseudonymize" },
    { "entity_type": "EMAIL_ADDRESS", "strategy": "mask" }
  ],
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
curl -X POST http://localhost:4000/api/v1/clean/preview \
  -H "Content-Type: application/json" \
  -d '{
    "file_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "rules": [
      { "entity_type": "PERSON", "strategy": "pseudonymize" }
    ],
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
curl http://localhost:4000/api/v1/clean/c3d4e5f6-a7b8-9012-cdef-123456789012/result
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
| `created_at` | `ISO 8601` | 建立時間 |
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
curl "http://localhost:4000/api/v1/tasks?limit=10&offset=0"
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
curl http://localhost:4000/api/v1/tasks/c3d4e5f6-a7b8-9012-cdef-123456789012
```

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
curl -X DELETE http://localhost:4000/api/v1/tasks/c3d4e5f6-a7b8-9012-cdef-123456789012
```

---

## Rule API - 規則設定

規則設定檔（Rule Profile）用於保存常用的去識別化規則組合，方便重複使用。

### 列出所有規則設定檔

- **方法**: `GET`
- **路徑**: `/api/v1/rules`

#### 回應結構

| 欄位 | 型別 | 說明 |
|------|------|------|
| `profiles` | `RuleProfile[]` | 規則設定檔列表 |

#### `RuleProfile` 結構

| 欄位 | 型別 | 說明 |
|------|------|------|
| `id` | `UUID` | 設定檔 ID |
| `name` | `string` | 設定檔名稱 |
| `description` | `string` | 描述說明 |
| `rules` | `EntityRuleSchema[]` | 去識別化規則列表 |
| `is_default` | `boolean` | 是否為預設設定檔 |
| `created_at` | `ISO 8601` | 建立時間 |
| `updated_at` | `ISO 8601` | 最後更新時間 |

#### 回應範例

```json
{
  "success": true,
  "data": {
    "profiles": [
      {
        "id": "d4e5f6a7-b8c9-0123-defa-234567890123",
        "name": "標準台灣 PII 規則",
        "description": "適用於台灣地區的標準個資去識別化規則",
        "rules": [
          { "entity_type": "PERSON", "strategy": "pseudonymize", "params": {} },
          { "entity_type": "TW_ID", "strategy": "mask", "params": { "mask_char": "*" } },
          { "entity_type": "TW_PHONE", "strategy": "partial_mask", "params": { "visible_chars": 4 } }
        ],
        "is_default": true,
        "created_at": "2025-01-10T08:00:00.000Z",
        "updated_at": "2025-01-10T08:00:00.000Z"
      }
    ]
  }
}
```

#### curl 範例

```bash
curl http://localhost:4000/api/v1/rules
```

---

### 取得單一規則設定檔

- **方法**: `GET`
- **路徑**: `/api/v1/rules/{profile_id}`
- **路徑參數**: `profile_id` (UUID) - 設定檔 ID

#### 回應範例

```json
{
  "success": true,
  "data": {
    "id": "d4e5f6a7-b8c9-0123-defa-234567890123",
    "name": "標準台灣 PII 規則",
    "description": "適用於台灣地區的標準個資去識別化規則",
    "rules": [
      { "entity_type": "PERSON", "strategy": "pseudonymize", "params": {} },
      { "entity_type": "TW_ID", "strategy": "mask", "params": { "mask_char": "*" } }
    ],
    "is_default": true,
    "created_at": "2025-01-10T08:00:00.000Z",
    "updated_at": "2025-01-10T08:00:00.000Z"
  }
}
```

#### curl 範例

```bash
curl http://localhost:4000/api/v1/rules/d4e5f6a7-b8c9-0123-defa-234567890123
```

---

### 建立規則設定檔

- **方法**: `POST`
- **路徑**: `/api/v1/rules`
- **Content-Type**: `application/json`

#### 請求結構

| 欄位 | 型別 | 必填 | 說明 |
|------|------|------|------|
| `name` | `string` | 是 | 設定檔名稱 |
| `description` | `string` | 否 | 描述說明 |
| `rules` | `EntityRuleSchema[]` | 是 | 去識別化規則列表 |
| `is_default` | `boolean` | 否 | 是否設為預設（預設 false） |

#### 請求範例

```json
{
  "name": "金融業嚴格規則",
  "description": "適用於金融業的嚴格個資去識別化規則",
  "rules": [
    { "entity_type": "PERSON", "strategy": "pseudonymize" },
    { "entity_type": "TW_ID", "strategy": "mask" },
    { "entity_type": "CREDIT_CARD", "strategy": "mask" },
    { "entity_type": "EMAIL_ADDRESS", "strategy": "mask" },
    { "entity_type": "PHONE_NUMBER", "strategy": "mask" }
  ],
  "is_default": false
}
```

#### 回應範例

```json
{
  "success": true,
  "data": {
    "id": "e5f6a7b8-c9d0-1234-efab-345678901234",
    "name": "金融業嚴格規則",
    "description": "適用於金融業的嚴格個資去識別化規則",
    "rules": [ ... ],
    "is_default": false,
    "created_at": "2025-01-15T11:00:00.000Z",
    "updated_at": "2025-01-15T11:00:00.000Z"
  }
}
```

#### curl 範例

```bash
curl -X POST http://localhost:4000/api/v1/rules \
  -H "Content-Type: application/json" \
  -d '{
    "name": "金融業嚴格規則",
    "description": "適用於金融業的嚴格個資去識別化規則",
    "rules": [
      { "entity_type": "PERSON", "strategy": "pseudonymize" },
      { "entity_type": "TW_ID", "strategy": "mask" },
      { "entity_type": "CREDIT_CARD", "strategy": "mask" }
    ]
  }'
```

---

### 更新規則設定檔

- **方法**: `PUT`
- **路徑**: `/api/v1/rules/{profile_id}`
- **路徑參數**: `profile_id` (UUID) - 設定檔 ID
- **Content-Type**: `application/json`

#### 請求結構

同建立規則設定檔，所有欄位皆為可選（僅更新傳入的欄位）。

#### 請求範例

```json
{
  "name": "金融業嚴格規則 v2",
  "rules": [
    { "entity_type": "PERSON", "strategy": "pseudonymize" },
    { "entity_type": "TW_ID", "strategy": "encrypt" },
    { "entity_type": "CREDIT_CARD", "strategy": "mask" },
    { "entity_type": "IP_ADDRESS", "strategy": "generalize" }
  ]
}
```

#### curl 範例

```bash
curl -X PUT http://localhost:4000/api/v1/rules/e5f6a7b8-c9d0-1234-efab-345678901234 \
  -H "Content-Type: application/json" \
  -d '{
    "name": "金融業嚴格規則 v2",
    "rules": [
      { "entity_type": "PERSON", "strategy": "pseudonymize" },
      { "entity_type": "TW_ID", "strategy": "encrypt" }
    ]
  }'
```

---

### 刪除規則設定檔

- **方法**: `DELETE`
- **路徑**: `/api/v1/rules/{profile_id}`
- **路徑參數**: `profile_id` (UUID) - 設定檔 ID

#### 回應範例

```json
{
  "success": true,
  "data": {
    "message": "規則設定檔已刪除"
  }
}
```

#### curl 範例

```bash
curl -X DELETE http://localhost:4000/api/v1/rules/e5f6a7b8-c9d0-1234-efab-345678901234
```

---

### 匯出規則設定檔

將規則設定檔匯出為 JSON 檔案，便於備份或跨環境移轉。

- **方法**: `GET`
- **路徑**: `/api/v1/rules/{profile_id}/export`
- **路徑參數**: `profile_id` (UUID) - 設定檔 ID
- **回應格式**: `application/json`（直接下載）

#### 匯出 JSON 結構

```json
{
  "version": "1.0",
  "exported_at": "2025-01-15T12:00:00.000Z",
  "profile": {
    "name": "標準台灣 PII 規則",
    "description": "適用於台灣地區的標準個資去識別化規則",
    "rules": [
      { "entity_type": "PERSON", "strategy": "pseudonymize", "params": {} },
      { "entity_type": "TW_ID", "strategy": "mask", "params": { "mask_char": "*" } }
    ]
  }
}
```

#### curl 範例

```bash
curl -o rule_profile.json \
  http://localhost:4000/api/v1/rules/d4e5f6a7-b8c9-0123-defa-234567890123/export
```

---

### 匯入規則設定檔

從 JSON 檔案匯入規則設定檔。

- **方法**: `POST`
- **路徑**: `/api/v1/rules/import`
- **Content-Type**: `application/json`

#### 請求結構

同匯出的 JSON 結構。

#### 回應範例

```json
{
  "success": true,
  "data": {
    "id": "f6a7b8c9-d0e1-2345-fabc-456789012345",
    "name": "標準台灣 PII 規則",
    "message": "規則設定檔匯入成功"
  }
}
```

#### curl 範例

```bash
curl -X POST http://localhost:4000/api/v1/rules/import \
  -H "Content-Type: application/json" \
  -d @rule_profile.json
```

---

## Download API - 結果下載

### 下載所有清洗結果（ZIP）

將任務中所有已清洗檔案打包為 ZIP 下載。

- **方法**: `GET`
- **路徑**: `/api/v1/download/{task_id}`
- **路徑參數**: `task_id` (UUID) - 任務 ID
- **回應格式**: `application/zip`
- **Content-Disposition**: `attachment; filename="cleaned_{task_id}.zip"`

#### curl 範例

```bash
curl -o cleaned_result.zip \
  http://localhost:4000/api/v1/download/c3d4e5f6-a7b8-9012-cdef-123456789012
```

---

### 下載單一清洗檔案

下載任務中特定檔案的清洗結果。

- **方法**: `GET`
- **路徑**: `/api/v1/download/{task_id}/{file_id}`
- **路徑參數**:
  - `task_id` (UUID) - 任務 ID
  - `file_id` (UUID) - 檔案 ID
- **回應格式**: 依原始檔案類型而定

#### curl 範例

```bash
curl -o cleaned_report.pdf \
  http://localhost:4000/api/v1/download/c3d4e5f6-a7b8-9012-cdef-123456789012/a1b2c3d4-e5f6-7890-abcd-ef1234567890
```

---

### 下載清洗報告

下載任務的完整 JSON 清洗報告，包含所有偵測到的實體與統計資料。

- **方法**: `GET`
- **路徑**: `/api/v1/download/{task_id}/report`
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
  http://localhost:4000/api/v1/download/c3d4e5f6-a7b8-9012-cdef-123456789012/report
```

---

## WebSocket API - 即時通知

透過 WebSocket 即時接收任務狀態變更與處理進度，免去輪詢的開銷。

### 連線方式

#### 訂閱特定任務

```
ws://localhost:4000/ws/tasks/{task_id}
```

僅接收指定任務的狀態更新。

#### 訂閱所有任務

```
ws://localhost:4000/ws/all
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
const ws = new WebSocket('ws://localhost:4000/ws/tasks/c3d4e5f6-a7b8-9012-cdef-123456789012');

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
