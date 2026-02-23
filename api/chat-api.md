# Chat API Documentation

## 概述

Chat API 提供對話管理、訊息發送與 RAG（檢索增強生成）整合功能。所有端點皆需 JWT 認證。

**Base URL**: `http://localhost:4000/api/chat`

---

## 認證

所有請求需在 Header 包含 JWT Token：

```http
Authorization: Bearer <your_jwt_token>
```

---

## 端點清單

### 1. 取得可用回應模式

**GET** `/api/chat/modes`

根據當前使用者角色，取得可用的回應模式清單與預設模式。

#### Response (200 OK)

```json
{
  "success": true,
  "data": {
    "modes": ["beginner", "standard", "expert"],
    "defaultMode": "standard",
    "currentRole": "consultant"
  },
  "timestamp": "2026-02-11T14:30:00.000Z"
}
```

#### 角色與模式對應

| 角色 | 可用模式 | 預設模式 |
|------|----------|----------|
| user | beginner, standard | beginner |
| consultant | beginner, standard, expert | standard |
| admin | beginner, standard, expert | standard |

#### 模式說明

| 模式 | 說明 | 適用對象 |
|------|------|----------|
| beginner | 新手模式：簡化術語、提供基礎解釋、步驟化指引 | 資安初學者、中小企業主 |
| standard | 標準模式：專業用語、完整解釋、法規引用 | 一般使用者、資安從業人員 |
| expert | 顧問模式：深度分析、技術細節、進階建議 | 資安顧問、進階使用者 |

---

### 2. 發送訊息（標準模式）

**POST** `/api/chat`

建立新對話或在現有對話中發送訊息，並從 RAG 服務取得回覆。

> **注意**：NestJS 直接呼叫 LLM API（Gemini/OpenAI）生成回答，不再 proxy 至 Python RAG Service 的 `/query` 或 `/query/stream` 端點。NestJS 使用 Python RAG 的 `/retrieve` 端點進行知識庫檢索，然後自行建構上下文並呼叫 LLM。

#### Request Body

```json
{
  "question": "什麼是 PDPA？",
  "conversationId": "uuid-optional",
  "responseMode": "standard",
  "topK": 5,
  "hybrid": true,
  "stream": false
}
```

| 欄位 | 型別 | 必填 | 說明 |
|------|------|------|------|
| question | string | ✅ | 使用者問題 |
| conversationId | string | ❌ | 對話 ID（若無則建立新對話） |
| responseMode | string | ❌ | 回應模式：beginner / standard / expert（優先使用） |
| mode | string | ❌ | 回應模式別名（向後相容，建議使用 responseMode） |
| topK | number | ❌ | 檢索文件數量（1-20，預設 5） |
| hybrid | boolean | ❌ | 是否啟用混合搜尋（預設 true） |
| stream | boolean | ❌ | 是否使用串流模式（預設 false） |

#### Response (200 OK)

```json
{
  "success": true,
  "data": {
    "message": {
      "id": "msg-user-uuid",
      "role": "user",
      "content": "什麼是 PDPA？"
    },
    "answer": {
      "id": "msg-assistant-uuid",
      "role": "assistant",
      "content": "PDPA 全名為 Personal Data Protection Act...",
      "sources": [
        {
          "source": "pdpa-guide.pdf",
          "content_preview": "個人資料保護法...",
          "score": 0.92,
          "source_type": "knowledge_base"
        },
        {
          "source": "https://example.com/pdpa-article",
          "title": "個資法最新修正解析",
          "content_preview": "2024年個資法修正重點...",
          "score": 0.85,
          "source_type": "web_search"
        }
      ]
    },
    "conversationId": "conv-uuid"
  },
  "timestamp": "2026-02-09T14:30:00.000Z"
}
```

#### 回應模式行為

系統會依以下優先順序決定回應模式：
1. `responseMode` 參數（優先）
2. `mode` 參數（向後相容）
3. 預設值：`standard`

新建對話時，`responseMode` 會儲存至 Conversation 資料表供後續參考。

---

### 3. 發送訊息（串流模式）

**POST** `/api/chat/stream`

使用 Server-Sent Events (SSE) 串流回應，適合長文本生成。

> **注意**：NestJS 直接呼叫 LLM API（Gemini/OpenAI）生成回答，不再 proxy 至 Python RAG Service 的 `/query` 或 `/query/stream` 端點。NestJS 使用 Python RAG 的 `/retrieve` 端點進行知識庫檢索，然後自行建構上下文並呼叫 LLM。

#### Request Body

同標準模式（`stream` 欄位無效）。

#### Response Headers

```
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive
X-Conversation-Id: <conversation-uuid>
```

#### SSE Event Format

