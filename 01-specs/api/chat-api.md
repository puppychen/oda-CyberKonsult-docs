---
audience: ai-primary
---

# Chat API Documentation

## 概述

Chat API 提供對話管理、訊息發送與 RAG（檢索增強生成）整合功能。所有端點皆需 JWT 認證。

**Base URL**: `http://localhost:3051/api/chat`

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

根據當前使用者角色，取得可用回應層級、預設層級與每日額度。API 欄位保留 `mode` 以維持相容。

#### Response (200 OK)

```json
{
  "success": true,
  "data": {
    "modes": ["beginner"],
    "defaultMode": "beginner",
    "currentRole": "basic_user",
    "quota": {
      "limited": true,
      "limit": 20,
      "used": 3,
      "remaining": 17,
      "resetAt": "2026-07-11T16:00:00.000Z"
    }
  }
}
```

角色矩陣：`basic_user` 僅 beginner；`user` 為 beginner / standard；`consultant` 為三層；`data_cleaner`、`data_reviewer` 僅 standard；`admin` 為三層。

### 1.1 管理免費方案額度

`GET /api/chat/usage-config` 與 `PUT /api/chat/usage-config` 僅限 `admin`。

```json
{ "freeDailyMessageLimit": 20 }
```

有效範圍為 1～1000，台北時間每日 00:00 重置。新手額度用完時發送端點回 `429` 與 `CHAT_DAILY_QUOTA_EXCEEDED`。

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
| basic_user | beginner | beginner |
| user | beginner, standard | standard |
| consultant | beginner, standard, expert | expert |
| admin | beginner, standard, expert | standard |

#### 模式說明

| 模式 | 說明 | 適用對象 |
|------|------|----------|
| beginner | 新手模式：簡化術語、提供基礎解釋、步驟化指引 | 資安初學者、中小企業主 |
| standard | 一般回應：專業用語、完整解釋、法規引用 | IT/MIS 工程師、資安從業人員 |
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
  "stream": false,
  "regenerateFromMessageId": "assistant-message-uuid-optional",
  "regenerationAttemptId": "attempt-uuid-required-for-regeneration"
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
| regenerateFromMessageId | UUID | ❌ | 重新產出的原助理回覆 ID；必須同時提供既有 `conversationId`，且該 ID 必須是對話最後一則訊息 |
| regenerationAttemptId | UUID | 條件必填 | 提供 `regenerateFromMessageId` 時必填；同一次斷線重試必須沿用相同 UUID |

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
      "topicScope": "cybersecurity",
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
          "score": 0,
          "source_type": "web_search"
        }
      ]
    },
    "conversationId": "conv-uuid"
  },
  "timestamp": "2026-02-09T14:30:00.000Z"
}
```

#### 來源資料契約

`sources` 會隨助理訊息寫入 PostgreSQL，串流模式則在 `done` 事件一次送出完整陣列。Chatbot 以同一個「N 個引用來源」收合區呈現知識庫與網路來源，預設收合；切換或重新載入歷史對話時仍可展開查看。

| `source_type` | 必要欄位 | 前端呈現 |
|---------------|----------|----------|
| `knowledge_base` | `source`、`content_preview`、`score` | 文件名稱、內容摘要與相關度 |
| `web_search` | `source`（URL）、`title`、`content_preview`、`score` | 網頁標題、可開啟的原始連結、內容摘要與「網路搜尋」標籤 |

沒有來源時可回傳空陣列或省略 `sources`，Chatbot 不顯示空白收合區。網路搜尋來源的 `score` 目前固定為 `0`，前端不將其解讀為知識庫相關度。

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

data: {"type":"done","messageId":"msg-uuid","conversationId":"conv-uuid","topicScope":"cybersecurity","sources":[{"source":"pdpa-guide.pdf","content_preview":"個人資料保護法...","score":0.92,"source_type":"knowledge_base"},{"source":"https://example.com/pdpa-article","title":"個資法最新修正解析","content_preview":"2024年個資法修正重點...","score":0,"source_type":"web_search"}]}
```

