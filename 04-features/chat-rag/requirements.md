---
audience: ai-primary
---

# chat-rag — Requirements

> **Spec Pack 試點功能**（plan §D 唯一試點）
> 守 D0 紅線：只放**指針 + 功能特殊規則 + 踩坑歷史**，禁止複製 PRD/SRS 內容。
> 試點驗收：建立後 +7 天評估（plan §D5）。

---

## User Stories（指針）

完整 US/AC 定義見 [PRD.md §Epic 2 RAG 智慧問答](../../01-specs/PRD.md#epic-2)（line 76-131）：

| US | 指向 PRD 行號 | 範圍 |
|----|-------------|------|
| US-02-01 | line 78-88 | 自然語言查詢 + SSE 串流 |
| US-02-02 | line 90-96 | 引用來源顯示 |
| US-02-03 | line 98-106 | 歷史對話管理 |
| US-02-04 | line 108-115 | 多輪對話 + Query 改寫 |
| US-02-05 | line 117-126 | 分數過濾與品質保障（**閾值雙語意核心**）|

相關 Epic 指針：
- Epic 10 三層回應模式（FR-16）：[PRD line 313-354](../../01-specs/PRD.md#epic-10)
- Epic 16 回答品質回饋（FR-22）：[PRD line 556-568](../../01-specs/PRD.md#epic-16)
- Epic 17 RAG 信心度指示器（FR-23）：[PRD line 571-585](../../01-specs/PRD.md#epic-17)

---

## Acceptance Criteria（指針 + contracts 引用）

完整 AC 條列見 PRD 對應段落。**本檔只記錄與 contracts/* 連結的關鍵 AC**：

| AC | PRD 位置 | 規格值來源 | 實作位置 |
|----|---------|----------|---------|
| AC-02-05-01 過濾閾值 RRF | [PRD:124](../../01-specs/PRD.md) | `contracts/thresholds.yml#rag.hybrid_filter_min` (0.005) | `chat.service.ts:326` filterLowScoreResults |
| AC-02-05-02 過濾閾值 cosine | [PRD:125](../../01-specs/PRD.md) | `contracts/thresholds.yml#rag.cosine_filter_min` (0.3) | `chat.service.ts:326` 同上 |
| AC-23-01-02 高信心度 RRF | [PRD:581](../../01-specs/PRD.md) | `contracts/thresholds.yml#rag.rrf_high_confidence` (0.01) | `chat.service.ts:385` determineConfidenceLevel |
| AC-23-01-02 高信心度 cosine | [PRD:581](../../01-specs/PRD.md) | `contracts/thresholds.yml#rag.cosine_high_confidence` (0.7) | 同上 |
| AC-23-01-04 低信心度 RRF | [PRD:583](../../01-specs/PRD.md) | `contracts/thresholds.yml#rag.rrf_low_confidence` (0.003) | `chat.service.ts:386` 同上 |
| AC-23-01-04 低信心度 cosine | [PRD:583](../../01-specs/PRD.md) | `contracts/thresholds.yml#rag.cosine_low_confidence` (0.3) | 同上 |
| AC-16-02-01~03 三模式參數 | [PRD:331-334](../../01-specs/PRD.md) | `chat.service.ts:18-31 MODE_CONFIG` (已 SSoT) | 同 |
| AC-02-04-01 history 5 輪 | [PRD:114](../../01-specs/PRD.md) | `MODE_CONFIG.{mode}.maxHistoryTurns` | `query-preprocessor.service.ts` |

---

## ⚠ 此功能特殊規則（PRD/SRS 沒寫但開發必知）

### SEC-IDOR-01：Service 層 conversation/message ownership 強制驗證

**規則**：所有接受 `conversationId` / `messageId` 的 service 方法**必須**在操作前驗證 `resource.userId === currentUser.id`（除 admin）。Guard 層 `@UseGuards(JwtAuthGuard)` 不足——只證明「已登入」，不證明「擁有此資源」。

**實作位置**：
- `chat.service.ts:getConversation` ✅ 有檢查
- `chat.service.ts:deleteConversation` ✅ 有檢查
- `chat.service.ts:prepareStreamContext` ✅ 4/24 修復後有檢查（commit `6ccc26a`）
- `chat.service.ts:submitFeedback` ✅ 有檢查（line 371-372）
- `chat.service.ts:sendMessage` ✅ 有檢查
- 未來新增 chat conversation 相關 service 方法 → **必照此規則**

**根因（為什麼此規則不在 PRD）**：4/24 Codex 獨立審查發現 `prepareStreamContext` 在「繼續既有對話」路徑漏驗 ownership。CLAUDE_LESSONS「Service-layer IDOR 盲區（2026-04-24）」第二段指出：gap-analysis 是功能導向（「用戶能聊天」✅），不是攻擊面導向（「用戶能存取他人聊天」❌）。功能級規格有此空白。

### SEM-THRESHOLD-01：過濾閾值 vs 信心度判定閾值是兩層獨立語意

**規則**：開發者更動 contracts/thresholds.yml 時**必須區分**兩層：
- `*_filter_min`：過濾移除（filterLowScoreResults）。過濾後若無結果保留 top 1 fallback。
- `*_high/low_confidence`：對 fallback 後留存結果分級為 🟢/🟡/🔴（determineConfidenceLevel）。

**邊界值範例**：RRF score = 0.004 的結果：
- 0.004 < 0.005 過濾閾值 → 被過濾
- 但若整批結果都 < 0.005 → fallback 保留 top 1（含此 0.004）→ 進入 confidence 判定 → 0.004 > 0.003 低信心下界 → 判 🟡 中信心而非 🔴 低信心
- 唯有過濾後 fallback 留下且 score < 0.003 才會判 🔴 低信心

**根因（為什麼此規則不在 PRD）**：4/18 cross-validation §1.2「矛盾 1」一度誤判 0.005 vs 0.003 為矛盾，PRD line 127-131「閾值語意說明」（4/18 補強）才釐清。但工程師讀 PRD 時很容易跳過 line 127-131 註解。

### CONTRACTS-01：閾值不可硬編碼

**規則**：chat.service 中**禁止**寫入 magic number（0.005、0.3、0.01、0.7、0.003、0.05 等 RAG 相關常數）。一律從 `@oda-cyber/contracts` 載入。

**驗證機制**：`chat.service.contract.spec.ts` 用 jest.mock 注入 0.999 極端值，驗證 service 真的從 yml 載入（非硬編碼）。若有人改回 magic number，此測試紅。

**根因**：4/18 揭露的 6 項數值漂移（Expert maxTokens、ZIP 限制、history 輪數等）根因都是「PRD 數值由開發者人工複製到代碼」。CLAUDE_LESSONS「規格漂移與驗證漏洞（2026-04-22）」第三段。

---

## 踩坑歷史（指針 + 1 句脈絡）

| 事件 | 日期 | 教訓位置 | 1 句脈絡 |
|------|------|---------|---------|
| Maker-Checker UUID bug | 2026-04-18 | CLAUDE_LESSONS「型別比較陷阱」 | 雖然不在 chat-rag，但相關「身份比較統一用 `str(a)==str(b)`」原則對 chat conversation/message ownership 比較同樣適用 |
| Expert maxTokens 8192→4096 漂移 | 2026-04-18 | RTM v1.4.0 ③ | maxTokens 在 MODE_CONFIG 已 SSoT，但若未來新增 mode 必確認對齊 PRD |
| query history 2→5 輪修復 | 2026-04-18 | RTM v1.4.0 ③ | `slice(-N)` 中 N 是元素數而非對話輪數，每輪 = 2 元素（user + assistant） |
| 變數替換雙軌漂移 | 2026-04-22 | CLAUDE_LESSONS「規格雙軌漂移」 | chat-rag 與 prompts 模組共用 `prompt-renderer.util.ts`，未來新增 prompt 變數**必使用此 util** 而非自行實作 |
| Service-layer IDOR | 2026-04-24 | CLAUDE_LESSONS「Service-layer IDOR 盲區」 | 已內化為 SEC-IDOR-01（見上）|

---

## ❌ Out of Scope（明確不做）

- **Pure vector mode 信心度**：FR-23 v1 只支援 hybrid（RRF）+ vector 兩種，不支援未來可能的 keyword-only 或其他混合模式。
- **跨對話 context injection**：history 限定本對話最近 5 輪（AC-02-04-01），不跨對話注入。
- **Real-time RAG re-indexing**：使用者上傳問題不會即時更新知識庫；仍需走 cleaner app 審核流程。
- **多語言查詢自動翻譯**：AC-02-01-01 支援中英文提問，但不提供「中文問題自動翻譯為英文檢索」的隱式翻譯邏輯。

---

## 試點驗收（D5）連結

本 spec pack 是 plan §D 唯一試點，建立後 +7 天驗收：
- Q1 對人審查面：是否更快？
- Q2 對 AI 協作：是否聚焦 context？
- Q3 紅線：無複製 PRD？

驗收 → CLAUDE_TASK.md「持續追蹤」條目。
