---
audience: ai-primary
---

# cleaning-maker-checker — Tasks

> 此檔記錄 cleaning-maker-checker **功能級的待辦/已完成清單**，不是完整 sprint backlog（後者見 CLAUDE_TASK.md）。
> 只記與 spec pack 三件套同步的工作項。

---

## ✅ 已完成

### 4/18 series — Round 7 Maker-Checker 全項修復

- [x] **UUID vs str 比較統一**（commit Round 7 系列）
  - TYPE-COMPARE-01、KD-5
  - 影響端點：approve_task、reject_task、ingest_task 3 處
  - 修正：`str(body.approver_id) == str(task.submitted_by)`
- [x] **submit_for_review 加 SELECT FOR UPDATE row lock**
  - ROW-LOCK-01、KD-3
  - 防雙擊送審 race condition
- [x] **update_file_content 補 audit log**
  - AUDIT-01、KD-4
  - 之前漏寫，狀態改變不可追溯
- [x] **RejectTaskRequest empty note Pydantic validator**
  - AC-18-11-02
  - `rejection_reason: str = Field(..., min_length=1)`，空白 422
- [x] **15+ `inspect.getsource` 偽測試識別**（待 Q2 替換）
  - PSEUDO-TEST-01
  - `test_review_api.py::TestReviewIngestMakerChecker` 已標記，待替換為 httpx + ASGITransport 真實 HTTP test

### 4/24 series — Service-layer IDOR 盲區教訓延伸

- [x] **ACTOR-INJECTION-01 規則內化** （非 commit，是規則沉澱）
  - 雖然 chat-rag IDOR 修復不直接影響 cleaning，但「actor 偽造」同類風險記入此 spec pack 規則
  - `domain-code-review` skill IDOR 4-checklist 涵蓋本場景

### 4/29 series — SDD contracts SSoT 配套（間接受益）

- [x] **role 字面陣列改 USER_ROLES enum**（commit `2f5193a`）
  - 對 cleaning-maker-checker 的影響：`@Roles('data_cleaner', 'data_reviewer', 'admin')` 來源統一，DTO 用 `USER_ROLES`
  - 不直接修 cleaning 模組，但角色源頭已 SSoT

### 5/2 series — 本 batch（spec pack 試點）

- [x] **D 第 2 試點 spec pack 建立**：
  - `docs/04-features/cleaning-maker-checker/{README,requirements,design,tasks}.md` 四件套
  - 守 D0 紅線（指針 + 功能特殊規則 7 條 + 踩坑歷史）
  - 修正版 D5 三題（dry-run + 立刻測試 + +7 天補強觀察期）
- [x] **RTM v1.6.1**：FR-18 區塊 Spec Pack 連結

---

## 🔄 待辦（試點驗收期 +7 天內可選做）

- [ ] **T-PSEUDO-01: 替換 15 處 inspect.getsource 偽測試為真實 HTTP integration test**
  - PSEUDO-TEST-01
  - 位置：`python/rag-service/tests/test_review_api.py::TestReviewIngestMakerChecker`
  - 動機：4/18 已用 1 處真實 HTTP test 揭露 UUID bug；剩餘 15 處仍是 `inspect.getsource` 字串斷言，是 future bug 的盲區
  - 動作：`pattern-anti-pseudo-test` skill 已自動觸發，但替換需要實際撰寫 ASGITransport scenarios
  - 規模：M（單檔但 15 個 test case 需逐一改寫）
  - 對應 RTM TC-05-006 / TC-05-007

- [ ] **T-RACE-01: approve / reject / ingest 補 SELECT FOR UPDATE**
  - ROW-LOCK-01（目前只 submit 有；approve/reject/ingest 也有 race window）
  - 位置：`python/rag-service/src/rag_service/api/v1/review.py:approve_task / reject_task / ingest_task`
  - 動機：reviewer 雙擊批准、或 reviewer + admin 同時批准的 race condition
  - 規模：S（單檔三函式各加 `with_for_update()`）
  - 風險：低，但需確認 lock 順序避免 deadlock（all paths 鎖 task → file 順序）

- [ ] **T-IDOR-INTG-01: cleaning-maker-checker e2e IDOR 整合測試**
  - ACTOR-INJECTION-01
  - 規格：「user A 嘗試 POST /api/cleaning/review/{B 提交的 task}/approve → 應 403（Maker-Checker），且 actor 來自 JWT 而非 body」
  - 位置：新建 `apps/api/test/cleaning-review.e2e-spec.ts` 或加入 `app.e2e-spec.ts`
  - 規模：S
  - 對應 chat-rag 的 T-NEG-01 同類補強

- [ ] **T-LESSONS-01: CLAUDE_LESSONS.md 補 spec pack 第 2 試點教訓**
  - 試點期結束時記錄
  - 寫在 D5 驗收完成時

---

## 🔮 未來（v2 規劃）

- [ ] **複數審核者投票機制**：N 個 reviewer 全部批准才放行（PM 確認需求後啟用）
- [ ] **跨任務批次操作**：batch approve / reject endpoint（資料治理重構後啟用）
- [ ] **Approval workflow 自訂**：使用者可定義多步驟流程（高度客製化需求才啟用）
- [ ] **Audit log 即時通知**：寫入 cleaning_audit_logs 後推送 cleaner（websocket / email）
- [ ] **Submit 後自動分配 reviewer**：依工作量 / 領域自動指派（運營流程成熟後啟用）

---

## Spec Pack 維護紀律

> 此 tasks.md 不是專案 sprint backlog。其更新時機：

1. **每次本功能 commit**（含 fix / feature）→ ✅ 已完成段加一行
2. **規格變更**（PRD AC 改、新增 cleaning_audit_logs 欄位）→ 同步檢視 ✅/🔄/🔮 三段
3. **試點驗收結果寫回**（D5 +7 天 = 2026-05-09）→ 加 「## 驗收結果」段；若 ❌ 整個 spec pack 撤除（連同 RTM Spec Pack 欄位 + CLAUDE_LESSONS 補失敗教訓）

> **D0 紅線提醒**：requirements.md / design.md / tasks.md 三檔不可有 PRD/SRS/RTM 既有事實的複製。每次更新前自問：「這資訊在 PRD/SRS/ARCH/RTM/CLAUDE_LESSONS 有了嗎？有→指針，沒有→寫」。
