# Prompts Management API

## 概述

提示詞範本管理 API,用於建立、更新、查詢及測試系統提示詞範本。所有端點僅限管理員存取。

**Base URL**: `/api/prompts`
**認證**: JWT Bearer Token (Admin Only)

---

## API 端點

### 1. 列出提示詞範本

**Endpoint**: `GET /api/prompts`

**Query Parameters**:
- `limit` (number, optional): 每頁筆數,預設 20
- `offset` (number, optional): 偏移量,預設 0
- `role` (string, optional): 角色篩選 (`user` | `consultant` | `admin`)
- `mode` (string, optional): 模式篩選 (`beginner` | `standard` | `expert`)
- `isActive` (boolean, optional): 啟用狀態篩選

**Response** (200 OK):
```json
{
  "success": true,
  "data": {
    "prompts": [
      {
        "id": "uuid",
        "name": "專家模式-顧問提示詞",
        "role": "consultant",
        "mode": "expert",
        "content": "你是資深資安顧問,專長於 {{domain}}...",
        "variables": {
          "domain": "網路安全",
          "industry": "金融業"
        },
        "isActive": true,
        "createdBy": "uuid",
        "createdAt": "2026-02-11T00:00:00Z",
        "updatedAt": "2026-02-11T00:00:00Z"
      }
    ],
    "total": 1
  },
  "timestamp": "2026-02-11T00:00:00Z"
}
```

---

### 2. 取得單一提示詞範本

**Endpoint**: `GET /api/prompts/:id`

**Response** (200 OK):
```json
{
  "success": true,
  "data": {
    "id": "uuid",
    "name": "專家模式-顧問提示詞",
    "role": "consultant",
    "mode": "expert",
    "content": "你是資深資安顧問,專長於 {{domain}}...",
    "variables": {
      "domain": "網路安全"
    },
    "isActive": true,
    "createdBy": "uuid",
    "createdAt": "2026-02-11T00:00:00Z",
    "updatedAt": "2026-02-11T00:00:00Z"
  },
  "timestamp": "2026-02-11T00:00:00Z"
}
```

**Error** (404 Not Found):
```json
{
  "statusCode": 404,
  "message": "Prompt template {id} not found",
  "error": "Not Found"
}
```

---

### 3. 建立提示詞範本

**Endpoint**: `POST /api/prompts`

**Request Body**:
```json
{
  "name": "專家模式-顧問提示詞",
  "role": "consultant",
  "mode": "expert",
  "content": "你是資深資安顧問,專長於 {{domain}},服務於 {{industry}} 產業...",
  "variables": {
    "domain": "網路安全",
    "industry": "金融業"
  },
  "isActive": true
}
```

**欄位說明**:
- `name` (string, required): 範本名稱
- `role` (string, required): 角色 (`user` | `consultant` | `admin`)
- `mode` (string, required): 模式 (`beginner` | `standard` | `expert`)
- `content` (string, required): 提示詞內容,支援 `{{variableName}}` 佔位符
- `variables` (object, optional): 變數定義
- `isActive` (boolean, optional): 是否啟用,預設 false

**Response** (200 OK):
```json
{
  "success": true,
  "data": {
    "id": "uuid",
    "name": "專家模式-顧問提示詞",
    "role": "consultant",
    "mode": "expert",
    "content": "你是資深資安顧問,專長於 {{domain}},服務於 {{industry}} 產業...",
    "variables": {
      "domain": "網路安全",
      "industry": "金融業"
    },
    "isActive": true,
    "createdBy": "uuid",
    "createdAt": "2026-02-11T00:00:00Z",
    "updatedAt": "2026-02-11T00:00:00Z"
  },
  "timestamp": "2026-02-11T00:00:00Z"
}
```

---

### 4. 更新提示詞範本

**Endpoint**: `PUT /api/prompts/:id`

**Request Body**: 所有欄位皆為選填,與建立端點相同結構

```json
{
  "name": "更新後的名稱",
  "isActive": false
}
```

