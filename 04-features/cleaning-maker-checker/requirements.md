---
audience: ai-primary
---

# cleaning-maker-checker — Requirements

> **Spec Pack 第 2 試點**（chat-rag 為第 1 試點）。
> 守 D0 紅線：只放**指針 + 功能特殊規則 + 踩坑歷史**，禁止複製 PRD/SRS 內容。
> 試點驗收：建立後 +7 天評估（修正版 D5：dry-run 立刻測 + 補強觀察期）。

---

## User Stories（指針）

完整 US/AC 定義見 [PRD.md §Epic 18 清洗審核管理 — Cleaner App](../../01-specs/PRD.md)（line 467-498）：

| US | 指向 PRD 行號 | 範圍 |
|----|-------------|------|
| US-18-10 | line 467-476 | 送審任務（cleaner 送審）|
| US-18-11 | line 478-487 | 退回任務（reviewer 退回，必填理由）|
| US-18-12 | line 489-498 | Maker-Checker 職責分離（伺服器端強制 submitted_by ≠ approved_by）|

相關（同 Epic 18 補完整流程）：
- US-18-05 任務批准/駁回（[PRD line 對應 AC-18-05](../../01-specs/PRD.md)）
- 配套：US-18-04 內容編輯、US-18-13 ingest（核准後送入 RAG）

---

## Acceptance Criteria（指針 + 實作位置）

完整 AC 條列見 PRD 對應段落。**本檔只記錄與 Maker-Checker 不變式直接相關的關鍵 AC**：

| AC | PRD 位置 | 實作位置 | Maker-Checker 不變式 |
|----|---------|---------|---------------------|
| AC-18-10-03 任務送審後狀態 `review_requested`、檔案凍結 | [PRD:475](../../01-specs/PRD.md) | `python/rag-service/src/rag_service/api/v1/review.py` submit_for_review | STATE-FLOW-01 |
| AC-18-10-04 記錄 submitted_by + submitted_at | [PRD:476](../../01-specs/PRD.md) | 同上；`Task.submitted_by` UUID 欄位 | AUDIT-01 |
| AC-18-11-02 退回必填 rejection_reason，空白 422 | [PRD:485](../../01-specs/PRD.md) | `RejectTaskRequest` Pydantic validator + review.py reject_task | — |
| AC-18-11-03 退回後狀態回 `pending`、檔案解凍 | [PRD:486](../../01-specs/PRD.md) | review.py reject_task | STATE-FLOW-01 |
| AC-18-12-01 伺服器端強制 `submitted_by ≠ approved_by`，403 | [PRD:495](../../01-specs/PRD.md) | review.py approve_task / reject_task / ingest_task（**不在 NestJS proxy 層**）| MC-INVARIANT-01 |
| AC-18-12-03 actor identity 由 JWT `@CurrentUser` 注入，不依賴 body | [PRD:497](../../01-specs/PRD.md) | `apps/api/src/modules/cleaning/controllers/review.controller.ts:139/172` 透過 X-Internal headers 傳遞 | ACTOR-INJECTION-01 |
| AC-18-12-04 完整稽核軌跡（submitted_by/at + approved_by/at + edited_by/at）| [PRD:498](../../01-specs/PRD.md) | `cleaning_audit_logs` 表（`python/rag-service/src/rag_service/db/models.py:130`）| AUDIT-01 |

---

## ⚠ 此功能特殊規則（PRD/SRS 沒寫但開發必知）

### TYPE-COMPARE-01：跨來源 ID 比較統一規則

**規則**：FastAPI 端任何身份比較（actor_id vs Task.submitted_by / approved_by / edited_by）**禁止**直接 `==`，**必須**統一型別後比較：
```python
# ✅ 正確
if str(body.approver_id) == str(task.submitted_by):
    raise HTTPException(403)

# ❌ 錯誤（Pydantic UUID vs SQLAlchemy str → 永遠 False）
if body.approver_id == task.submitted_by:
    raise HTTPException(403)
```

**根因（為什麼此規則不在 PRD）**：4/18 Round 7 揭露 — Pydantic 解析 request body UUID 為 `uuid.UUID` 物件；SQLAlchemy ORM 若欄位定義為 `String` 則返回 str。`UUID == str` 永遠 False，造成 Maker-Checker 強制驗證**形同虛設 3 個端點**（submit/approve/reject）。CLAUDE_LESSONS「型別比較陷阱（2026-04-18）」line 49-51 完整背景。

### MC-INVARIANT-01：Maker-Checker 不變式（4 端點聯動）

**規則**：以下端點**必驗** `submitted_by ≠ actor_id`（除 admin 角色覆蓋）：