```
data: {"type":"chunk","content":"PDPA"}

data: {"type":"chunk","content":" 全名為"}

data: {"type":"done","messageId":"msg-uuid","conversationId":"conv-uuid","sources":[{"source":"pdpa-guide.pdf","content_preview":"個人資料保護法...","score":0.92,"source_type":"knowledge_base"},{"source":"https://example.com/pdpa-article","title":"個資法最新修正解析","content_preview":"2024年個資法修正重點...","score":0.85,"source_type":"web_search"}]}
```

| Event Type | 說明 |
|------------|------|
| chunk | 部分回應內容 |
| done | 生成完成，包含訊息 ID 與對話 ID |
| error | 發生錯誤 |

---

### 4. 列出對話

**GET** `/api/chat/conversations?limit=20&offset=0`

取得當前使用者的對話清單，包含每個對話的最後一則訊息。

#### Query Parameters

| 參數 | 型別 | 預設值 | 說明 |
|------|------|--------|------|
| limit | number | 20 | 每頁筆數（最小 1） |
| offset | number | 0 | 偏移量（最小 0） |

#### Response (200 OK)

```json
{
  "success": true,
  "data": {
    "conversations": [
      {
        "id": "conv-uuid-1",
        "userId": "user-uuid",
        "title": "什麼是 PDPA？",
        "createdAt": "2026-02-09T10:00:00.000Z",
        "updatedAt": "2026-02-09T10:05:00.000Z",
        "messages": [
          {
            "id": "msg-uuid",
            "role": "assistant",
            "content": "PDPA 全名為...",
            "createdAt": "2026-02-09T10:05:00.000Z"
          }
        ]
      }
    ],
    "total": 42
  },
  "timestamp": "2026-02-09T14:30:00.000Z"
}
```

---

### 5. 取得單一對話

**GET** `/api/chat/conversations/:id`

取得指定對話的所有訊息（依時間升序排列）。

#### Path Parameters

| 參數 | 型別 | 說明 |
|------|------|------|
| id | string (UUID) | 對話 ID |

#### Response (200 OK)

```json
{
  "success": true,
  "data": {
    "id": "conv-uuid",
    "userId": "user-uuid",
    "title": "什麼是 PDPA？",
    "createdAt": "2026-02-09T10:00:00.000Z",
    "updatedAt": "2026-02-09T10:05:00.000Z",
    "messages": [
      {
        "id": "msg-user-uuid",
        "conversationId": "conv-uuid",
        "role": "user",
        "content": "什麼是 PDPA？",
        "sources": null,
        "metadata": null,
        "createdAt": "2026-02-09T10:00:00.000Z"
      },
      {
        "id": "msg-assistant-uuid",
        "conversationId": "conv-uuid",
        "role": "assistant",
        "content": "PDPA 全名為 Personal Data Protection Act...",
        "sources": [
          {
            "source": "pdpa-guide.pdf",
            "content_preview": "個人資料保護法...",
            "score": 0.92,
            "source_type": "knowledge_base"
          },
          {
            "source": "https://example.com/pdpa-article",
            "title": "個資法最新修正解析",
            "content_preview": "2024年個資法修正重點...",
            "score": 0.85,
            "source_type": "web_search"
          }
        ],
        "metadata": {
          "usage": {
            "total_tokens": 512,
            "total_cost_usd": 0.0032
          }
        },
        "createdAt": "2026-02-09T10:05:00.000Z"
      }
    ]
  },
  "timestamp": "2026-02-09T14:30:00.000Z"
}
```

---

### 6. 刪除對話

**DELETE** `/api/chat/conversations/:id`

刪除指定對話及其所有訊息（Cascade Delete）。

#### Path Parameters

| 參數 | 型別 | 說明 |
|------|------|------|
| id | string (UUID) | 對話 ID |

#### Response (200 OK)

```json
{
  "success": true,
  "data": {
    "deleted": true
  },
  "timestamp": "2026-02-09T14:30:00.000Z"
}
```

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

### 404 Not Found

```json
{
  "statusCode": 404,
  "message": "Conversation not found",
  "error": "Not Found"
}
```

### 400 Bad Request

```json
{
  "statusCode": 400,
  "message": [
    "question should not be empty",
    "topK must be between 1 and 20"
  ],
  "error": "Bad Request"
}
```

---

## RAG 整合

### 內部呼叫流程

```
ChatController → ChatService → RagProxyService → FastAPI (localhost:8000)
```

### RAG Retrieve Request (Internal)

NestJS ChatService 呼叫 Python RAG `/api/v1/rag/retrieve` 端點時的請求格式：

```typescript
{
  question: string;
  top_k?: number;        // 預設 5
  hybrid?: boolean;      // 預設 true
  hierarchical?: boolean; // 預設 true（ChatService 固定帶入）
  rerank?: boolean;
  source_filter?: string | null;
  tag_filter?: string | null;
}
```

### RAG Query Response (Internal)

