---
audience: ai-primary
---

# chat-rag — Design

> 守 D0 紅線：架構/技術細節若已在 ARCH.md / SRS-T，本檔只放指針；只記錄**功能級設計決策**與 trade-off。

---

## Architecture（指針）

宏觀架構見 [ARCH.md](../../01-specs/architecture/ARCH.md)：
- ADR-002 NestJS API Gateway → chat 端點集中於 `apps/api`
- ADR-004 Hybrid Search (Vector+BM25+RRF) → `python/rag-service`
- ADR-005 SSE 串流 → `chat.controller.ts:/stream`
- ADR-006 三前端應用 → chatbot 5502 為主要 UI

完整資料流見 [system-architecture.drawio](../../01-specs/architecture/system-architecture.drawio)。

### chat-rag 特定通訊鏈

```
Chatbot UI (5502)
  → POST /api/chat/stream (NestJS, JWT auth)
    → ChatService.prepareStreamContext (含 ownership 檢查)
      ├─ ConversationRepository.findById (讀取後由 service 層驗 ownership)
      ├─ MessageRepository.findHistory (limit=10 = 5 輪)
      └─ QueryPreprocessor.rewriteWithContext (代名詞改寫)
    → ChatService.sendMessage
      ├─ RagProxyService.retrieve (FastAPI :3502)
      │   └─ Hybrid Search → filterLowScoreResults
      ├─ enrichWithWebSearch (條件觸發 SearXNG)
      ├─ ContextBuilderService.buildPrompt (含 prompt template + RAG context + web context + history)
      ├─ LlmService.generate (Gemini/OpenAI)
      └─ determineConfidenceLevel
    → SSE stream tokens to client
    → MessageRepository.create (儲存完整對話)
```

---

## Key Design Decisions（chat-rag 特有）

### KD-1：閾值來源 — `@oda-cyber/contracts`

**Decision**：所有 RAG 閾值常數從 `contracts/thresholds.yml` 載入，禁止 hardcode。

**Why**：4/18 cross-validation 揭露 6 項數值漂移（含 Expert maxTokens、ZIP 限制、history 輪數），根因都是 PRD 數值由開發者人工複製到代碼。SDD contracts SSoT（plan B 任務）解決此根因。

**How**：
- `chat.service.ts` 從 `@oda-cyber/contracts` import `RAG_THRESHOLDS`
- 編譯期：TypeScript 型別檢查 key 存在
- 啟動期：`packages/contracts/src/index.ts` 用 Zod 驗證 schema
- 測試期：`chat.service.contract.spec.ts` 用 jest.mock 注入極端值驗證行為依賴 contracts

**Trade-off**：runtime 載入 vs 編譯期常數
- 選 runtime（yaml file）：跨語言（TS+Python）共享、變更不需重新 build
- 棄編譯期（TS const）：失去跨語言能力，但 IDE 支持更佳
- 結論：因 Python 端有未來 wire-up 潛力（plan §B5 stub），選 runtime

### KD-2：IDOR 防線 — Service 層強制 ownership 驗證

**Decision**：每個接受 `conversationId` / `messageId` 的 service 方法**必驗** `resource.userId === currentUser.id`（除 admin）。Guard 不足。

**Why**：4/24 Codex 獨立審查發現 `prepareStreamContext` 漏驗。CLAUDE_LESSONS 指出：Guard 層 (`@UseGuards(JwtAuthGuard)`) 只證明「已登入」，不證明「擁有此資源」。功能導向 gap-analysis 看不到攻擊面。

**How**：
- 統一 pattern：service 方法第一步 `findById` → 第二步檢查 `userId === currentUser.id` → 第三步操作
- repository 端 `findById` 不主動加 user_id filter（保持 read-only repository pattern），由 service 層負責 authorization
- 對應 `domain-code-review` skill IDOR 4-checklist（C 任務 line 33 強化）

**Trade-off**：service 層檢查 vs repository 層 filter
- 選 service 層：authorization 邏輯集中、可記錄拒絕事件、可區分 NotFound vs Forbidden
- 棄 repo 層 filter：repo 層需注入 currentUser context，違反 layered 架構

### KD-3：兩層閾值語意設計

**Decision**：filter 閾值與 confidence 閾值是兩層獨立常數，不可合併。

**Why**：產品語意需求：
1. 過濾 = 「這結果分數太低，使用者不該看到」
2. 信心度判定 = 「這結果留下了（含 fallback），但要告訴使用者可信度等級」

