---
audience: ai-primary
---

# cleaning-maker-checker Spec Pack

> Audience: AI-primary. Feature-level working memory for Cleaning Maker-Checker review flow.

This spec pack exists because Cleaning Maker-Checker crosses NestJS（@CurrentUser JWT 注入）+ FastAPI（伺服器端強制驗證）+ SQLAlchemy（cleaning_audit_logs + SELECT FOR UPDATE row lock），是 PII 處理 + 職責分離雙敏感領域。Round 7（2026-04-18）一次發現 8 個關聯缺陷（UUID vs str 比較失敗、submit_for_review 缺 row lock、update_file_content 缺 audit log、RejectTaskRequest empty note 422、15+ inspect.getsource 偽測試等），多數屬「PRD/SRS 不會寫但開發必知」性質。

## 三件套

- **[requirements.md](./requirements.md)** — US/AC 指針 + 7 條功能特殊規則（TYPE-COMPARE-01、MC-INVARIANT-01、ACTOR-INJECTION-01、STATE-FLOW-01、ROW-LOCK-01、AUDIT-01、PSEUDO-TEST-01）+ 踩坑歷史
- **[design.md](./design.md)** — Architecture 指針 + 5 個功能級 KD（Actor 流向 / 不變式位置 / Row Lock 粒度 / Audit JSON Snapshot / ID 比較統一規則）
- **[tasks.md](./tasks.md)** — Round 7 完成清單 + 待辦（替換 15 處 inspect.getsource、補 race condition e2e）+ 未來規劃

## 試點期

- 建立日：2026-05-02
- D5 +7 天驗收：2026-05-09
- 驗收方式：dry-run（不等真實 commit），三題對照 — Q1 self-review 加速 / Q2 AI context 聚焦 / Q3 D0 紅線無複製
- 試點失敗處置：rm spec pack + RTM 移除 cleaning-maker-checker 列 + CLAUDE_LESSONS 補一條失敗教訓

## 守則

- D0 紅線：禁止複製 PRD/SRS/RTM/CLAUDE_LESSONS 既有內容；只放指針 + 功能特殊規則 + 踩坑歷史
- 每次本功能 commit（含 fix / feature）→ 同步更新 tasks.md ✅ 段
