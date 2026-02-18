# Google Drive 資料來源整合 API 文件

> 本文件涵蓋 Google Drive 資料來源整合的完整 API 規格，包含連線設定、同步觸發、同步歷程查詢及已同步檔案管理。
>
> **使用範圍**：本 API 僅供系統管理員使用（需 JWT Bearer Token 驗證），用於將 Google Drive 資料夾中的文件自動匯入 RAG 知識庫。

## 目錄

- [通用說明](#通用說明)
- [Config API - 連線設定](#config-api---連線設定)
- [Sync API - 同步操作](#sync-api---同步操作)
- [Status API - 同步狀態](#status-api---同步狀態)
- [History API - 同步歷程](#history-api---同步歷程)
- [Files API - 已同步檔案](#files-api---已同步檔案)
- [錯誤碼一覽](#錯誤碼一覽)
- [共用型別定義](#共用型別定義)

---

## 通用說明

### Base URL

本系統採雙層架構，NestJS 作為統一入口代理至 FastAPI 後端：

| 服務 | Base URL | 說明 |
|------|----------|------|
| NestJS Proxy | `http://localhost:4000/api/datasources/gdrive` | 統一入口（建議使用） |
| FastAPI Direct | `http://localhost:8000/api/v1/gdrive` | 直接存取後端 |

> **建議**：前端應一律透過 NestJS Proxy（Port 4000）存取，以統一認證與錯誤處理。FastAPI 直連僅供開發除錯使用。

### 認證方式

所有端點皆需於 HTTP Header 附帶 JWT Bearer Token：

```
Authorization: Bearer <your_jwt_token>
```

未附帶或無效的 Token 將回傳 `401 Unauthorized`。僅具備 `ADMIN` 角色的使用者可存取本 API。

### 回應格式

所有 REST API 回應均使用 `ApiResponse<T>` 包裝：

```json
{
  "success": true,
  "data": { ... },
  "message": "操作成功",
  "timestamp": "2026-02-10T10:30:00.000Z"
}
```

### 錯誤回應

```json
{
  "success": false,
  "error": {
    "code": "GDRIVE_INVALID_CREDENTIALS",
    "message": "Service Account JSON 格式無效或缺少必要欄位"
  },
  "timestamp": "2026-02-10T10:30:00.000Z"
}
```

### 常見 HTTP 狀態碼

| 狀態碼 | 說明 |
|--------|------|
| 200 | 請求成功 |
| 201 | 資源建立成功 |
| 400 | 請求參數錯誤（含驗證失敗、憑證無效等） |
| 401 | 未認證或 Token 無效 |
| 403 | 權限不足（非 Admin 角色） |
| 404 | 資源不存在 |
| 409 | 同步作業衝突（已有同步正在執行） |
| 422 | 資料驗證失敗 |
| 500 | 伺服器內部錯誤 |
| 502 | FastAPI 後端無回應 |

---

## Config API - 連線設定

### 建立或更新 GDrive 設定

建立或更新 Google Drive 連線設定。系統會先驗證 Service Account 是否能存取指定的 `folder_id`，驗證失敗時回傳 `400`。

- **方法**: `PUT`
- **路徑**: `/api/datasources/gdrive/config`
- **Content-Type**: `application/json`

#### 請求結構

| 欄位 | 型別 | 必填 | 說明 |
|------|------|------|------|
| `folder_id` | `string` | 是 | Google Drive 資料夾 ID |
| `folder_name` | `string` | 否 | 資料夾顯示名稱（未指定則由系統自動讀取） |
| `service_account_json` | `string` | 是 | GCP Service Account 金鑰 JSON 字串 |
| `sync_schedule` | `string` | 否 | 自動同步排程，預設 `"manual"`。可選值：`"manual"` / `"hourly"` / `"daily"` / `"weekly"` |
| `auto_tag` | `string` | 否 | 自動標記，匯入知識庫時作為文件標籤 |
| `hierarchical_chunking` | `boolean` | 否 | 是否啟用階層式切塊（Parent 2000 / Child 500），預設 `true` |

#### 請求範例

```json
{
  "folder_id": "1aBcDeFgHiJkLmNoPqRsTuVwXyZ",
  "folder_name": "ODA 資安知識庫",
  "service_account_json": "{\"type\":\"service_account\",\"project_id\":\"my-project\",...}",
  "sync_schedule": "daily",
  "auto_tag": "cybersecurity",
  "hierarchical_chunking": true
}
```

#### 回應結構 - `GDriveConfigResponse`

| 欄位 | 型別 | 說明 |
|------|------|------|
| `id` | `UUID` | 設定唯一識別碼 |
| `folder_id` | `string` | Google Drive 資料夾 ID |
| `folder_name` | `string` | 資料夾顯示名稱 |
| `sync_schedule` | `string` | 同步排程 |
| `is_active` | `boolean` | 是否啟用 |
| `auto_tag` | `string \| null` | 自動標記 |
| `hierarchical_chunking` | `boolean` | 是否啟用階層式切塊 |
| `last_synced_at` | `ISO 8601 \| null` | 最後同步時間 |
| `created_at` | `ISO 8601` | 建立時間 |
| `updated_at` | `ISO 8601` | 最後更新時間 |

#### 回應範例

```json
{
  "success": true,
  "data": {
    "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "folder_id": "1aBcDeFgHiJkLmNoPqRsTuVwXyZ",
    "folder_name": "ODA 資安知識庫",
    "sync_schedule": "daily",
    "is_active": true,
    "auto_tag": "cybersecurity",
    "hierarchical_chunking": true,
    "last_synced_at": null,
    "created_at": "2026-02-10T10:30:00.000Z",
    "updated_at": "2026-02-10T10:30:00.000Z"
  }
}
```

#### curl 範例

```bash
curl -X PUT http://localhost:4000/api/datasources/gdrive/config \
  -H "Authorization: Bearer <your_jwt_token>" \
  -H "Content-Type: application/json" \
  -d '{
    "folder_id": "1aBcDeFgHiJkLmNoPqRsTuVwXyZ",
    "folder_name": "ODA 資安知識庫",
    "service_account_json": "{\"type\":\"service_account\",\"project_id\":\"my-project\",\"private_key_id\":\"key123\",\"private_key\":\"-----BEGIN RSA PRIVATE KEY-----\\n...\\n-----END RSA PRIVATE KEY-----\\n\",\"client_email\":\"sa@my-project.iam.gserviceaccount.com\",\"client_id\":\"123456789\",\"auth_uri\":\"https://accounts.google.com/o/oauth2/auth\",\"token_uri\":\"https://oauth2.googleapis.com/token\"}",
    "sync_schedule": "daily",
    "auto_tag": "cybersecurity",
    "hierarchical_chunking": true
  }'
```

---

### 取得目前設定

查詢目前的 Google Drive 連線設定。`service_account_json` 欄位會被隱藏，僅回傳 `service_account_email` 供確認。

- **方法**: `GET`
- **路徑**: `/api/datasources/gdrive/config`

#### 回應結構

同 `GDriveConfigResponse`，額外包含：

| 欄位 | 型別 | 說明 |
|------|------|------|
| `service_account_email` | `string` | Service Account 的 email（從金鑰中擷取，供確認用） |

> **注意**：`service_account_json` 不會出現在回應中，以確保憑證安全。

#### 回應範例（已設定）

```json
{
  "success": true,
  "data": {
    "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "folder_id": "1aBcDeFgHiJkLmNoPqRsTuVwXyZ",
    "folder_name": "ODA 資安知識庫",
    "sync_schedule": "daily",
    "is_active": true,
    "auto_tag": "cybersecurity",
    "hierarchical_chunking": true,
    "service_account_email": "sa@my-project.iam.gserviceaccount.com",
    "last_synced_at": "2026-02-10T08:00:00.000Z",
    "created_at": "2026-02-10T10:30:00.000Z",
    "updated_at": "2026-02-10T10:30:00.000Z"
  }
}
```

#### 回應範例（尚未設定）

```json
{
  "success": true,
  "data": null
}
```

#### curl 範例

```bash
curl http://localhost:4000/api/datasources/gdrive/config \
  -H "Authorization: Bearer <your_jwt_token>"
```

---

### 中斷 GDrive 連線

移除 Google Drive 連線設定。可選擇是否同時清除已匯入 Qdrant 向量資料庫的文件資料。

- **方法**: `DELETE`
- **路徑**: `/api/datasources/gdrive/config`
- **查詢參數**:

| 參數 | 型別 | 必填 | 預設 | 說明 |
|------|------|------|------|------|
| `purge_data` | `boolean` | 否 | `false` | 是否同時刪除已匯入 Qdrant 的向量資料 |

#### 回應結構

| 欄位 | 型別 | 說明 |
|------|------|------|
| `message` | `string` | 操作結果訊息 |
| `purged` | `boolean` | 是否已清除向量資料 |

#### 回應範例

```json
{
  "success": true,
  "data": {
    "message": "Google Drive 連線已中斷，相關向量資料已清除",
    "purged": true
  }
}
```

#### curl 範例

```bash
# 僅中斷連線，保留已匯入的資料
curl -X DELETE "http://localhost:4000/api/datasources/gdrive/config" \
  -H "Authorization: Bearer <your_jwt_token>"

# 中斷連線並清除所有已匯入的向量資料
curl -X DELETE "http://localhost:4000/api/datasources/gdrive/config?purge_data=true" \
  -H "Authorization: Bearer <your_jwt_token>"
```

---

## Sync API - 同步操作

### 觸發手動同步

手動觸發一次 Google Drive 同步作業。同步作業在背景執行，立即回傳同步 ID 供後續查詢進度。

若已有同步作業正在執行，回傳 `409 Conflict`。

- **方法**: `POST`
- **路徑**: `/api/datasources/gdrive/sync`

#### 回應結構

| 欄位 | 型別 | 說明 |
|------|------|------|
| `sync_id` | `UUID` | 同步作業唯一識別碼 |
| `status` | `string` | 同步狀態，固定為 `"running"` |
| `message` | `string` | 操作訊息 |

#### 回應範例

```json
{
  "success": true,
  "data": {
    "sync_id": "b2c3d4e5-f6a7-8901-bcde-f12345678901",
    "status": "running",
    "message": "同步作業已啟動，正在掃描 Google Drive 資料夾"
  }
}
```

#### 錯誤回應範例（已有同步進行中）

```json
{
  "success": false,
  "error": {
    "code": "GDRIVE_SYNC_CONFLICT",
    "message": "已有同步作業正在執行中，請等待完成後再試"
  },
  "timestamp": "2026-02-10T10:35:00.000Z"
}
```

#### curl 範例

```bash
curl -X POST http://localhost:4000/api/datasources/gdrive/sync \
  -H "Authorization: Bearer <your_jwt_token>"
```

---

## Status API - 同步狀態

### 取得同步狀態總覽

取得 Google Drive 連線與同步的整體狀態資訊，包含是否已設定、目前追蹤的檔案數、最近一次同步的摘要。

- **方法**: `GET`
- **路徑**: `/api/datasources/gdrive/status`

#### 回應結構

| 欄位 | 型別 | 說明 |
|------|------|------|
| `is_configured` | `boolean` | 是否已完成連線設定 |
| `is_active` | `boolean` | 連線是否啟用中 |
| `folder_id` | `string \| null` | 目前設定的資料夾 ID |
| `folder_name` | `string \| null` | 資料夾顯示名稱 |
| `sync_schedule` | `string \| null` | 同步排程 |
| `last_synced_at` | `ISO 8601 \| null` | 最後成功同步時間 |
| `tracked_files` | `number` | 目前追蹤的檔案總數 |
| `latest_sync` | `SyncSummary \| null` | 最近一次同步的摘要資訊 |

#### `SyncSummary` 結構

| 欄位 | 型別 | 說明 |
|------|------|------|
| `id` | `UUID` | 同步紀錄 ID |
| `status` | `string` | 同步狀態（`running` / `completed` / `failed`） |
| `trigger_type` | `string` | 觸發方式（`manual` / `scheduled`） |
| `files_added` | `number` | 新增檔案數 |
| `files_updated` | `number` | 更新檔案數 |
| `files_deleted` | `number` | 刪除檔案數 |
| `started_at` | `ISO 8601` | 同步開始時間 |
| `completed_at` | `ISO 8601 \| null` | 同步完成時間 |

#### 回應範例（已設定且有同步紀錄）

```json
{
  "success": true,
  "data": {
    "is_configured": true,
    "is_active": true,
    "folder_id": "1aBcDeFgHiJkLmNoPqRsTuVwXyZ",
    "folder_name": "ODA 資安知識庫",
    "sync_schedule": "daily",
    "last_synced_at": "2026-02-10T08:00:00.000Z",
    "tracked_files": 42,
    "latest_sync": {
      "id": "b2c3d4e5-f6a7-8901-bcde-f12345678901",
      "status": "completed",
      "trigger_type": "scheduled",
      "files_added": 3,
      "files_updated": 1,
      "files_deleted": 0,
      "started_at": "2026-02-10T08:00:00.000Z",
      "completed_at": "2026-02-10T08:02:35.000Z"
    }
  }
}
```

#### 回應範例（尚未設定）

```json
{
  "success": true,
  "data": {
    "is_configured": false,
    "is_active": false,
    "folder_id": null,
    "folder_name": null,
    "sync_schedule": null,
    "last_synced_at": null,
    "tracked_files": 0,
    "latest_sync": null
  }
}
```

#### curl 範例

```bash
curl http://localhost:4000/api/datasources/gdrive/status \
  -H "Authorization: Bearer <your_jwt_token>"
```

---

## History API - 同步歷程

### 查詢同步歷程

取得 Google Drive 同步歷史紀錄列表，支援分頁。依開始時間倒序排列。

- **方法**: `GET`
- **路徑**: `/api/datasources/gdrive/history`
- **查詢參數**:

| 參數 | 型別 | 必填 | 預設 | 說明 |
|------|------|------|------|------|
| `limit` | `int` | 否 | 20 | 每頁筆數（最大 100） |
| `offset` | `int` | 否 | 0 | 跳過筆數 |

#### 回應結構

| 欄位 | 型別 | 說明 |
|------|------|------|
| `items` | `SyncHistoryItem[]` | 同步歷程列表 |
| `total` | `int` | 總紀錄數 |
| `limit` | `int` | 每頁筆數 |
| `offset` | `int` | 目前偏移量 |

#### `SyncHistoryItem` 結構

| 欄位 | 型別 | 說明 |
|------|------|------|
| `id` | `UUID` | 同步紀錄 ID |
| `trigger_type` | `string` | 觸發方式（`manual` / `scheduled`） |
| `status` | `string` | 同步狀態（`running` / `completed` / `failed`） |
| `files_added` | `number` | 新增檔案數 |
| `files_updated` | `number` | 更新檔案數 |
| `files_deleted` | `number` | 刪除檔案數 |
| `files_skipped` | `number` | 跳過檔案數（未變更） |
| `chunks_added` | `number` | 新增向量切塊數 |
| `error_message` | `string \| null` | 錯誤訊息（僅失敗時） |
| `started_at` | `ISO 8601` | 同步開始時間 |
| `completed_at` | `ISO 8601 \| null` | 同步完成時間 |

#### 同步狀態值

| 狀態 | 說明 |
|------|------|
| `running` | 同步執行中 |
| `completed` | 同步完成 |
| `failed` | 同步失敗 |

#### 回應範例

```json
{
  "success": true,
  "data": {
    "items": [
      {
        "id": "b2c3d4e5-f6a7-8901-bcde-f12345678901",
        "trigger_type": "manual",
        "status": "completed",
        "files_added": 5,
        "files_updated": 2,
        "files_deleted": 0,
        "files_skipped": 35,
        "chunks_added": 128,
        "error_message": null,
        "started_at": "2026-02-10T10:30:00.000Z",
        "completed_at": "2026-02-10T10:33:15.000Z"
      },
      {
        "id": "c3d4e5f6-a7b8-9012-cdef-123456789012",
        "trigger_type": "scheduled",
        "status": "completed",
        "files_added": 1,
        "files_updated": 0,
        "files_deleted": 1,
        "files_skipped": 40,
        "chunks_added": 24,
        "error_message": null,
        "started_at": "2026-02-09T08:00:00.000Z",
        "completed_at": "2026-02-09T08:01:45.000Z"
      },
      {
        "id": "d4e5f6a7-b8c9-0123-defa-234567890123",
        "trigger_type": "scheduled",
        "status": "failed",
        "files_added": 0,
        "files_updated": 0,
        "files_deleted": 0,
        "files_skipped": 0,
        "chunks_added": 0,
        "error_message": "Google Drive API 回傳 403: Service Account 無權存取指定資料夾",
        "started_at": "2026-02-08T08:00:00.000Z",
        "completed_at": "2026-02-08T08:00:02.000Z"
      }
    ],
    "total": 15,
    "limit": 20,
    "offset": 0
  }
}
```

#### curl 範例

```bash
# 取得前 10 筆同步歷程
curl "http://localhost:4000/api/datasources/gdrive/history?limit=10&offset=0" \
  -H "Authorization: Bearer <your_jwt_token>"

# 取得第二頁
curl "http://localhost:4000/api/datasources/gdrive/history?limit=10&offset=10" \
  -H "Authorization: Bearer <your_jwt_token>"
```

---

## Files API - 已同步檔案

### 查詢已同步檔案列表

取得已從 Google Drive 同步的檔案列表，支援分頁。依最後同步時間倒序排列。

- **方法**: `GET`
- **路徑**: `/api/datasources/gdrive/files`
- **查詢參數**:

| 參數 | 型別 | 必填 | 預設 | 說明 |
|------|------|------|------|------|
| `limit` | `int` | 否 | 20 | 每頁筆數（最大 100） |
| `offset` | `int` | 否 | 0 | 跳過筆數 |

#### 回應結構

| 欄位 | 型別 | 說明 |
|------|------|------|
| `items` | `SyncedFileItem[]` | 已同步檔案列表 |
| `total` | `int` | 總檔案數 |
| `limit` | `int` | 每頁筆數 |
| `offset` | `int` | 目前偏移量 |

#### `SyncedFileItem` 結構

| 欄位 | 型別 | 說明 |
|------|------|------|
| `id` | `UUID` | 系統內部檔案 ID |
| `drive_file_id` | `string` | Google Drive 檔案 ID |
| `filename` | `string` | 檔案名稱 |
| `mime_type` | `string` | 檔案 MIME 類型 |
| `md5_checksum` | `string` | MD5 校驗碼（用於變更偵測） |
| `file_size` | `number` | 檔案大小（bytes） |
| `sync_status` | `string` | 同步狀態 |
| `last_synced_at` | `ISO 8601` | 最後同步時間 |

#### 檔案同步狀態值

| 狀態 | 說明 |
|------|------|
| `synced` | 已同步，與 Drive 一致 |
| `pending` | 等待同步 |
| `error` | 同步時發生錯誤 |
| `deleted` | 已從 Drive 中刪除 |

#### 回應範例

```json
{
  "success": true,
  "data": {
    "items": [
      {
        "id": "e5f6a7b8-c9d0-1234-efab-345678901234",
        "drive_file_id": "1xYzAbCdEfGhIjKlMnOpQrStUvWx",
        "filename": "資安法規彙編_2026.pdf",
        "mime_type": "application/pdf",
        "md5_checksum": "d41d8cd98f00b204e9800998ecf8427e",
        "file_size": 2097152,
        "sync_status": "synced",
        "last_synced_at": "2026-02-10T08:00:15.000Z"
      },
      {
        "id": "f6a7b8c9-d0e1-2345-fabc-456789012345",
        "drive_file_id": "2aBcDeFgHiJkLmNoPqRsTuVwXyZa",
        "filename": "個資保護實務指引.docx",
        "mime_type": "application/vnd.openxmlformats-officedocument.wordprocessingml.document",
        "md5_checksum": "098f6bcd4621d373cade4e832627b4f6",
        "file_size": 524288,
        "sync_status": "synced",
        "last_synced_at": "2026-02-10T08:00:20.000Z"
      },
      {
        "id": "a7b8c9d0-e1f2-3456-abcd-567890123456",
        "drive_file_id": "3cDeFgHiJkLmNoPqRsTuVwXyZaBc",
        "filename": "弱點掃描報告_Q1.xlsx",
        "mime_type": "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
        "md5_checksum": "5d41402abc4b2a76b9719d911017c592",
        "file_size": 1048576,
        "sync_status": "synced",
        "last_synced_at": "2026-02-09T08:01:30.000Z"
      }
    ],
    "total": 42,
    "limit": 20,
    "offset": 0
  }
}
```

#### curl 範例

```bash
# 取得前 20 筆已同步檔案
curl "http://localhost:4000/api/datasources/gdrive/files?limit=20&offset=0" \
  -H "Authorization: Bearer <your_jwt_token>"
```

---

## 錯誤碼一覽

以下為 Google Drive 整合相關的專屬錯誤碼：

| 錯誤碼 | HTTP 狀態碼 | 說明 |
|--------|------------|------|
| `GDRIVE_NOT_CONFIGURED` | 400 | 尚未設定 Google Drive 連線 |
| `GDRIVE_INVALID_CREDENTIALS` | 400 | Service Account JSON 格式無效或缺少必要欄位 |
| `GDRIVE_FOLDER_NOT_ACCESSIBLE` | 400 | 無法存取指定的 Google Drive 資料夾（權限不足或資料夾不存在） |
| `GDRIVE_SYNC_CONFLICT` | 409 | 已有同步作業正在執行中 |
| `GDRIVE_SYNC_NOT_FOUND` | 404 | 指定的同步紀錄不存在 |
| `GDRIVE_FILE_NOT_FOUND` | 404 | 指定的檔案紀錄不存在 |
| `GDRIVE_API_ERROR` | 502 | Google Drive API 呼叫失敗 |
| `GDRIVE_QUOTA_EXCEEDED` | 429 | Google Drive API 配額已用盡 |
| `UNAUTHORIZED` | 401 | 未提供 JWT Token 或 Token 已過期 |
| `FORBIDDEN` | 403 | 非 Admin 角色，權限不足 |
| `FASTAPI_UNAVAILABLE` | 502 | FastAPI 後端服務無回應 |

### 錯誤回應範例

#### 憑證無效

```json
{
  "success": false,
  "error": {
    "code": "GDRIVE_INVALID_CREDENTIALS",
    "message": "Service Account JSON 格式無效：缺少 'client_email' 欄位"
  },
  "timestamp": "2026-02-10T10:30:00.000Z"
}
```

#### 資料夾不可存取

```json
{
  "success": false,
  "error": {
    "code": "GDRIVE_FOLDER_NOT_ACCESSIBLE",
    "message": "無法存取資料夾 '1aBcDeFg...'：請確認已將資料夾共用給 Service Account (sa@my-project.iam.gserviceaccount.com)"
  },
  "timestamp": "2026-02-10T10:30:00.000Z"
}
```

#### 同步衝突

```json
{
  "success": false,
  "error": {
    "code": "GDRIVE_SYNC_CONFLICT",
    "message": "已有同步作業正在執行中 (sync_id: b2c3d4e5-...)，請等待完成後再試"
  },
  "timestamp": "2026-02-10T10:35:00.000Z"
}
```

---

## 共用型別定義

以下型別定義供前後端共用參考：

```typescript
// ============================================================
// Google Drive 連線設定
// ============================================================

/** 同步排程選項 */
type SyncSchedule = 'manual' | 'hourly' | 'daily' | 'weekly';

/** 建立/更新 GDrive 設定 - 請求 */
interface GDriveConfigRequest {
  folder_id: string;
  folder_name?: string;
  service_account_json: string;
  sync_schedule?: SyncSchedule;
  auto_tag?: string;
  hierarchical_chunking?: boolean;
}

/** GDrive 設定 - 回應 */
interface GDriveConfigResponse {
  id: string;
  folder_id: string;
  folder_name: string;
  sync_schedule: SyncSchedule;
  is_active: boolean;
  auto_tag: string | null;
  hierarchical_chunking: boolean;
  service_account_email?: string;
  last_synced_at: string | null;
  created_at: string;
  updated_at: string;
}

/** 中斷連線 - 回應 */
interface GDriveDisconnectResponse {
  message: string;
  purged: boolean;
}

// ============================================================
// 同步操作
// ============================================================

/** 觸發同步 - 回應 */
interface GDriveSyncTriggerResponse {
  sync_id: string;
  status: 'running';
  message: string;
}

// ============================================================
// 同步狀態
// ============================================================

/** 同步摘要 */
interface SyncSummary {
  id: string;
  status: 'running' | 'completed' | 'failed';
  trigger_type: 'manual' | 'scheduled';
  files_added: number;
  files_updated: number;
  files_deleted: number;
  started_at: string;
  completed_at: string | null;
}

/** 同步狀態總覽 - 回應 */
interface GDriveStatusResponse {
  is_configured: boolean;
  is_active: boolean;
  folder_id: string | null;
  folder_name: string | null;
  sync_schedule: SyncSchedule | null;
  last_synced_at: string | null;
  tracked_files: number;
  latest_sync: SyncSummary | null;
}

// ============================================================
// 同步歷程
// ============================================================

/** 同步歷程項目 */
interface SyncHistoryItem {
  id: string;
  trigger_type: 'manual' | 'scheduled';
  status: 'running' | 'completed' | 'failed';
  files_added: number;
  files_updated: number;
  files_deleted: number;
  files_skipped: number;
  chunks_added: number;
  error_message: string | null;
  started_at: string;
  completed_at: string | null;
}

/** 同步歷程 - 分頁回應 */
interface SyncHistoryResponse {
  items: SyncHistoryItem[];
  total: number;
  limit: number;
  offset: number;
}

// ============================================================
// 已同步檔案
// ============================================================

/** 檔案同步狀態 */
type FileSyncStatus = 'synced' | 'pending' | 'error' | 'deleted';

/** 已同步檔案項目 */
interface SyncedFileItem {
  id: string;
  drive_file_id: string;
  filename: string;
  mime_type: string;
  md5_checksum: string;
  file_size: number;
  sync_status: FileSyncStatus;
  last_synced_at: string;
}

/** 已同步檔案 - 分頁回應 */
interface SyncedFilesResponse {
  items: SyncedFileItem[];
  total: number;
  limit: number;
  offset: number;
}
```

---

## 附錄：FastAPI 直連端點對照

NestJS Proxy 會將請求透明轉發至 FastAPI 後端。以下為兩者的路徑對照：

| 功能 | NestJS Proxy (Port 4000) | FastAPI Direct (Port 8000) |
|------|--------------------------|---------------------------|
| 建立/更新設定 | `PUT /api/datasources/gdrive/config` | `PUT /api/v1/gdrive/config` |
| 取得設定 | `GET /api/datasources/gdrive/config` | `GET /api/v1/gdrive/config` |
| 中斷連線 | `DELETE /api/datasources/gdrive/config` | `DELETE /api/v1/gdrive/config` |
| 觸發同步 | `POST /api/datasources/gdrive/sync` | `POST /api/v1/gdrive/sync` |
| 同步狀態 | `GET /api/datasources/gdrive/status` | `GET /api/v1/gdrive/status` |
| 同步歷程 | `GET /api/datasources/gdrive/history` | `GET /api/v1/gdrive/history` |
| 已同步檔案 | `GET /api/datasources/gdrive/files` | `GET /api/v1/gdrive/files` |

> **注意**：FastAPI 直連不經過 NestJS 的 JWT 驗證中介層，但 FastAPI 端仍會透過自身的認證機制驗證請求。開發環境除錯時方可直連。