| 端點 | 不變式 | 違反處置 |
|------|--------|---------|
| `POST /api/v1/review/{task_id}/approve` | actor ≠ submitted_by | 403 Forbidden |
| `POST /api/v1/review/{task_id}/reject` | actor ≠ submitted_by | 403 Forbidden |
| `POST /api/v1/review/{task_id}/ingest` | actor ≠ submitted_by ∧ actor ≠ approved_by | 403 Forbidden |
| `POST /api/v1/review/{task_id}/submit` | （submit 不需 ≠ 檢查，但需 row lock，見 ROW-LOCK-01） | — |

**根因**：4/18 Round 7 同時 3 端點壞（approve/reject/ingest），不能只修一個。新增任何狀態轉換端點時，必先決定其 Maker-Checker 不變式。

### ACTOR-INJECTION-01：Actor identity 必由 NestJS JWT 注入，FastAPI 禁從 body 接受 actor_id

**規則**：
- NestJS `review.controller.ts` 使用 `@CurrentUser() user` 從 JWT 取 actor_id
- 透過 X-Internal headers（如 `X-Actor-Id`、`X-Actor-Role`）傳給 FastAPI proxy
- FastAPI `review.py` 從 headers 讀 actor，**禁止**從 request body 讀 `actor_id` / `approver_id`
- 配合 `X-Internal-Token` 確認來源是 NestJS 而非外部偽造

**根因**：若 actor 從 body 來，惡意客戶端可送 `{ "approver_id": "victim-uuid" }` 偽造他人身份。Guard 層只驗 JWT 已登入，不驗 body 內容真實性。CLAUDE_LESSONS「Service-layer IDOR 盲區（2026-04-24）」line 58-61 同類問題，此處放大為跨服務邊界風險。

### STATE-FLOW-01：嚴格狀態流轉

**規則**：Task `approval_status` 嚴格按以下流轉，跳級或回退**禁止**：
```
pending ─submit_for_review→ review_requested ─approve→ approved ─ingest→ ingested
                                ↓ reject
                              pending（回退；檔案解凍，cleaner 可重新編輯）
```

**規則細節**：
- `pending → approved` ❌（必經 review_requested）
- `approved → pending` ❌（已批准不能回退）
- `ingested → 任何狀態` ❌（已入庫終態）
- `rejected` 不是獨立狀態，是 `review_requested → pending` 帶 rejection_reason 寫入 audit log

**根因**：4/18 RejectTaskRequest empty note 422 修復間接揭露此鏈路 — 退回後檔案解凍邏輯依賴 `pending` 狀態而非 `rejected` 狀態。新增狀態時必先補完整流轉圖。

### ROW-LOCK-01：submit_for_review 必加 SELECT FOR UPDATE

**規則**：`review.py:submit_for_review` 在改 task 狀態前必執行 `SELECT ... FOR UPDATE` row lock：
```python
result = await session.execute(
    select(Task).where(Task.id == task_id).with_for_update()
)
task = result.scalar_one_or_none()
# ... 狀態檢查與更新
```

**根因**：4/18 Round 7 揭露 — 同一 cleaner 雙擊送審按鈕、或前端網路重試導致重複 POST，沒有 row lock 時兩個 transaction 並行讀到 `pending` 狀態都通過檢查，最終寫入兩次 `review_requested`，觸發 audit log 重複。lock 後第二個 transaction 等鎖釋放才讀，必看到已是 `review_requested` 狀態，可正確拒絕。

**邊界**：approve / reject / ingest 也建議加（同類 race），但目前優先級低於 submit（這是首次狀態轉換最易觸發雙擊）。

### AUDIT-01：每次狀態轉換必寫 cleaning_audit_logs（before/after JSON）

**規則**：以下操作**必寫**一筆 `cleaning_audit_logs` 條目，含 `actor_id` / `action` / `before_state` / `after_state`（JSON snapshot）/ `reason`（rejection 必填）：
- submit_for_review
- approve_task
- reject_task
- ingest_task
- update_file_content（4/18 Round 7 修復前漏寫）
- update_file_tags
- update_file_quality_checks

**規則細節**：
- `before_state` / `after_state` 是 task + task_files 的 JSON snapshot（不只狀態欄位）
- audit log 寫入失敗（DB 連線斷）→ 整個 transaction rollback，使用者收 500
- 對應 NestJS `@AuditEvent` interceptor，但**清洗側 audit 寫在 FastAPI 端**（NestJS proxy 不重複寫）

**根因**：4/18 Round 7 同時揭露 update_file_content 缺 audit log（Round 7 第 8 項修復）。新增任何修改 task / task_file 的端點時，audit log 是強制配套，不是可選。CLAUDE_LESSONS「稽核報告與實作的雙向同步（2026-04-18）」line 56 同概念。

### PSEUDO-TEST-01：禁 inspect.getsource 字串斷言；用 httpx + ASGITransport 真實 HTTP test

