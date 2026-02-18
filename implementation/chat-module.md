# Chat Module 實作摘要

## 實作時間
2026-02-09

## 概述

已完成 NestJS Chat 模組的完整實作，包含對話管理、訊息持久化與 RAG 整合（支援標準與 SSE 串流模式）。

---

## 檔案結構

```
apps/api/src/modules/chat/
├── chat.module.ts                           # 模組定義
├── controllers/
│   └── chat.controller.ts                   # HTTP 端點處理
├── services/
│   ├── chat.service.ts                      # 業務邏輯層
│   └── rag-proxy.service.ts                 # RAG 服務代理
├── repositories/
│   ├── conversation.repository.ts           # Conversation 資料存取
│   └── message.repository.ts                # Message 資料存取
└── dto/
    └── chat.dto.ts                          # 請求驗證 DTO
```

---

## 分層架構

```
Controller (HTTP) → Service (業務邏輯) → Repository (資料存取) → Prisma → PostgreSQL
                                    ↓
                              RagProxyService → FastAPI (localhost:8000)
```

### 職責分離

| 層級 | 職責 | 範例 |
|------|------|------|
| **Controller** | HTTP 請求處理、回應格式化、JWT 認證 | 路由定義、@Body/@Query 解析 |
| **Service** | 業務邏輯、權限檢查、協調多個 Repository | 建立對話、驗證使用者權限、呼叫 RAG |
| **Repository** | 資料庫 CRUD、查詢優化 | Prisma 查詢封裝、關聯資料載入 |
| **RagProxyService** | RAG API 呼叫、串流處理 | HTTP 請求代理、SSE 串流轉發 |

---

## 核心功能

### 1. 對話管理

- **建立對話**：首次發送訊息時自動建立，標題取問題前 50 字
- **列出對話**：支援分頁，包含每個對話的最後一則訊息
- **取得對話**：載入完整訊息歷史（升序排列）
- **刪除對話**：Cascade 刪除所有關聯訊息

### 2. 訊息發送

#### 標準模式 (POST /api/chat)

```typescript
// Request
{
  question: string;
  conversationId?: string;
  topK?: number;      // 1-20，預設 5
  hybrid?: boolean;   // 預設 true
}

// Response
{
  message: { id, role: 'user', content },
  answer: { id, role: 'assistant', content, sources },
  conversationId
}
```

#### 串流模式 (POST /api/chat/stream)

- 使用 Server-Sent Events (SSE)
- 逐字回傳 RAG 生成內容
- Header 包含 `X-Conversation-Id`
- 事件類型：`chunk`（部分內容）、`done`（完成）、`error`（錯誤）

### 3. RAG 整合

**RagProxyService** 負責與 FastAPI RAG 服務通訊：

- **標準模式**：使用 `@nestjs/axios` + `firstValueFrom`
- **串流模式**：使用 `responseType: 'stream'` + AsyncGenerator
- **超時設定**：120 秒
- **環境變數**：`FASTAPI_BASE_URL`（預設 `http://localhost:8000`）

---

## 資料模型

### Conversation (Prisma)

```prisma
model Conversation {
  id        String    @id @default(uuid())
  userId    String    @map("user_id")
  title     String?
  createdAt DateTime  @default(now()) @map("created_at")
  updatedAt DateTime  @updatedAt @map("updated_at")
  user      User      @relation(fields: [userId], references: [id], onDelete: Cascade)
  messages  Message[]

  @@map("conversations")
}
```

### Message (Prisma)

```prisma
model Message {
  id             String       @id @default(uuid())
  conversationId String       @map("conversation_id")
  role           String       // 'user' | 'assistant'
  content        String
  sources        Json?        // RAG 來源文件
  metadata       Json?        // Token 使用量等
  createdAt      DateTime     @default(now()) @map("created_at")
  conversation   Conversation @relation(fields: [conversationId], references: [id], onDelete: Cascade)

  @@map("messages")
}
```

---

## API 端點

| 方法 | 路徑 | 說明 | 認證 |
|------|------|------|------|
| POST | `/api/chat` | 發送訊息（標準模式） | JWT ✅ |
| POST | `/api/chat/stream` | 發送訊息（串流模式） | JWT ✅ |
| GET | `/api/chat/conversations` | 列出對話 | JWT ✅ |
| GET | `/api/chat/conversations/:id` | 取得單一對話 | JWT ✅ |
| DELETE | `/api/chat/conversations/:id` | 刪除對話 | JWT ✅ |

詳細 API 文件：[docs/api/chat-api.md](/docs/api/chat-api.md)

---

## 安全性與權限

### JWT 認證

- 所有端點使用 `@UseGuards(JwtAuthGuard)`
- `@CurrentUser('id')` 裝飾器取得使用者 ID

### 權限檢查

- **橫向越權防護**：Service 層驗證 `conversation.userId === currentUserId`
- **錯誤處理**：
  - `NotFoundException` (404)：對話不存在
  - `ForbiddenException` (403)：嘗試存取他人對話

---

## 依賴項

### NestJS 套件

- `@nestjs/common` - 核心框架
- `@nestjs/axios` - HTTP 客戶端
- `class-validator` - DTO 驗證
- `class-transformer` - 型別轉換

### 已安裝

