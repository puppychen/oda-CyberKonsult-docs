# 資料分析 API 文件

> **版本**: v1
> **基礎路徑**: `/api/v1/analytics`
> **服務**: NestJS API (Port 4000) → FastAPI (Port 8000)
>
> **使用範圍**：本 API 僅供系統管理員與資料清洗人員使用，提供清洗任務統計、知識庫概覽及時間軸分析等數據。

## 目錄

- [通用說明](#通用說明)
- [端點列表](#端點列表)
- [清洗統計](#清洗統計)
- [知識庫統計](#知識庫統計)
- [時間軸統計](#時間軸統計)
- [共用型別定義](#共用型別定義)

---

## 通用說明

### Base URL

```
http://localhost:4000/api/v1
```

### 認證方式

所有端點皆需 JWT Bearer Token，於 HTTP Header 中帶入：

```http
Authorization: Bearer <accessToken>
```

### 允許角色

| 角色 | 說明 |
|------|------|
| `admin` | 系統管理員，具備完整數據存取權限 |
| `data_cleaner` | 資料清洗人員，具備清洗相關數據存取權限 |

### 回應格式

所有 REST API 回應均使用 `ApiResponse<T>` 包裝：

```json
{
  "success": true,
  "data": { ... },
  "timestamp": "2026-02-11T10:30:00.000Z"
}
```

### 錯誤回應

```json
{
  "success": false,
  "error": {
    "code": "ANALYTICS_ERROR",
    "message": "統計資料查詢失敗"
  },
  "timestamp": "2026-02-11T10:30:00.000Z"
}
```

### 常見 HTTP 狀態碼

| 狀態碼 | 說明 |
|--------|------|
| 200 | 請求成功 |
| 400 | 請求參數錯誤 |
| 401 | 未授權（token 無效或過期） |
| 403 | 權限不足 |
| 500 | 伺服器內部錯誤 |

---

## 端點列表

| 方法 | 路徑 | 說明 | 需驗證 |
|------|------|------|--------|
| GET | `/api/v1/analytics/cleaning` | 清洗統計 | 是 |
| GET | `/api/v1/analytics/knowledge-base` | 知識庫統計 | 是 |
| GET | `/api/v1/analytics/timeline` | 時間軸統計 | 是 |

---

## 清洗統計

### 取得清洗統計資料

取得清洗任務的整體統計概覽，包含任務狀態分佈、成功率、平均處理時間、實體偵測統計及審批狀態分佈。適用於 Admin Dashboard 的統計面板呈現。

- **方法**: `GET`
- **路徑**: `/api/v1/analytics/cleaning`

#### 回應結構 - `CleaningAnalyticsResponse`

| 欄位 | 型別 | 說明 |
|------|------|------|
| `total_tasks` | `int` | 歷史任務總數 |
| `status_counts` | `StatusCounts` | 各任務狀態的計數 |
| `success_rate` | `float` | 任務成功率（0-1，已完成 / 總完結任務數） |
| `avg_processing_time_seconds` | `float` | 平均處理時間（秒） |
| `total_entities_found` | `int` | 所有任務偵測到的實體總數 |
| `entity_distribution` | `Record<string, int>` | 依實體類型的偵測數量分佈 |
| `approval_counts` | `ApprovalCounts` | 審批狀態分佈 |

#### `StatusCounts` 結構

| 欄位 | 型別 | 說明 |
|------|------|------|
| `pending` | `int` | 等待處理的任務數 |
| `processing` | `int` | 處理中的任務數 |
| `completed` | `int` | 已完成的任務數 |
| `failed` | `int` | 失敗的任務數 |
| `cancelled` | `int` | 已取消的任務數 |

#### `ApprovalCounts` 結構

| 欄位 | 型別 | 說明 |
|------|------|------|
| `pending_review` | `int` | 等待審核的任務數 |
| `approved` | `int` | 已批准的任務數 |
| `rejected` | `int` | 已退回的任務數 |
| `ingested` | `int` | 已匯入知識庫的任務數 |

#### 回應範例

```json
{
  "success": true,
  "data": {
    "total_tasks": 156,
    "status_counts": {
      "pending": 2,
      "processing": 1,
      "completed": 140,
      "failed": 8,
      "cancelled": 5
    },
    "success_rate": 0.946,
    "avg_processing_time_seconds": 45.3,
    "total_entities_found": 12847,
    "entity_distribution": {
      "PERSON": 4521,
      "TW_ID": 2103,
      "EMAIL_ADDRESS": 1876,
      "PHONE_NUMBER": 1543,
      "TW_PHONE": 987,
      "DATE_TIME": 832,
      "LOCATION": 456,
      "ORGANIZATION": 312,
      "CREDIT_CARD": 98,
      "IP_ADDRESS": 67,
      "URL": 52
    },
    "approval_counts": {
      "pending_review": 12,
      "approved": 95,
      "rejected": 33,
      "ingested": 82
    }
  }
}
```

#### curl 範例

```bash
curl http://localhost:4000/api/v1/analytics/cleaning \
  -H "Authorization: Bearer <accessToken>"
```

---

## 知識庫統計

### 取得知識庫統計資料

取得 RAG 知識庫的整體統計概覽，包含總 Chunk 數、來源文件列表、標籤分佈及 Qdrant 連線狀態。適用於 Admin Dashboard 的知識庫管理面板。

- **方法**: `GET`
- **路徑**: `/api/v1/analytics/knowledge-base`

#### 回應結構 - `KnowledgeBaseAnalyticsResponse`

| 欄位 | 型別 | 說明 |
|------|------|------|
| `total_chunks` | `int` | 知識庫中的向量 Chunk 總數 |
| `sources` | `SourceInfo[]` | 來源文件統計列表 |
| `tags` | `TagInfo[]` | 標籤統計列表 |
| `status` | `string` | Qdrant 向量資料庫連線狀態（`green` / `yellow` / `red`） |

#### `SourceInfo` 結構

| 欄位 | 型別 | 說明 |
|------|------|------|
| `source` | `string` | 來源文件路徑或標識 |
| `chunk_count` | `int` | 該來源的 Chunk 數 |
| `ingested_at` | `ISO 8601` | 匯入時間 |

#### `TagInfo` 結構

| 欄位 | 型別 | 說明 |
|------|------|------|
| `tag` | `string` | 標籤名稱 |
| `chunk_count` | `int` | 擁有此標籤的 Chunk 數 |

#### 回應範例

```json
{
  "success": true,
  "data": {
    "total_chunks": 1523,
    "sources": [
      {
        "source": "review://c3d4e5f6/Q1_財務報告.pdf",
        "chunk_count": 45,
        "ingested_at": "2026-02-10T15:00:00.000Z"
      },
      {
        "source": "review://d4e5f6a7/資安政策手冊.docx",
        "chunk_count": 120,
        "ingested_at": "2026-02-08T10:30:00.000Z"
      },
      {
        "source": "gdrive://1a2b3c4d/個資保護法規彙整.pdf",
        "chunk_count": 85,
        "ingested_at": "2026-02-05T09:00:00.000Z"
      }
    ],
    "tags": [
      { "tag": "資安政策", "chunk_count": 350 },
      { "tag": "法規遵循", "chunk_count": 280 },
      { "tag": "財務", "chunk_count": 210 },
      { "tag": "已審核", "chunk_count": 185 },
      { "tag": "客戶資料", "chunk_count": 120 },
      { "tag": "2026-Q1", "chunk_count": 95 }
    ],
    "status": "green"
  }
}
```

#### 狀態值說明

| 狀態 | 說明 |
|------|------|
| `green` | Qdrant 連線正常，所有 Shard 運作中 |
| `yellow` | Qdrant 連線正常，部分 Shard 處於恢復中 |
| `red` | Qdrant 連線異常或不可用 |

#### curl 範例

```bash
curl http://localhost:4000/api/v1/analytics/knowledge-base \
  -H "Authorization: Bearer <accessToken>"
```

---

## 時間軸統計

### 取得時間軸統計資料

取得指定天數區間的逐日統計資料，包含每日的任務數量與實體偵測數量。適用於 Admin Dashboard 的趨勢圖表呈現。

- **方法**: `GET`
- **路徑**: `/api/v1/analytics/timeline`
- **查詢參數**:

| 參數 | 型別 | 必填 | 預設 | 範圍 | 說明 |
|------|------|------|------|------|------|
| `days` | `int` | 否 | 30 | 1-365 | 查詢的天數區間（從今天往前推算） |

#### 回應結構 - `TimelineAnalyticsResponse`

| 欄位 | 型別 | 說明 |
|------|------|------|
| `days` | `int` | 查詢的天數區間 |
| `timeline` | `TimelineEntry[]` | 逐日統計資料列表（依日期升序排列） |

#### `TimelineEntry` 結構

| 欄位 | 型別 | 說明 |
|------|------|------|
| `date` | `string` | 日期（格式：`YYYY-MM-DD`） |
| `task_count` | `int` | 當日建立的任務數 |
| `entity_count` | `int` | 當日所有任務偵測到的實體總數 |

#### 回應範例 - 預設 30 天

```json
{
  "success": true,
  "data": {
    "days": 30,
    "timeline": [
      { "date": "2026-01-13", "task_count": 3, "entity_count": 125 },
      { "date": "2026-01-14", "task_count": 5, "entity_count": 210 },
      { "date": "2026-01-15", "task_count": 2, "entity_count": 87 },
      { "date": "2026-01-16", "task_count": 0, "entity_count": 0 },
      { "date": "2026-01-17", "task_count": 4, "entity_count": 156 },
      "... (省略中間日期)",
      { "date": "2026-02-10", "task_count": 6, "entity_count": 342 },
      { "date": "2026-02-11", "task_count": 1, "entity_count": 47 }
    ]
  }
}
```

#### 回應範例 - 自訂 7 天

```json
{
  "success": true,
  "data": {
    "days": 7,
    "timeline": [
      { "date": "2026-02-05", "task_count": 4, "entity_count": 198 },
      { "date": "2026-02-06", "task_count": 2, "entity_count": 89 },
      { "date": "2026-02-07", "task_count": 5, "entity_count": 267 },
      { "date": "2026-02-08", "task_count": 3, "entity_count": 145 },
      { "date": "2026-02-09", "task_count": 0, "entity_count": 0 },
      { "date": "2026-02-10", "task_count": 6, "entity_count": 342 },
      { "date": "2026-02-11", "task_count": 1, "entity_count": 47 }
    ]
  }
}
```

#### 參數驗證錯誤

```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "days 參數必須介於 1 至 365 之間"
  },
  "timestamp": "2026-02-11T10:30:00.000Z"
}
```

#### curl 範例

```bash
# 預設 30 天
curl http://localhost:4000/api/v1/analytics/cleaning \
  -H "Authorization: Bearer <accessToken>"

# 自訂天數
curl "http://localhost:4000/api/v1/analytics/timeline?days=7" \
  -H "Authorization: Bearer <accessToken>"

# 查詢最近一年
curl "http://localhost:4000/api/v1/analytics/timeline?days=365" \
  -H "Authorization: Bearer <accessToken>"
```

---

## 共用型別定義

以下型別定義可納入 `packages/shared-types/` 中供前後端共用：

```typescript
// 清洗統計回應
interface CleaningAnalyticsResponse {
  total_tasks: number;
  status_counts: StatusCounts;
  success_rate: number;
  avg_processing_time_seconds: number;
  total_entities_found: number;
  entity_distribution: Record<string, number>;
  approval_counts: ApprovalCounts;
}

// 任務狀態計數
interface StatusCounts {
  pending: number;
  processing: number;
  completed: number;
  failed: number;
  cancelled: number;
}

// 審批狀態計數
interface ApprovalCounts {
  pending_review: number;
  approved: number;
  rejected: number;
  ingested: number;
}

// 知識庫統計回應
interface KnowledgeBaseAnalyticsResponse {
  total_chunks: number;
  sources: SourceInfo[];
  tags: TagInfo[];
  status: 'green' | 'yellow' | 'red';
}

// 來源資訊
interface SourceInfo {
  source: string;
  chunk_count: number;
  ingested_at: string;
}

// 標籤資訊
interface TagInfo {
  tag: string;
  chunk_count: number;
}

// 時間軸統計回應
interface TimelineAnalyticsResponse {
  days: number;
  timeline: TimelineEntry[];
}

// 時間軸每日項目
interface TimelineEntry {
  date: string;
  task_count: number;
  entity_count: number;
}
```