| Event Type | 說明 |
|------------|------|
| chunk | 部分回應內容 |
| done | 生成完成，包含訊息 ID 與對話 ID |
| error | 發生錯誤 |

#### `topicScope` 議題範圍

`topicScope` 可能為 `cybersecurity`、`mixed`、`non_cybersecurity` 或 `unclear`。非資安與不明確問題仍會先送出一個固定文字的 `chunk`，再送出 `done`；其 `sources` 為空陣列，且不包含信心度。

#### 重新產出最新回覆

重新產出沿用相同端點與 SSE 格式。前端重送最後提示詞，並提供目前對話 ID、目前選擇的 `mode` 與原助理回覆 ID：

```json
{
  "question": "公司發生資料外洩時，第一時間應該怎麼處理？",
  "conversationId": "conversation-uuid",
  "mode": "standard",
  "hybrid": true,
  "regenerateFromMessageId": "assistant-message-uuid",
  "regenerationAttemptId": "attempt-uuid"
}
```

成功時，原問答保持不變，API 另外建立一則相同內容的使用者訊息與一則新助理訊息。兩則新訊息的 metadata 同時記錄 `regeneratedFromMessageId`、`regenerationAttemptId` 與 `regenerationLeaseId`；待完成使用者訊息另記錄 `regenerationStatus` 與 `regenerationClaimedAt`。新的 attempt 視為一次新用量；相同 attempt 的 failed／逾時重試或已落盤回答回放不重複扣額度。系統會加入獨立回答指示，但不以犧牲正確性換取文字差異。

API 使用 conversation row lock 串行化訊息寫入，並驗證來源回答的 `topicScope` 必須是 `cybersecurity`／`mixed`，且其前一則 user prompt 必須與 `question` 完全相同。不同 attempt 的並發請求只有第一個可建立待完成訊息；額度保留與 claim 建立在同一資料庫交易內提交或回滾，訊息 `createdAt` 會在同一對話內維持嚴格遞增。claim 與回答以 conversation、來源、attempt 精確定位，其他分頁插入訊息不影響重試或回放。前置處理更新 metadata、assistant 寫入及失敗釋放都會比對最新 lease ID，避免逾時接手後舊 worker 把舊 lease 寫回、重複落盤或把新 worker 標為失敗。全程沿用 JSON metadata，不需新增資料庫欄位。

| 狀況 | HTTP | 行為 |
|------|------|------|
| 未提供 `conversationId` | 400 | 拒絕重新產出，不扣額度、不寫入訊息 |
| 未提供 `regenerationAttemptId` | 400 | 拒絕重新產出，不扣額度、不寫入訊息 |
| `question` 與來源回答前一則 user prompt 不同 | 400 | 拒絕請求，避免把新問題偽裝成重新產出 |
| 來源回答的 `topicScope` 不是 `cybersecurity`／`mixed` | 400 | 拒絕請求；即使直接呼叫 API 也不能繞過前端限制 |
| 原回覆不是對話最後一則訊息 | 400 | 拒絕重新產出，不扣額度、不寫入訊息；只有 metadata 同時符合相同來源與相同 attempt 的 pending／completed 重試例外 |
| 最後一則訊息不是助理回覆 | 400 | 拒絕重新產出，不扣額度、不寫入訊息 |
| 相同 attempt 的 user 訊息仍為 `processing` 且 5 分鐘 lease 有效 | 409 | 回傳「重新產出仍在進行中，請稍後再試」；不重複呼叫 LLM、不重複扣額度 |
| 相同 attempt 的 `processing` lease 已逾時 | 200 | 換發 lease ID 並沿用 user 訊息；舊 worker 後續寫入會回 409，失敗回報也不得釋放新 lease |
| SSE 中斷或伺服器錯誤後，同 attempt 的 user 訊息為 `failed` | 200 | 沿用 user 訊息並恢復為 `processing`，不重複扣額度 |
| 相同 attempt 的 assistant 已落盤但 `done` 遺失 | 200 | 回放相同回答、來源與 message ID，不重新呼叫 LLM、不重複扣額度 |
| 新手額度已用完 | 429 | 回傳 `CHAT_DAILY_QUOTA_EXCEEDED` |

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

