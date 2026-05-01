---
audience: ai-primary
---

# cleaning-maker-checker — Design

> 守 D0 紅線：架構/技術細節若已在 ARCH.md / SRS-T，本檔只放指針；只記錄**功能級設計決策**與 trade-off。

---

## Architecture（指針）

宏觀架構見 [ARCH.md](../../01-specs/architecture/ARCH.md)：
- ADR-002 NestJS API Gateway → cleaning 端點集中於 `apps/api/src/modules/cleaning/`
- ADR-003 雙 ORM 策略 → cleaning 相關表（files, tasks, task_files, cleaning_audit_logs）由 SQLAlchemy + Alembic 管理（**非 Prisma**）
- ADR-009 Microsoft Presidio + spaCy → 上游清洗階段（不在本 spec pack 範圍）

完整資料清洗流程見 [data-cleaning-workflow.md](../../01-specs/architecture/data-cleaning-workflow.md)（NestJS proxy → FastAPI 處理 → 4 階段流轉）。

### cleaning-maker-checker 特定通訊鏈

```
Cleaner App (5503) / Admin (5501)
  → POST /api/cleaning/review/:taskId/{submit|approve|reject|ingest}
    → NestJS review.controller.ts
      ├─ JwtAuthGuard 驗 JWT
      ├─ RolesGuard 驗 RBAC（cleaner / reviewer / admin）
      ├─ @CurrentUser 取 actor identity（id / role）
      └─ CleaningProxyService.forward
          → POST FastAPI /api/v1/review/{task_id}/{submit|approve|reject|ingest}
            ├─ X-Internal-Token 驗證
            ├─ X-Actor-Id / X-Actor-Role headers（identity 從 NestJS 注入）
            ├─ review.py
            │   ├─ SELECT ... FOR UPDATE row lock（submit）
            │   ├─ str(actor_id) == str(task.submitted_by) 比較（MC-INVARIANT-01）
            │   ├─ 狀態流轉檢查（STATE-FLOW-01）
            │   ├─ Task / TaskFile.update（before snapshot 已留）
            │   └─ cleaning_audit_logs.insert（after snapshot + actor + reason）
            └─ HTTP 200 / 403 / 422
```

---

## Key Design Decisions（cleaning-maker-checker 特有）

### KD-1：Actor Identity 流向 — JWT @CurrentUser → X-Internal headers → FastAPI

**Decision**：actor identity 由 NestJS JWT `@CurrentUser` 取得，透過 `X-Actor-Id` / `X-Actor-Role` headers 傳給 FastAPI。FastAPI 不從 request body 接受 `actor_id` / `approver_id`。

**Why**：若 actor 從 body 來，惡意客戶端可送 `{ "approver_id": "victim-uuid" }` 偽造他人身份觸發 Maker-Checker 不變式檢查通過。Guard 層只驗 JWT 已登入，不驗 body 內容真實性。

**How**：
- NestJS `review.controller.ts` 用 `@CurrentUser() user` 注入
- `CleaningProxyService.forward(req, headers)` 自動帶上 `X-Actor-Id: ${user.id}` + `X-Actor-Role: ${user.role}` + `X-Internal-Token: ${env.INTERNAL_TOKEN}`
- FastAPI `review.py` 從 `request.headers` 讀，不從 `request_body.actor_id` 讀
- `X-Internal-Token` 確認來源是 NestJS（防外部直接 POST 到 FastAPI）

**Trade-off**：headers vs body
- 選 headers：與 Pydantic 驗證解耦（actor 不污染 schema），語意正確（actor 是 metadata 非 payload）
- 棄 body：失去 schema 驗證（如 UUID 格式），但 headers 有自訂 middleware 可補

### KD-2：Maker-Checker 不變式檢查放 FastAPI 而非 NestJS proxy

**Decision**：`submitted_by ≠ actor` 不變式檢查放在 FastAPI `review.py` 內，**不**在 NestJS proxy 層做（NestJS 只負責驗 JWT + RBAC + 注入 actor headers）。

**Why**：
- FastAPI 是真正擁有 task 狀態（DB row）的服務，能在 row lock 同一 transaction 內檢查不變式
- NestJS proxy 若做檢查需先讀 task → 再做檢查 → 再 forward，多一次 round trip 且 race window 大
- 若有人未來繞過 NestJS 直接打 FastAPI（如批量 import 工具），FastAPI 仍受不變式保護
- 對應 ARCH ADR-002 NestJS Gateway 角色：authn/authz boundary，業務不變式留給 owner service

**Trade-off**：proxy 層 vs service 層
- 選 service 層（FastAPI）：擁有狀態 + transaction 整合，安全邊界清晰
- 棄 proxy 層（NestJS）：複製檢查邏輯，且無 row lock 整合，race window 風險

### KD-3：SELECT FOR UPDATE 鎖粒度 — task 級而非 file 級

**Decision**：`submit_for_review` 用 task 級 `SELECT FOR UPDATE`（鎖 `tasks` 表單一 row），不鎖 `task_files`。

**Why**：
- 狀態 race 發生在 task.approval_status 欄位，不在 file 內容
- file 級鎖會放大 lock contention 範圍（一 task 可能 50+ files）
- task 級鎖足夠序列化「同一 cleaner 雙擊送審」場景

**How**：
```python
result = await session.execute(
    select(Task).where(Task.id == task_id).with_for_update()
)
task = result.scalar_one_or_none()
if task.approval_status != "pending":
    raise HTTPException(409, "Already submitted")
task.approval_status = "review_requested"
# audit log + commit
```

