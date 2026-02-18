# Chat Module 快速開始指南

## 前置需求

確保以下服務正在運行：

```bash
# 1. PostgreSQL (Docker)
docker ps | grep postgres

# 2. RAG FastAPI Service
curl http://localhost:8000/health

# 3. NestJS API
curl http://localhost:4000
```

---

## 1. 環境設定

### 複製環境變數

```bash
cd /Users/puppychen/Job/EcMap/zProjectsSource/oda-cyber-konsult
cp .env.example .env
```

### 確認關鍵配置

```bash
# .env
DATABASE_URL="postgresql://postgres:postgres@localhost:5432/oda_cyber?schema=public"
FASTAPI_BASE_URL=http://localhost:8000
```

---

## 2. 資料庫遷移

確保 Conversation 與 Message 表已建立：

```bash
cd apps/api

# 查看 Prisma Schema
cat prisma/schema.prisma | grep -A 10 "model Conversation"

# 執行遷移（若尚未執行）
pnpm prisma:migrate

# 查看資料庫狀態
pnpm prisma studio
```

---

## 3. 啟動 API 服務

```bash
cd apps/api
pnpm dev
```

**預期輸出**：

```
[Nest] INFO [Bootstrap] Application running on port 4000
```

---

## 4. 取得 JWT Token

### 註冊新使用者

```bash
curl -X POST http://localhost:4000/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "email": "test@example.com",
    "password": "password123",
    "name": "Test User"
  }' | jq
```

### 登入取得 Token

```bash
TOKEN=$(curl -X POST http://localhost:4000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "email": "test@example.com",
    "password": "password123"
  }' | jq -r '.data.token')

echo "Token: $TOKEN"
```

---

## 5. 測試 Chat API

### 5.1 發送第一則訊息（自動建立對話）

```bash
curl -X POST http://localhost:4000/api/chat \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "question": "什麼是 PDPA？",
    "topK": 5,
    "hybrid": true
  }' | jq
```

**預期回應**：

```json
{
  "success": true,
  "data": {
    "message": {
      "id": "uuid-1",
      "role": "user",
      "content": "什麼是 PDPA？"
    },
    "answer": {
      "id": "uuid-2",
      "role": "assistant",
      "content": "PDPA 全名為 Personal Data Protection Act...",
      "sources": [
        {
          "source": "pdpa-guide.pdf",
          "content_preview": "個人資料保護法...",
          "score": 0.92
        }
      ]
    },
    "conversationId": "conv-uuid"
  },
  "timestamp": "2026-02-09T14:30:00.000Z"
}
```

### 5.2 繼續對話（使用 conversationId）

```bash
# 將上一步的 conversationId 存為變數
CONV_ID="conv-uuid"

curl -X POST http://localhost:4000/api/chat \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{
    \"question\": \"請舉例說明 PDPA 的適用範圍\",
    \"conversationId\": \"$CONV_ID\",
    \"topK\": 5
  }" | jq
```

### 5.3 列出所有對話

```bash
curl -X GET "http://localhost:4000/api/chat/conversations?limit=10&offset=0" \
  -H "Authorization: Bearer $TOKEN" | jq
```

**預期回應**：

```json
{
  "success": true,
  "data": {
    "conversations": [
      {
        "id": "conv-uuid",
        "userId": "user-uuid",
        "title": "什麼是 PDPA？",
        "createdAt": "2026-02-09T10:00:00.000Z",
        "updatedAt": "2026-02-09T10:05:00.000Z",
        "messages": [
          {
            "id": "msg-uuid",
            "role": "assistant",
            "content": "請舉例說明...",
            "createdAt": "2026-02-09T10:05:00.000Z"
          }
        ]
      }
    ],
    "total": 1
  },
  "timestamp": "2026-02-09T14:30:00.000Z"
}
```

### 5.4 取得完整對話歷史

```bash
curl -X GET "http://localhost:4000/api/chat/conversations/$CONV_ID" \
  -H "Authorization: Bearer $TOKEN" | jq
```

### 5.5 刪除對話

