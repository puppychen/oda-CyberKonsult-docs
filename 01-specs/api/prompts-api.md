---
audience: ai-primary
---

# Prompts Management API

## 概述

提示詞只依回應層級 `mode` 選取，不再依帳號角色分流。所有端點需 JWT，且僅限 `admin`。

**Base URL**：`/api/prompts`

## 資料契約

| 欄位 | 型別 | 說明 |
|------|------|------|
| `id` | string | UUID |
| `name` | string | 顯示名稱 |
| `role` | string \| null | 舊資料相容欄位；執行期忽略 |
| `mode` | beginner \| standard \| expert | 唯一提示詞查找條件 |
| `content` | string | 支援 `{user_name}`、`{query}`、`{context}` |
| `variables` | object \| null | 變數定義 |
| `isActive` | boolean | 每個 mode 最多一筆啟用版本 |

## 端點

### 列出提示詞

`GET /api/prompts?limit=20&offset=0&mode=standard&isActive=true`

查詢參數：`limit`、`offset`、`mode`、`isActive`。`role` 已移除。

```json
{
  "success": true,
  "data": {
    "prompts": [
      {
        "id": "uuid",
        "name": "一般回應層級",
        "role": null,
        "mode": "standard",
        "content": "你是資安助手...{context}",
        "variables": {},
        "isActive": true
      }
    ],
    "total": 1
  }
}
```

### 取得單筆

`GET /api/prompts/:id`

不存在回 `404`。

### 建立版本

`POST /api/prompts`

```json
{
  "name": "一般回應層級 v2",
  "mode": "standard",
  "content": "請依據 {context} 回答 {query}",
  "variables": {},
  "isActive": true
}
```

若 `isActive=true`，同一交易內先停用目前 mode 的啟用版本，再建立新版本。Admin UI 不提供新增操作，此端點保留給版本管理與維運使用。

### 更新內容

`PUT /api/prompts/:id`

只接受 `name`、`content`、`variables`；`mode` 與 `isActive` 不可透過此端點修改。

### 刪除歷史版本

`DELETE /api/prompts/:id`

僅能刪除停用版本。啟用版本回 `409`；Admin UI 不提供刪除操作。

### 測試變數

`POST /api/prompts/:id/test`

```json
{
  "variables": {
    "user_name": "王小明",
    "query": "如何防範釣魚攻擊？",
    "context": "RAG 檢索資料"
  }
}
```

回應包含 `original`、`rendered`、`runtimeData`、`variables`。`rendered` 是模型正式收到的靜態 system message；`runtimeData` 是承載 `user_name`、`query`、`context` 的獨立 user JSON data message。執行期值不會展開到 `rendered`，避免知識庫或網頁內容取得 system 權限。