兩個任務不同：過濾追求 precision（避免 noise），信心度判定追求 transparency（讓使用者自行判斷）。混為一談會丟失 fallback 機制（過濾後保留 top 1）。

**How**：
- `chat.service.ts:filterLowScoreResults` 用 `*_filter_min`
- `chat.service.ts:determineConfidenceLevel` 用 `*_{high,low}_confidence`
- contracts/thresholds.yml 開頭註解明列兩層差異
- 行為驗證：`chat.service.contract.spec.ts` 同時測 filter 與 confidence 路徑

**Trade-off**：簡化單層 vs 保留兩層
- 棄單層：丟失 fallback「保留 top 1」設計，UX 退化
- 選兩層：規格較複雜，但 PRD line 127-131 已語意說明

### KD-4：SSE 串流（ADR-005 細化）

**Decision**：chat 回應用 SSE（Server-Sent Events）逐 token 串流，而非 WebSocket 或 long polling。

**Why**：
- LLM call 平均延遲 3-8 秒，使用者首字元響應 (TTFT) 體驗關鍵
- SSE 單向 server→client 符合 chat 場景（client 不需中途插話）
- HTTP/1.1 原生支援，proxy 友善（WebSocket 在企業 proxy 常被擋）

**Trade-off**：SSE vs WebSocket
- 選 SSE：簡單、防火牆友善、原生 HTTP
- 棄 WebSocket：本案無雙向實時需求；clean app 任務進度走獨立 WS gateway 是另一場景

### KD-5：History injection 5 輪（AC-02-04-01）

**Decision**：對話歷史注入最近 5 輪（10 元素：5 user + 5 assistant）。

**Why**：
- 4/18 修復前：`slice(-4)` 等於 2 輪（每輪 2 元素），不足以理解多步追問
- 5 輪是「夠用就好」的成本/品質平衡：
  - 每多一輪 +200 tokens × LLM 千次/天 = 顯著成本
  - 5 輪足以覆蓋 90% 追問場景（PRD AC-02-04-02 代名詞改寫補足剩餘 10%）

**How**：`MODE_CONFIG.{mode}.maxHistoryTurns`，beginner=3, standard=5, expert=8

**Trade-off**：固定 5 vs 動態（依 token budget）
- 棄動態：複雜度高，需估算 token，且不同 LLM tokenizer 不同
- 選固定：簡單，由 mode 差異化處理

---

## Known Pitfalls（指針）

| 坑 | 位置 | 防護 |
|---|------|------|
| 4/18 Expert maxTokens 8096 vs 4096 漂移 | RTM v1.4.0 ③ | KD-1 contracts SSoT |
| 4/18 history `slice(-4)` 2 輪 vs 5 輪 | RTM v1.4.0 ③ | MODE_CONFIG SSoT，新增 mode 必對齊 PRD |
| 4/22 prompt 變數替換 `{{key}}` vs `{key}` 雙軌 | CLAUDE_LESSONS「規格雙軌漂移」 | 共用 `prompt-renderer.util.ts`，新增變數**必過此 util** |
| 4/24 prepareStreamContext IDOR | CLAUDE_LESSONS「Service-layer IDOR 盲區」 | KD-2 service 層強制 ownership |
| 4/18 Maker-Checker UUID 比較 | CLAUDE_LESSONS「型別比較陷阱」 | 雖在 cleaning，但 chat conversation 比較統一用 `===`（TS 嚴格相等） |

---

## 跨功能依賴

| 依賴方向 | 對象 | 介面 |
|---------|------|------|
| chat-rag → contracts | `@oda-cyber/contracts` | RAG_THRESHOLDS |
| chat-rag → llm | `LlmService` | generate(messages, options) |
| chat-rag → rag-service | `RagProxyService` → FastAPI :3502 | POST /api/v1/rag/retrieve |
| chat-rag → websearch | `SearXNGService` + `WebFetcherService` | conditional fallback |
| chat-rag → prompts | `PromptsService` | getPromptTemplate(role, mode) |
| chat-rag → users | `UsersService` | getDisplayName (4/22 變數注入修復) |
| ← 被 chatbot UI 呼叫 | apps/chatbot | SSE stream consumer |

未來變更若涉跨服務 contract，需同步更新 [api/chat-api.md](../../01-specs/api/chat-api.md) 與本 design.md KD 段。
