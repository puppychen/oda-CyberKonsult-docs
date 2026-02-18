# Audit & Users API Documentation

**Version:** 1.0.0
**Base URL:** `http://localhost:4000`

---

## Audit Module (稽核日誌模組)

### 1. GET /api/audit

**說明:** 取得稽核日誌清單（僅限 Admin）

**認證:** Bearer Token (JWT)

**權限:** `admin`

**查詢參數:**

| 參數 | 型別 | 必填 | 預設值 | 說明 |
|------|------|------|--------|------|
| limit | number | 否 | 20 | 每頁筆數 |
| offset | number | 否 | 0 | 起始偏移量 |
| action | string | 否 | - | 過濾操作類型（部分比對） |
| userId | string | 否 | - | 過濾使用者 ID |
| startDate | string | 否 | - | 起始時間（ISO 8601） |
| endDate | string | 否 | - | 結束時間（ISO 8601） |

**回應範例:**

```json
{
  "success": true,
  "data": {
    "logs": [
      {
        "id": "550e8400-e29b-41d4-a716-446655440000",
        "action": "USER_LOGIN",
        "userId": "660e8400-e29b-41d4-a716-446655440001",
        "details": { "ip": "192.168.1.1", "userAgent": "Mozilla/5.0..." },
        "timestamp": "2026-02-09T10:30:00Z",
        "user": {
          "id": "660e8400-e29b-41d4-a716-446655440001",
          "email": "admin@example.com",
          "name": "Admin User"
        }
      }
    ],
    "total": 150
  },
  "timestamp": "2026-02-09T10:30:00Z"
}
```

---

### 2. GET /api/audit/:id

**說明:** 取得單一稽核日誌詳情（僅限 Admin）

**認證:** Bearer Token (JWT)

**權限:** `admin`

**路徑參數:**

| 參數 | 型別 | 說明 |
|------|------|------|
| id | string | 稽核日誌 ID (UUID) |

**回應範例:**

```json
{
  "success": true,
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "action": "RULE_UPDATED",
    "userId": "660e8400-e29b-41d4-a716-446655440001",
    "details": {
      "ruleId": "rule-123",
      "changes": { "strategy": "mask" }
    },
    "timestamp": "2026-02-09T10:30:00Z",
    "user": {
      "id": "660e8400-e29b-41d4-a716-446655440001",
      "email": "admin@example.com",
      "name": "Admin User"
    }
  },
  "timestamp": "2026-02-09T10:30:00Z"
}
```

---

## Users Module (使用者管理模組)

### 3. GET /api/users

**說明:** 取得使用者清單（僅限 Admin）

**認證:** Bearer Token (JWT)

**權限:** `admin`

**查詢參數:**

| 參數 | 型別 | 必填 | 預設值 | 說明 |
|------|------|------|--------|------|
| limit | number | 否 | 20 | 每頁筆數 |
| offset | number | 否 | 0 | 起始偏移量 |

**回應範例:**

```json
{
  "success": true,
  "data": {
    "users": [
      {
        "id": "660e8400-e29b-41d4-a716-446655440001",
        "email": "user@example.com",
        "name": "John Doe",
        "role": "user",
        "isActive": true,
        "createdAt": "2026-01-01T00:00:00Z",
        "updatedAt": "2026-02-09T10:30:00Z"
      }
    ],
    "total": 42
  },
  "timestamp": "2026-02-09T10:30:00Z"
}
```

---

### 4. GET /api/users/me

**說明:** 取得當前登入使用者資訊

**認證:** Bearer Token (JWT)

**權限:** 任何已認證使用者

**回應範例:**

```json
{
  "success": true,
  "data": {
    "id": "660e8400-e29b-41d4-a716-446655440001",
    "email": "user@example.com",
    "name": "John Doe",
    "role": "user"
  },
  "timestamp": "2026-02-09T10:30:00Z"
}
```

---

### 5. GET /api/users/:id

**說明:** 取得指定使用者資訊（僅限 Admin）

**認證:** Bearer Token (JWT)

**權限:** `admin`

**路徑參數:**

| 參數 | 型別 | 說明 |
|------|------|------|
| id | string | 使用者 ID (UUID) |

**回應範例:**

```json
{
  "success": true,
  "data": {
    "id": "660e8400-e29b-41d4-a716-446655440001",
    "email": "user@example.com",
    "name": "John Doe",
    "role": "user",
    "isActive": true,
    "createdAt": "2026-01-01T00:00:00Z",
    "updatedAt": "2026-02-09T10:30:00Z"
  },
  "timestamp": "2026-02-09T10:30:00Z"
}
```