```typescript
{
  answer: string;
  sources: Array<{
    source: string;             // 來源路徑或 URL
    content_preview: string;    // 內容預覽
    score: number;              // 相關性分數
    source_type: 'knowledge_base' | 'web_search';  // 來源類型
    title?: string;             // 網頁標題（僅 web_search 類型時包含）
  }>;
  usage?: {
    total_tokens: number;
    total_cost_usd: number;
  };
}
```

> **檢索模式**：ChatService 預設使用 `hybrid: true, hierarchical: true`，即 Hybrid Hierarchical 模式（BM25 + 向量搜尋 child chunks → 回傳 parent chunks），兼顧法規條文編號的精確匹配與完整章節上下文。

#### source_type 說明

| 類型 | 說明 | 額外欄位 |
|------|------|----------|
| knowledge_base | 來自 Qdrant 知識庫的文件 | 無 |
| web_search | 來自網路搜尋的結果 | `title`（網頁標題） |

---

## 使用範例

### cURL - 發送訊息

```bash
curl -X POST http://localhost:4000/api/chat \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "question": "什麼是 PDPA？",
    "topK": 5,
    "hybrid": true
  }'
```

### cURL - 串流模式

```bash
curl -X POST http://localhost:4000/api/chat/stream \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "question": "請詳細說明資料去識別化流程",
    "conversationId": "existing-conv-uuid"
  }'
```

### JavaScript (Fetch) - 標準模式

```javascript
const response = await fetch('http://localhost:4000/api/chat', {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${token}`,
    'Content-Type': 'application/json',
  },
  body: JSON.stringify({
    question: '什麼是 PDPA？',
    topK: 5,
    hybrid: true,
  }),
});

const result = await response.json();
console.log(result.data.answer.content);
```

### JavaScript (EventSource) - 串流模式

```javascript
// 注意：EventSource 不支援自訂 Header，需使用 fetch + ReadableStream
const response = await fetch('http://localhost:4000/api/chat/stream', {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${token}`,
    'Content-Type': 'application/json',
  },
  body: JSON.stringify({
    question: '請詳細說明資料去識別化流程',
  }),
});

const conversationId = response.headers.get('X-Conversation-Id');
const reader = response.body.getReader();
const decoder = new TextDecoder();

while (true) {
  const { done, value } = await reader.read();
  if (done) break;

  const chunk = decoder.decode(value);
  const lines = chunk.split('\n\n');

  for (const line of lines) {
    if (line.startsWith('data: ')) {
      const data = JSON.parse(line.slice(6));

      if (data.type === 'chunk') {
        console.log(data.content); // 即時顯示
      } else if (data.type === 'done') {
        console.log('Done:', data.messageId);
      }
    }
  }
}
```

---

## 資料模型

### Conversation

| 欄位 | 型別 | 說明 |
|------|------|------|
| id | UUID | 主鍵 |
| userId | UUID | 使用者 ID |
| title | string? | 對話標題（預設為首則問題的前 50 字） |
| responseMode | string | 回應模式（beginner / standard / expert，預設 standard） |
| createdAt | DateTime | 建立時間 |
| updatedAt | DateTime | 更新時間 |
| messages | Message[] | 訊息列表 |

### Message

| 欄位 | 型別 | 說明 |
|------|------|------|
| id | UUID | 主鍵 |
| conversationId | UUID | 對話 ID |
| role | string | 角色（user / assistant） |
| content | string | 訊息內容 |
| sources | JSON? | RAG 來源文件（僅 assistant） |
| metadata | JSON? | 額外資訊（如 token 使用量） |
| createdAt | DateTime | 建立時間 |

---

## 注意事項

1. **回應模式權限**：user 角色僅能使用 beginner 和 standard 模式，consultant 和 admin 可使用全部三種模式。
2. **模式欄位優先順序**：建議使用 `responseMode`，`mode` 僅作向後相容保留。
3. **串流模式限制**：EventSource 不支援自訂 Header，前端需使用 Fetch API + ReadableStream。
4. **對話權限**：使用者僅能存取自己的對話，跨使用者存取會返回 403 Forbidden。
5. **自動標題生成**：若未提供 `title`，系統會自動取問題前 50 字作為標題。
6. **Cascade Delete**：刪除對話時會自動刪除所有關聯訊息。
7. **RAG 超時**：RAG 服務請求超時設定為 120 秒。
8. **環境變數**：需設定 `FASTAPI_BASE_URL`（預設 `http://localhost:8000`）。
9. **來源類型**：`sources` 陣列中每個來源包含 `source_type` 欄位，值為 `knowledge_base`（知識庫）或 `web_search`（網路搜尋）。當 `source_type` 為 `web_search` 時，額外包含 `title` 欄位（網頁標題）。

---

## 相關文件

- [RAG API Documentation](./rag-api.md)
- [Authentication API](./auth-api.md)
- [Prisma Schema](/apps/api/prisma/schema.prisma)