**Response** (200 OK): 與建立端點相同

---

### 5. 刪除提示詞範本

**Endpoint**: `DELETE /api/prompts/:id`

**Response** (204 No Content): 無回應內容

**Error** (404 Not Found): 範本不存在

---

### 6. 測試提示詞範本

**Endpoint**: `POST /api/prompts/:id/test`

用於測試提示詞變數注入效果,將變數值填入 `{{variableName}}` 佔位符。

**Request Body**:
```json
{
  "variables": {
    "domain": "雲端安全",
    "industry": "電商業"
  }
}
```

**Response** (200 OK):
```json
{
  "success": true,
  "data": {
    "original": "你是資深資安顧問,專長於 {{domain}},服務於 {{industry}} 產業...",
    "rendered": "你是資深資安顧問,專長於雲端安全,服務於電商業產業...",
    "variables": {
      "domain": "雲端安全",
      "industry": "電商業"
    }
  },
  "timestamp": "2026-02-11T00:00:00Z"
}
```

---

## 資料模型

### PromptTemplate

| 欄位 | 類型 | 說明 |
|------|------|------|
| id | string (UUID) | 唯一識別碼 |
| name | string | 範本名稱 |
| role | string | 角色 (`user`, `consultant`, `admin`) |
| mode | string | 模式 (`beginner`, `standard`, `expert`) |
| content | string | 提示詞內容 (支援 `{{變數}}` 佔位符) |
| variables | object \| null | 變數定義 (JSON) |
| isActive | boolean | 是否啟用 |
| createdBy | string \| null | 建立者 ID |
| createdAt | datetime | 建立時間 |
| updatedAt | datetime | 更新時間 |

---

## 權限要求

所有端點皆需:
- JWT 認證 (`Authorization: Bearer <token>`)
- 角色為 `admin`

---

## 錯誤回應

### 401 Unauthorized
```json
{
  "statusCode": 401,
  "message": "Unauthorized"
}
```

### 403 Forbidden
```json
{
  "statusCode": 403,
  "message": "Forbidden resource",
  "error": "Forbidden"
}
```

### 400 Bad Request (驗證失敗)
```json
{
  "statusCode": 400,
  "message": [
    "role must be one of the following values: user, consultant, admin",
    "content should not be empty"
  ],
  "error": "Bad Request"
}
```

---

## 使用範例

### 建立新手模式提示詞
```bash
curl -X POST http://localhost:4000/api/prompts \
  -H "Authorization: Bearer <admin-token>" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "新手模式-使用者提示詞",
    "role": "user",
    "mode": "beginner",
    "content": "你是友善的資安助手,請用淺顯易懂的方式解釋 {{topic}}",
    "variables": {
      "topic": "防火牆概念"
    },
    "isActive": true
  }'
```

### 測試提示詞變數注入
```bash
curl -X POST http://localhost:4000/api/prompts/{prompt-id}/test \
  -H "Authorization: Bearer <admin-token>" \
  -H "Content-Type: application/json" \
  -d '{
    "variables": {
      "topic": "多因素驗證"
    }
  }'
```

### 查詢特定角色與模式的啟用提示詞
```bash
curl -X GET "http://localhost:4000/api/prompts?role=consultant&mode=expert&isActive=true" \
  -H "Authorization: Bearer <admin-token>"
```

---

## 實作細節

### 變數注入邏輯
- 使用 `{{variableName}}` 語法定義佔位符
- `testPrompt` 方法使用 `String.replaceAll()` 進行全域替換
- 未提供的變數不會被替換,保留原佔位符

### 啟用狀態管理
- 同一 `role + mode` 組合可有多個範本,但建議僅啟用一個
- `getActivePrompt(role, mode)` 方法會依 `createdAt DESC` 取得最新的啟用範本
- 前端應用可透過此方法取得當前有效的提示詞

### 資料庫索引建議
```sql
-- 加速查詢啟用範本
CREATE INDEX idx_prompt_templates_active
ON prompt_templates(role, mode, is_active, created_at DESC);
```