**錯誤回應 (404):**

```json
{
  "success": false,
  "error": "User not found",
  "timestamp": "2026-02-09T10:30:00Z"
}
```

---

### 6. PUT /api/users/:id/role

**說明:** 更新使用者角色（僅限 Admin）

**認證:** Bearer Token (JWT)

**權限:** `admin`

**路徑參數:**

| 參數 | 型別 | 說明 |
|------|------|------|
| id | string | 使用者 ID (UUID) |

**請求 Body:**

```json
{
  "role": "consultant"
}
```

**Body 參數:**

| 參數 | 型別 | 必填 | 說明 |
|------|------|------|------|
| role | string | 是 | 新角色（user/consultant/admin） |

**回應範例:**

```json
{
  "success": true,
  "data": {
    "id": "660e8400-e29b-41d4-a716-446655440001",
    "email": "user@example.com",
    "name": "John Doe",
    "role": "consultant"
  },
  "timestamp": "2026-02-09T10:30:00Z"
}
```

---

## WebSocket Module (即時通訊模組)

### WebSocket 連線

**Namespace:** `/cleaning/ws`

**URL:** `ws://localhost:4000/cleaning/ws`

**支援的來源 (CORS):**
- `http://localhost:5173` (Admin Dashboard)
- `http://localhost:5174` (Chatbot UI)

---

### 連線方式

#### 1. 訂閱特定任務更新

```javascript
import io from 'socket.io-client';

const socket = io('http://localhost:4000/cleaning/ws', {
  query: { taskId: '550e8400-e29b-41d4-a716-446655440000' }
});

socket.on('task_update', (data) => {
  console.log('Task update:', data);
});
```

#### 2. 訂閱所有任務更新

```javascript
const socket = io('http://localhost:4000/cleaning/ws');

socket.on('task_update', (data) => {
  console.log('Task update:', data);
});
```

#### 3. 動態訂閱任務

```javascript
const socket = io('http://localhost:4000/cleaning/ws');

socket.emit('subscribe_task', '550e8400-e29b-41d4-a716-446655440000');

socket.on('task_update', (data) => {
  console.log('Task update:', data);
});
```

---

### 訊息格式

**事件名稱:** `task_update`

**訊息結構:**

```json
{
  "type": "progress" | "status" | "complete" | "error",
  "taskId": "550e8400-e29b-41d4-a716-446655440000",
  "status": "processing" | "completed" | "failed",
  "progress": 75,
  "message": "Processing file 3 of 4",
  "timestamp": "2026-02-09T10:30:00Z"
}
```

**欄位說明:**

| 欄位 | 型別 | 說明 |
|------|------|------|
| type | string | 訊息類型 |
| taskId | string | 任務 ID (UUID) |
| status | string | 任務狀態 |
| progress | number | 進度百分比 (0-100) |
| message | string | 狀態訊息 |
| timestamp | string | 時間戳記 (ISO 8601) |

---

## 錯誤處理

所有 API 端點統一使用以下錯誤回應格式：

```json
{
  "success": false,
  "error": "Error message",
  "timestamp": "2026-02-09T10:30:00Z"
}
```

### HTTP 狀態碼

| 狀態碼 | 說明 |
|--------|------|
| 200 | 成功 |
| 201 | 建立成功 |
| 400 | 請求參數錯誤 |
| 401 | 未認證 |
| 403 | 無權限 |
| 404 | 資源不存在 |
| 500 | 伺服器錯誤 |

---

## 認證說明

所有需要認證的端點都需要在 HTTP Header 中提供 JWT Token：

```
Authorization: Bearer <your-jwt-token>
```

Token 取得方式請參考 [Auth API 文件](./auth-api.md)。

---

## 權限說明

系統定義三種角色：

| 角色 | 權限範圍 |
|------|----------|
| `user` | 一般使用者，可使用聊天機器人 |
| `consultant` | 顧問，可使用進階對話模式 |
| `admin` | 管理員，可存取所有 API 與管理功能 |

---

## 套件依賴需求

WebSocket 功能需要以下套件：

```bash
pnpm add @nestjs/websockets @nestjs/platform-socket.io socket.io ws
pnpm add -D @types/ws
```

請在使用 WebSocket 功能前先安裝這些依賴。

---

**文件版本:** 1.0.0
**最後更新:** 2026-02-09
**維護者:** ODA Cyber Konsult Team