```bash
curl -X DELETE "http://localhost:4000/api/chat/conversations/$CONV_ID" \
  -H "Authorization: Bearer $TOKEN" | jq
```

---

## 6. 測試串流模式

### 6.1 使用 cURL（基礎測試）

```bash
curl -X POST http://localhost:4000/api/chat/stream \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "question": "請詳細說明資料去識別化流程",
    "topK": 5
  }' \
  --no-buffer
```

**預期輸出（SSE 格式）**：

```
data: {"type":"chunk","content":"資料"}

data: {"type":"chunk","content":"去識別化"}

data: {"type":"chunk","content":"流程"}

data: {"type":"done","messageId":"msg-uuid","conversationId":"conv-uuid"}
```

### 6.2 使用 JavaScript (前端整合)

```javascript
async function streamChat(token, question) {
  const response = await fetch('http://localhost:4000/api/chat/stream', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({ question, topK: 5 }),
  });

  const conversationId = response.headers.get('X-Conversation-Id');
  console.log('Conversation ID:', conversationId);

  const reader = response.body.getReader();
  const decoder = new TextDecoder();
  let buffer = '';

  while (true) {
    const { done, value } = await reader.read();
    if (done) break;

    buffer += decoder.decode(value, { stream: true });
    const lines = buffer.split('\n\n');
    buffer = lines.pop(); // 保留未完整的行

    for (const line of lines) {
      if (line.startsWith('data: ')) {
        const data = JSON.parse(line.slice(6));

        if (data.type === 'chunk') {
          process.stdout.write(data.content); // 即時顯示
        } else if (data.type === 'done') {
          console.log('\n✅ Done:', data.messageId);
        } else if (data.type === 'error') {
          console.error('\n❌ Error:', data.message);
        }
      }
    }
  }
}

// 使用範例
streamChat('YOUR_JWT_TOKEN', '請詳細說明資料去識別化流程');
```

---

## 7. 錯誤排查

### 問題 1：401 Unauthorized

**原因**：JWT Token 無效或過期

**解決方案**：

```bash
# 重新登入取得新 Token
TOKEN=$(curl -X POST http://localhost:4000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test@example.com","password":"password123"}' \
  | jq -r '.data.token')
```

### 問題 2：404 Conversation not found

**原因**：conversationId 不存在或已被刪除

**解決方案**：

```bash
# 檢查對話是否存在
curl -X GET "http://localhost:4000/api/chat/conversations" \
  -H "Authorization: Bearer $TOKEN" | jq '.data.conversations[].id'
```

### 問題 3：403 Forbidden

**原因**：嘗試存取他人的對話

**解決方案**：確保使用正確的使用者 Token

### 問題 4：RAG 服務連線失敗

**錯誤訊息**：

```json
{
  "statusCode": 500,
  "message": "Internal server error"
}
```

**檢查步驟**：

```bash
# 1. 確認 RAG 服務運行中
curl http://localhost:8000/health

# 2. 檢查環境變數
cat .env | grep FASTAPI_BASE_URL

# 3. 查看 NestJS 日誌
# 應顯示 RAG 請求細節
```

### 問題 5：串流模式無回應

**原因**：前端未正確處理 SSE

**解決方案**：

- 使用 Fetch API 而非 EventSource（EventSource 不支援自訂 Header）
- 確保使用 `--no-buffer`（cURL）或 `stream: true`（Fetch）
- 檢查 `Content-Type: text/event-stream` Header

---

## 8. 開發提示

### 8.1 查看即時日誌

```bash
cd apps/api
pnpm dev
```

**關鍵日誌**：

```
[ChatController] POST /api/chat
[ChatService] Creating conversation for user: user-uuid
[RagProxyService] Querying RAG: http://localhost:8000/api/v1/rag/query
```

### 8.2 使用 Prisma Studio 檢視資料

```bash
cd apps/api
pnpm prisma studio
```

瀏覽器開啟 `http://localhost:5555`，可視化檢視 Conversation 與 Message 表。

### 8.3 清除測試資料