### 4.1 管理者列出全部對話

**GET** `/api/chat/admin/conversations?limit=20&offset=0`

僅限管理者使用。每筆資料包含使用者摘要、最新一則訊息，以及該對話實際儲存的訊息總數。

```json
{
  "success": true,
  "data": {
    "conversations": [
      {
        "id": "conv-uuid-1",
        "userId": "user-uuid",
        "title": "資安問題處理策略",
        "responseMode": "standard",
        "messageCount": 6,
        "messages": [
          {
            "id": "latest-message-uuid",
            "role": "assistant",
            "content": "最新一則回答",
            "createdAt": "2026-07-13T01:06:00.000Z"
          }
        ],
        "user": {
          "id": "user-uuid",
          "email": "user@example.com",
          "name": "使用者"
        },
        "createdAt": "2026-07-13T01:00:00.000Z",
        "updatedAt": "2026-07-13T01:06:00.000Z"
      }
    ],
    "total": 1
  },
  "timestamp": "2026-07-13T01:10:00.000Z"
}
```

`messageCount` 計算該對話中所有 `user` 與 `assistant` 訊息。完成一回合問答通常為 2；若 AI 回覆前發生錯誤或串流中斷，可能出現單數。`messages` 僅保留最新一則供列表摘要使用，不可用其陣列長度推算總數。

### 4.2 管理者取得單一對話詳情

**GET** `/api/chat/admin/conversations/:id`

僅限管理者使用。回應包含完整訊息記錄與建立該對話的使用者摘要：

```json
{
  "success": true,
  "data": {
    "id": "conv-uuid-1",
    "userId": "user-uuid",
    "title": "資安問題處理策略",
    "responseMode": "standard",
    "user": {
      "id": "user-uuid",
      "email": "user@example.com",
      "name": "使用者"
    },
    "messages": []
  }
}
```

一般使用者的 `/api/chat/conversations/:id` 維持只回傳本人對話與訊息，不額外回傳使用者摘要。

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
            "score": 0,
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
ChatController → ChatService → RagProxyService → FastAPI (localhost:3502)
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
curl -X POST http://localhost:3051/api/chat \
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
curl -X POST http://localhost:3051/api/chat/stream \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "question": "請詳細說明資料去識別化流程",
    "conversationId": "existing-conv-uuid"
  }'
```

### JavaScript (Fetch) - 標準模式

```javascript
const response = await fetch('http://localhost:3051/api/chat', {
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
const response = await fetch('http://localhost:3051/api/chat/stream', {
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

1. **回應層級權限**：basic_user 僅能使用 beginner；user 可使用 beginner、standard；consultant 與 admin 可使用全部三種層級。
2. **模式欄位優先順序**：建議使用 `responseMode`，`mode` 僅作向後相容保留。
3. **串流模式限制**：EventSource 不支援自訂 Header，前端需使用 Fetch API + ReadableStream。
4. **對話權限**：使用者僅能存取自己的對話，跨使用者存取會返回 403 Forbidden。
5. **自動標題生成**：若未提供 `title`，系統會自動取問題前 50 字作為標題。
6. **Cascade Delete**：刪除對話時會自動刪除所有關聯訊息。
7. **RAG 超時**：RAG 服務請求超時設定為 120 秒。
8. **環境變數**：需設定 `FASTAPI_BASE_URL`（預設 `http://localhost:3502`）。
9. **來源類型**：`sources` 陣列中每個來源包含 `source_type` 欄位，值為 `knowledge_base`（知識庫）或 `web_search`（網路搜尋）。當 `source_type` 為 `web_search` 時，額外包含 `title` 欄位（網頁標題）。

---

## 相關文件

- [RAG API Documentation](./rag-api.md)
- [Authentication API](./auth-api.md)
- [Prisma Schema](/apps/api/prisma/schema.prisma)