所有必要依賴皆已在 `apps/api/package.json` 中定義，無需額外安裝。

---

## 配置需求

### 環境變數 (.env)

```bash
# NestJS API
FASTAPI_BASE_URL=http://localhost:8000   # RAG 服務位址

# Database
DATABASE_URL="postgresql://postgres:postgres@localhost:5432/oda_cyber?schema=public"
```

### 全域 ValidationPipe

已在 `main.ts` 配置：

```typescript
app.useGlobalPipes(
  new ValidationPipe({
    whitelist: true,
    forbidNonWhitelisted: true,
    transform: true,
  }),
);
```

### CORS

已在 `main.ts` 配置允許前端存取：

```typescript
app.enableCors({
  origin: ['http://localhost:5173', 'http://localhost:5174'],
  credentials: true,
});
```

---

## 測試建議

### 單元測試

1. **ConversationRepository**
   - 測試 CRUD 操作
   - 驗證 include 關聯正確載入

2. **MessageRepository**
   - 測試訊息建立
   - 驗證 JSON 欄位（sources、metadata）儲存

3. **ChatService**
   - Mock Repository 與 RagProxyService
   - 測試權限檢查邏輯
   - 測試自動建立對話流程

4. **RagProxyService**
   - Mock HttpService
   - 測試標準與串流模式
   - 測試錯誤處理

### 整合測試

1. **E2E 測試**
   - 完整對話流程（建立 → 發送訊息 → 查詢 → 刪除）
   - 測試跨使用者權限隔離
   - 測試 RAG 整合（需 Mock FastAPI）

### 手動測試

```bash
# 1. 啟動服務
cd apps/api
pnpm dev

# 2. 取得 JWT Token（透過 Auth API）
TOKEN=$(curl -X POST http://localhost:4000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test@example.com","password":"password"}' \
  | jq -r '.data.token')

# 3. 發送訊息
curl -X POST http://localhost:4000/api/chat \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"question":"什麼是 PDPA？","topK":5}' | jq

# 4. 列出對話
curl -X GET "http://localhost:4000/api/chat/conversations?limit=10" \
  -H "Authorization: Bearer $TOKEN" | jq

# 5. 串流測試（需支援 SSE 的客戶端）
curl -X POST http://localhost:4000/api/chat/stream \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"question":"請詳細說明資料去識別化流程"}' \
  --no-buffer
```

---

## 已知限制與注意事項

### 1. EventSource 限制

瀏覽器原生 `EventSource` API 不支援自訂 Header，前端需使用 Fetch API + ReadableStream 實作 SSE 接收。

**解決方案**：

```javascript
const response = await fetch('/api/chat/stream', {
  method: 'POST',
  headers: { 'Authorization': `Bearer ${token}` },
  body: JSON.stringify({ question: '...' }),
});

const reader = response.body.getReader();
const decoder = new TextDecoder();

while (true) {
  const { done, value } = await reader.read();
  if (done) break;
  // 處理 SSE 資料
}
```

### 2. RAG 服務依賴

- Chat API 完全依賴 FastAPI RAG 服務
- 若 RAG 服務離線，訊息發送會失敗
- 建議實作降級策略或錯誤通知機制

### 3. 串流模式錯誤處理

- 串流開始後無法修改 HTTP 狀態碼
- 錯誤透過 `data: {"type":"error"}` 事件回傳
- 前端需處理 `error` 事件類型

### 4. 對話標題

- 目前自動取問題前 50 字作為標題
- 未來可考慮使用 LLM 自動生成摘要標題

---

## 後續擴充建議

### 1. 訊息編輯與刪除

```typescript
// MessageRepository
async update(id: string, content: string) {
  return this.prisma.message.update({ where: { id }, data: { content } });
}

async delete(id: string) {
  return this.prisma.message.delete({ where: { id } });
}
```

### 2. 對話搜尋

```typescript
// ConversationRepository
async search(userId: string, keyword: string) {
  return this.prisma.conversation.findMany({
    where: {
      userId,
      OR: [
        { title: { contains: keyword } },
        { messages: { some: { content: { contains: keyword } } } },
      ],
    },
  });
}
```

### 3. 訊息評分與回饋

```prisma
model Message {
  // ... existing fields
  rating       Int?
  feedback     String?
}
```

### 4. 對話標籤系統

```prisma
model ConversationTag {
  id             String   @id @default(uuid())
  conversationId String
  tag            String
  conversation   Conversation @relation(fields: [conversationId], references: [id])

  @@unique([conversationId, tag])
}
```

### 5. 對話分享功能

```prisma
model SharedConversation {
  id             String   @id @default(uuid())
  conversationId String
  shareToken     String   @unique
  expiresAt      DateTime?
  conversation   Conversation @relation(fields: [conversationId], references: [id])
}
```

---

## 相關文件

- [Chat API 文件](/docs/api/chat-api.md)
- [RAG API 文件](/docs/api/rag-api.md)
- [Prisma Schema](/apps/api/prisma/schema.prisma)
- [NestJS 官方文件](https://docs.nestjs.com/)

---

## 變更歷史

| 日期 | 版本 | 變更內容 |
|------|------|----------|
| 2026-02-09 | 1.0.0 | 初始實作完成 |