**Trade-off**：task 級 vs file 級 vs 樂觀鎖（version column）
- 選 task 級：粒度合適，PostgreSQL row lock 成本低
- 棄 file 級：lock contention 過大
- 棄樂觀鎖（version）：實作複雜度增、需處理重試；race window 仍存在第二次重試前

### KD-4：cleaning_audit_logs before/after JSON snapshot 設計

**Decision**：每次狀態轉換寫入 `cleaning_audit_logs.before_state` + `after_state` 為 task + task_files JSON 快照（不只變動欄位）。

**Why**：
- 完整 snapshot 可支援未來「回溯任意時點任務狀態」需求（如 reviewer 想看 cleaner 提交當下的全貌）
- 若只記變動欄位，跨多次 audit log 拼接歷史狀態時容易漏（特別是 file content 大幅編輯後）
- JSON 欄位（PostgreSQL JSONB）查詢成本低，儲存成本可接受（單筆 < 100KB）

**How**：
- `cleaning_audit_logs` 表（[`python/rag-service/src/rag_service/db/models.py:130`](../../../python/rag-service/src/rag_service/db/models.py)）含 `before_state JSONB` + `after_state JSONB` + `actor_id` + `action` + `reason`
- 每個 review.py endpoint 內：先 read task + files → 序列化為 before_state → 改狀態 → 序列化 after_state → insert audit log
- 寫入失敗（DB 連線斷、JSONB 太大）→ 整個 transaction rollback（依賴 SQLAlchemy session 機制）

**Trade-off**：full snapshot vs diff
- 選 full snapshot：查詢簡單、回溯完整、儲存成本可接受
- 棄 diff：查詢需重組多筆 audit log，邏輯複雜易錯

### KD-5：跨來源 ID 比較統一規則 — `str(a) == str(b)`

**Decision**：FastAPI 端任何身份 / ID 比較統一用 `str(a) == str(b)` 模式，不依賴 Python `==` 自動類型推斷。

**Why**：4/18 Round 7 第 1 項揭露 — Pydantic 解析 request body UUID 為 `uuid.UUID` 物件；SQLAlchemy ORM 若欄位定義為 `String` 則返回 str。`UUID == str` Python 評估為 False（即使值看起來相同），造成 Maker-Checker 強制驗證**形同虛設 3 個端點**（approve/reject/ingest）。

**How**：
- 統一 pattern：`if str(actor_id) == str(task.submitted_by):` （而非 `if actor_id == task.submitted_by:`）
- 對應 schema 設計：未來 SQLAlchemy 欄位若改為 `UUID(as_uuid=True)`，現有 `str(...) == str(...)` 比較仍正確（不需修改）
- 對應測試：`test_review_api.py` 必含「Pydantic UUID body vs SQLAlchemy str DB column」整合測試（用真實 DB，不用 mock）

**Trade-off**：`str()` 統一 vs `UUID()` 統一
- 選 `str()`：對 SQLAlchemy `String` / `UUID(as_uuid=True)` 雙模式皆正確，不需知道欄位型別
- 棄 `UUID()`：若欄位是 `String` 但值不是合法 UUID 字串，`UUID(...)` 會 ValueError；穩健性略差

---

## Known Pitfalls（指針）

| 坑 | 位置 | 防護 |
|---|------|------|
| 4/18 UUID vs str 比較失敗（approve/reject/ingest 3 端點）| CLAUDE_LESSONS「型別比較陷阱」 | KD-5 統一 `str(a)==str(b)` |
| 4/18 submit_for_review 缺 row lock | CLAUDE_LESSONS「Bug Fix Round 7」 | KD-3 task 級 SELECT FOR UPDATE |
| 4/18 update_file_content 缺 audit log | CLAUDE_LESSONS「Bug Fix Round 7」 | KD-4 + AUDIT-01：所有修改端點必寫 |
| 4/18 15+ `inspect.getsource` 偽測試 | CLAUDE_LESSONS「測試反模式」 | PSEUDO-TEST-01；`pattern-anti-pseudo-test` skill 自動觸發 |
| 4/18 RejectTaskRequest empty note 422 | RTM v1.4.0 ③ | Pydantic `min_length=1` validator |
| 4/24 Alembic schema 漂移 | CLAUDE_LESSONS「Alembic Migration 漂移」 | 環境建置 SOP 加 `\d <table>` 驗證；非本 spec pack 直接負責 |
| 4/24 Service-layer IDOR（chat-rag）| CLAUDE_LESSONS「Service-layer IDOR 盲區」 | 同類風險：ACTOR-INJECTION-01 防 actor 偽造 |

---

## 跨功能依賴

| 依賴方向 | 對象 | 介面 |
|---------|------|------|
| cleaning-maker-checker → cleaning-pipeline | 上游清洗 | task.approval_status 從 `cleaning_complete` 自動進入 `pending`（待送審）|
| cleaning-maker-checker → knowledge-base | ingest 端點 | `ingest_task` 將 approved task_files 寫入 Qdrant + BM25 索引 |
| cleaning-maker-checker → audit | cleaning_audit_logs | 與系統級 audit_logs（NestJS 端）並存，職責分離（cleaning-side audit 寫在 FastAPI）|
| cleaning-maker-checker → identity | actor identity | JWT @CurrentUser → X-Internal headers |
| ← 被 Cleaner App / Admin 呼叫 | apps/cleaner / apps/admin | REST 透過 NestJS proxy |

未來變更若涉跨服務 contract，需同步更新 [api/cleaning-api.md](../../01-specs/api/cleaning-api.md) 與本 design.md KD 段。
