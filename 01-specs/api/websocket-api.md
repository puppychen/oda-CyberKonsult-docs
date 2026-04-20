# WebSocket API 規格

> **ODA Cyber Konsult - WebSocket 事件規格**
>
> 建立日期：2026-03-16

## 概述

NestJS WebSocket Gateway 使用 socket.io 協議，轉發 FastAPI 端的清洗任務進度事件至前端。

## 連線端點

| 項目 | 說明 |
|------|------|
| URL | `ws://localhost:3051` |
| 協議 | socket.io v4 |
| 認證 | JWT Bearer Token（handshake query 參數） |
| 命名空間 | `/`（預設） |

## 認證

```javascript
const socket = io('http://localhost:3051', {
  query: { token: accessToken }
});
```

## 事件清單

### 伺服器 → 客戶端

| 事件名稱 | 說明 | Payload |
|----------|------|---------|
| `task:progress` | 清洗任務進度更新 | `{ taskId, progress, filesProcessed, filesTotal, currentFile }` |
| `task:completed` | 清洗任務完成 | `{ taskId, status: 'completed', totalEntitiesFound }` |
| `task:failed` | 清洗任務失敗 | `{ taskId, status: 'failed', error }` |

### 客戶端 → 伺服器

| 事件名稱 | 說明 | Payload |
|----------|------|---------|
| `subscribe:task` | 訂閱任務進度 | `{ taskId }` |
| `unsubscribe:task` | 取消訂閱 | `{ taskId }` |

## Payload 範例

### task:progress

```json
{
  "taskId": "task-uuid",
  "progress": 66,
  "filesProcessed": 2,
  "filesTotal": 3,
  "currentFile": "document.pdf"
}
```

### task:completed

```json
{
  "taskId": "task-uuid",
  "status": "completed",
  "totalEntitiesFound": 42,
  "processingTimeSeconds": 12.5
}
```

## 錯誤處理

| 錯誤 | 處理方式 |
|------|---------|
| Token 無效 | 連線被拒絕，回傳 `connect_error` |
| 連線中斷 | socket.io 自動重連（預設 3 次） |
| Dead connection | 伺服器端清除，避免記憶體洩漏 |

## 相關文件

| 文件 | 路徑 |
|------|------|
| WebSocket Gateway 實作 | `apps/api/src/modules/websocket/` |
| 前端連接 | Admin Dashboard `/tasks` 頁面 |