**規則**：cleaning-maker-checker 相關測試**禁止**：
```python
# ❌ 偽測試（4/18 Round 7 揭露 15+ 處此類）
def test_approve_blocks_submitter():
    source = inspect.getsource(approve_task)
    assert "submitted_by" in source
    assert "403" in source
```

必改為真實 HTTP integration test：
```python
# ✅ 真實 HTTP 行為驗證
async def test_approve_blocks_submitter():
    async with AsyncClient(transport=ASGITransport(app=app)) as client:
        # cleaner 送審
        await client.post(f"/review/{task_id}/submit", headers=cleaner_headers)
        # cleaner 嘗試自批 → 應 403
        resp = await client.post(f"/review/{task_id}/approve", headers=cleaner_headers)
        assert resp.status_code == 403
```

**根因**：4/18 Round 7 揭露 `test_review_api.py::TestReviewIngestMakerChecker` 15+ 處 `inspect.getsource` 偽測試，UUID vs str bug 通過所有測試但實際強制驗證失效。CLAUDE_LESSONS「測試反模式（2026-04-18）」line 36-38 + 「強制觸發 vs 用戶記住觸發的差別」line 85 完整背景。已強化 `pattern-anti-pseudo-test` skill 自動載入。

---

## 踩坑歷史（指針 + 1 句脈絡）

| 事件 | 日期 | 教訓位置 | 1 句脈絡 |
|------|------|---------|---------|
| Maker-Checker UUID vs str 比較失敗（3 端點）| 2026-04-18 | CLAUDE_LESSONS「型別比較陷阱」line 49-51 | 已內化為 TYPE-COMPARE-01（見上）|
| 15+ `inspect.getsource` 偽測試讓 UUID bug 存活 | 2026-04-18 | CLAUDE_LESSONS「測試反模式」line 36-38 | 已內化為 PSEUDO-TEST-01；待 Q2 替換實際測試（見 tasks.md 🔄）|
| submit_for_review 缺 SELECT FOR UPDATE | 2026-04-18 | RTM v1.4.0 ③ + Round 7 第 8 項 | 已內化為 ROW-LOCK-01 |
| update_file_content 缺 audit log | 2026-04-18 | RTM v1.4.0 ③ + Round 7 第 8 項 | 已內化為 AUDIT-01 |
| RejectTaskRequest empty note 422 | 2026-04-18 | RTM v1.4.0 ③ | Pydantic validator min_length=1，前端必送非空字串；已修復 |
| 稽核報告與實作雙向同步 | 2026-04-18 | CLAUDE_LESSONS「安全規劃模式」line 53-56 | 修補 gap 時必同步 gap-analysis + RTM；event-driven `Closes GAP-XX` |
| Service-layer IDOR 盲區 | 2026-04-24 | CLAUDE_LESSONS line 58-61 | 雖在 chat-rag，但同類風險（actor 偽造）對應 ACTOR-INJECTION-01 |
| Alembic version 漂移（task_files.quality_checks 缺欄）| 2026-04-24 | CLAUDE_LESSONS line 67-69 | 影響 update_file_quality_checks audit；環境建置必驗 schema 實際狀態 |

---

## ❌ Out of Scope（明確不做）

- **複數審核者投票機制**：US-18-12 v1 只支援單一 reviewer 批准/退回，不做「需 N 個 reviewer 全部批准才放行」。
- **Approval workflow 自訂**：審核流程嚴格按 `pending → review_requested → approved → ingested`，不支援使用者自訂多步驟流程（如：兩階審批、領域別審批）。
- **跨任務批次操作**：每個操作針對單一 task_id，不支援「一次批准 100 個任務」的 batch endpoint。
- **Submit 後自動分配 reviewer**：US-18-12 v1 由前端依角色顯示按鈕讓 reviewer 主動拾起，不做自動指派。
- **Audit log 即時通知**：寫入 cleaning_audit_logs 後不主動推送 cleaner 通知（無 websocket / email / Slack hook）。

---

## 試點驗收（D5）連結

本 spec pack 是 plan §D 第 2 試點，建立後 +7 天驗收（**修正版**：dry-run 立刻測，不等真實事件；+7 天為補強觀察期）：

- Q1 對人 self-review：選 1 個既有 commit（如 4/18 Round 7、4/24 IDOR fix）做 dry-run，是否 30 秒鎖定相關規則？
- Q2 對 AI 協作：開新 session 給虛構 bug 任務（「approve 端點漏寫 audit log」），是否 30-50 行 spec pack 建立完整 context、不追問已記錄的事？
- Q3 D0 紅線：grep `submitted_by|approved_by|UUID|SELECT FOR UPDATE|inspect` 抽 5 段，是否全為指針 + 功能特殊規則 + 踩坑歷史？

驗收 → CLAUDE_TASK.md「持續追蹤」條目（D5 +7 天 = 2026-05-09）。