```bash
# 進入 PostgreSQL
docker exec -it postgres-container psql -U postgres -d oda_cyber

# 刪除所有對話（Cascade 刪除訊息）
DELETE FROM conversations WHERE user_id = 'your-test-user-uuid';

# 或重設整個表
TRUNCATE conversations, messages RESTART IDENTITY CASCADE;
```

---

## 9. 整合前端

### React 範例（標準模式）

```typescript
import { useState } from 'react';

function ChatComponent({ token }: { token: string }) {
  const [question, setQuestion] = useState('');
  const [answer, setAnswer] = useState('');
  const [conversationId, setConversationId] = useState<string | null>(null);

  const sendMessage = async () => {
    const response = await fetch('http://localhost:4000/api/chat', {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        question,
        conversationId,
        topK: 5,
        hybrid: true,
      }),
    });

    const result = await response.json();
    setAnswer(result.data.answer.content);
    setConversationId(result.data.conversationId);
  };

  return (
    <div>
      <input value={question} onChange={(e) => setQuestion(e.target.value)} />
      <button onClick={sendMessage}>發送</button>
      <div>{answer}</div>
    </div>
  );
}
```

### React 範例（串流模式）

```typescript
import { useState } from 'react';

function StreamChatComponent({ token }: { token: string }) {
  const [question, setQuestion] = useState('');
  const [answer, setAnswer] = useState('');
  const [isStreaming, setIsStreaming] = useState(false);

  const streamMessage = async () => {
    setIsStreaming(true);
    setAnswer('');

    const response = await fetch('http://localhost:4000/api/chat/stream', {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({ question, topK: 5 }),
    });

    const reader = response.body!.getReader();
    const decoder = new TextDecoder();
    let buffer = '';

    while (true) {
      const { done, value } = await reader.read();
      if (done) break;

      buffer += decoder.decode(value, { stream: true });
      const lines = buffer.split('\n\n');
      buffer = lines.pop()!;

      for (const line of lines) {
        if (line.startsWith('data: ')) {
          const data = JSON.parse(line.slice(6));
          if (data.type === 'chunk') {
            setAnswer((prev) => prev + data.content);
          } else if (data.type === 'done') {
            setIsStreaming(false);
          }
        }
      }
    }
  };

  return (
    <div>
      <input value={question} onChange={(e) => setQuestion(e.target.value)} />
      <button onClick={streamMessage} disabled={isStreaming}>
        {isStreaming ? '生成中...' : '發送'}
      </button>
      <div style={{ whiteSpace: 'pre-wrap' }}>{answer}</div>
    </div>
  );
}
```

---

## 10. 下一步

1. **探索完整 API 文件**：[docs/api/chat-api.md](/docs/api/chat-api.md)
2. **檢視實作細節**：[docs/implementation/chat-module-summary.md](/docs/implementation/chat-module-summary.md)
3. **整合 Chatbot UI**：參考前端專案 `apps/chatbot`
4. **開發單元測試**：使用 Jest + NestJS Testing

---

## 常見問題

### Q1：如何修改對話標題？

目前標題自動取問題前 50 字，若要手動修改：

```typescript
// ConversationRepository
async updateTitle(id: string, title: string) {
  return this.prisma.conversation.update({
    where: { id },
    data: { title },
  });
}
```

### Q2：如何實作訊息編輯？

參考實作摘要文件的「後續擴充建議」章節。

### Q3：串流模式是否支援取消？

可透過中斷 HTTP 連線實作，前端範例：

```javascript
const controller = new AbortController();
fetch(url, { signal: controller.signal });

// 取消請求
controller.abort();
```

### Q4：如何限制訊息長度？

在 DTO 中加入驗證：

```typescript
export class SendMessageDto {
  @IsString()
  @MaxLength(5000)
  question: string;
}
```

---

## 相關資源

- [NestJS 官方文件](https://docs.nestjs.com/)
- [Prisma 官方文件](https://www.prisma.io/docs)
- [Server-Sent Events 規範](https://html.spec.whatwg.org/multipage/server-sent-events.html)
- [RAG System Blueprint](/docs/RAG_SYSTEM_BLUEPRINT.md)
