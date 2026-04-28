# chat-rag — Tasks

> 此檔記錄 chat-rag **功能級的待辦/已完成清單**，不是完整 sprint backlog（後者見 CLAUDE_TASK.md）。
> 只記與 spec pack 三件套同步的工作項。

---

## ✅ 已完成（2026-04 系列）

### 4/13 ~ 4/18 series — Wave 1-5 + 機制修正

- [x] **Expert mode maxTokens 8192 → 4096** (commit `fd20f53`)
  - AC-16-02-03、`chat.service.ts:30 MODE_CONFIG.expert.maxTokens`
  - 規格漂移修復；`chat.service.spec.ts` 斷言已對齊
- [x] **History injection 2 輪 → 5 輪** (commit `fd20f53`)
  - AC-02-04-01、`query-preprocessor.service.ts`
  - `slice(-4)` → `slice(-10)`（每輪 2 元素 = 5 user + 5 assistant）
- [x] **changePassword tokenVersion 遞增** (commit `fd20f53`)
  - AC-01-01-05（A2 任務新增規格條文）
  - 影響 chat-rag：所有 chat 操作 JWT 驗證 tokenVersion 對齊 DB

### 4/22 series — Codex 稽核修復

- [x] **`{user_name}` / `{query}` / `{context}` 三變數注入** (commit `be6a1c4`)
  - AC-03-01-03、新增 `prompt-renderer.util.ts` 共用 util
  - `chat.service.ts` 透過 `users.getDisplayName` 取得 user_name
- [x] **三層 topK 差異化（前端不覆蓋後端）** (commit `be6a1c4`)
  - AC-16-02-01~03、`useChat.ts` 不再固定送 topK=5
  - 後端 `chat.service.ts:dto.topK || modeConfig.topK` mode-aware fallback 生效

### 4/24 series — Codex 獨立審查

- [x] **prepareStreamContext IDOR 修復** (commit `6ccc26a`)
  - SEC-IDOR-01、KD-2
  - `chat.service.ts:prepareStreamContext` 加 `conversation.userId === userId` 檢查

### 4/29 series — 本 batch（plan 執行）

- [x] **A1 文件對齊**：cross-validation §1.2/§3.2 校正撤回 + PRD line 583 閾值語意交叉註腳
  - SEM-THRESHOLD-01 規則來源
- [x] **A2 AC-01-01-05 tokenVersion 規格條文** 新增
- [x] **A3 RTM v1.5.1** 升版
- [x] **B1-B7 contracts/ SDD 局部植入**：
  - `contracts/thresholds.yml` (7 RAG 閾值)
  - `contracts/enums.yml` (User 7 角色)
  - `packages/contracts/` workspace 建立
  - `chat.service.ts` 7 處 magic number 替換為 `RAG_THRESHOLDS.*`
  - `create-user.dto.ts` 用 `USER_ROLES`
  - `chat.service.contract.spec.ts` anti-drift 行為測試（5/5 綠）
  - `python/shared/contracts.py` + parity test（7/7 綠）
- [x] **C1-C5 ai-native-sdlc skill QG-3 補強**：
  - 三道防線文件 `qg3-defense-lines.md`
  - `pattern-anti-pseudo-test` QG-3 自動載入觸發
  - `domain-code-review` IDOR 4-checklist
- [x] **D1-D2 spec pack 試點**：本三件套 + RTM v1.6.0 FR-02 區塊 Spec Pack 連結（試點期不擴 RTM 表格）
- [x] **D4 spec-pack lint 補強**：新增 `pnpm lint:spec-pack` + `--self-test`
  - 檢查 broken links、Tier 1 長段落複製、RTM 連結、三件套完整性、D5 提前擴散
  - 壞案例演練已覆蓋 broken link / PRD copy / missing RTM / early promotion 四類錯誤

---

## 🔄 待辦（試點驗收期 +7 天內可選做）

- [ ] **T-NEG-01: 補 IDOR negative integration test**
  - 規格：「user A 嘗試 GET /api/chat/conversations/{B 的對話 id} → 應 403」
  - 位置：`apps/api/test/app.e2e-spec.ts` 或新建 `chat.e2e-spec.ts`
  - 動機：unit test 已驗證 service 層 ownership 檢查；e2e 補完整 HTTP 行為驗證（KD-2 強化）
  - 對應 `domain-code-review` IDOR Check 4

- [ ] **T-PERF-01: 對 chat-rag 跑 k6 壓測**（Wave 7 範圍，本批次外）
  - AC-02-01-02 P95 < 10s 未驗證
  - 走獨立 NFR baseline plan，不在 chat-rag spec pack 自動範圍

- [ ] **T-LESSONS-01: CLAUDE_LESSONS.md 補本 batch 教訓**（plan 收尾任務）
  - 寫在 batch 收尾時統一處理

---

## 🔮 未來（v2 規劃）

- [ ] **多語言查詢自動翻譯**（Out of Scope 變更時啟用）
- [ ] **Pure vector mode 信心度標籤**（FR-23 v2）
- [ ] **跨對話 context injection**（PM 確認需求後啟用）
- [ ] **Real-time RAG re-indexing**（資料治理重構後啟用）

---

## Spec Pack 維護紀律

> 此 tasks.md 不是專案 sprint backlog。其更新時機：

1. **每次本功能 commit**（含 fix / feature）→ ✅ 已完成段加一行
2. **規格變更**（PRD AC 改、contracts 改）→ 同步檢視 ✅/🔄/🔮 三段
3. **試點驗收結果寫回**（D5 +7 天後）→ 加 「## 驗收結果」段；若 ❌ 整個 spec pack 撤除（連同 RTM Spec Pack 欄位）

> **D0 紅線提醒**：requirements.md / design.md / tasks.md 三檔不可有 PRD/SRS/RTM 既有事實的複製。每次更新前自問：「這資訊在 PRD/SRS/ARCH/RTM/CLAUDE_LESSONS 有了嗎？有→指針，沒有→寫」。
