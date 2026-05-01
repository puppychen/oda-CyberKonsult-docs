---
audience: both
---

# docs/04-features/ — 功能級 Spec Pack 規約

> **此目錄存放「功能級」三件套**：requirements.md / design.md / tasks.md。
> 不是 PRD/SRS/RTM 的副本，**只放它們不該寫但開發必知的功能特殊規則**。
>
> 起點：2026-04-29（Codex 獨立稽核建議 + 4/13 / 4/18 / 4/24 三組漂移事件驅動）。
> 試點 #1：chat-rag — D5 三題提前通過（2026-05-02 修正版 dry-run 全 ✅，原 +7 天驗收 2026-05-06 改為補強觀察期）。
> 試點 #2：cleaning-maker-checker — 2026-05-02 啟動，D5 +7 天補強觀察期至 2026-05-09。

---

## 為什麼存在這個目錄

PRD/SRS/RTM 是 **Tier 1 規格 SSoT**，回答「系統做什麼」。
但有些資訊**不適合**放 Tier 1：

| 不適合放 PRD 的內容 | 為什麼 | 應該放哪 |
|--------------------|--------|----------|
| IDOR ownership 4 項檢查清單 | 跨多功能，PRD 只描述使用者視角行為 | 功能級 spec pack（指向 `domain-code-review`） |
| 「閾值 0.005 是過濾、0.003 是信心度」雙語意警告 | PRD 已寫值，但語意誤解導致的開發踩坑屬實作層 | spec pack design.md KD 段 |
| 4/24 IDOR 案踩坑歷史 + 防範要點 | CLAUDE_LESSONS 有，但與某功能強相關的需 1-2 句重點 | spec pack requirements.md 指針 |
| 跨服務通訊細節（NestJS → rag-service → Qdrant） | ARCH.md 有總體架構，但功能級對話需即時參考 | spec pack design.md Architecture 段 |

**核心定位**：spec pack ≠ 縮小版 SRS；是「功能操作手冊」，AI session 修這支功能 bug 時 30-50 行即建立完整 context。

---

## Three-Tier SSoT 在本專案的對應

```
Tier 1（規格層）        docs/01-specs/PRD.md, SRS_TECHNICAL.md, ARCH.md, RTM.md
                        ↓ 指針
Tier 2A（橫向常數/規則）contracts/{thresholds,enums}.yml + ~/.claude/skills/*
                        ↓ 指針
Tier 2B（功能級 SDD）   docs/04-features/{feature}/{requirements,design,tasks}.md  ← 本目錄
                        ↓ 引用
Tier 3（實作）          apps/api, packages/contracts, python/*
```

完整決策樹 + 跨專案通用規則：`~/.claude/skills/process-ai-native-sdlc/references/three-tier-ssot-architecture.md`。

---

## 何時建立 spec pack（4 觸發條件）

**至少 1 條觸發即可建立**：

1. **歷史事件累積**：CLAUDE_LESSONS 有 ≥ 2 條與此功能相關（4/24 IDOR + 4/22 dual-track + 4/18 漂移 → chat-rag 滿足）
2. **跨服務邊界**：NestJS ↔ Python ↔ Qdrant ↔ SearXNG 多端協作
3. **資安敏感**：Auth、Ownership、PII、Maker-Checker
4. **業務複雜**：多角色、多狀態、多步驟工作流

**不建立**（保持輕量）：
- 純 CRUD（PRD + framework-nestjs 已涵蓋）
- 健康檢查、ping
- Phase 3 規劃功能
- 即將汰除的功能

---

## 三件套結構與內容守則

### requirements.md

| 區塊 | 內容 | 紅線 |
|------|------|------|
| User Stories | **僅指針**到 PRD US-XX | ❌ 不複製 US 描述 |
| Acceptance Criteria 表格 | AC ID + 一句話事實註腳（值來自 contracts/ 指針）| ❌ 不複製 PRD AC 完整描述 |
| Authorization Rules | SEC-IDOR-XX 等功能特殊條目（指向 `domain-code-review`）| ✅ Tier 1 沒有的功能規則才寫 |
| Pitfall History | CLAUDE_LESSONS 相關片段 + 1-2 句脈絡 | ❌ 不複述完整 lesson |
| Out of Scope | 列出 v2 / 不在本功能範圍的關聯項 | ✅ |

### design.md

| 區塊 | 內容 |
|------|------|
| Architecture | controller → service → 各依賴的「呼叫鏈」一張圖（mermaid 或文字）|
| Key Design Decisions (KD) | KD-1, KD-2 ... 每條一段話，含「採用什麼」+「為什麼」+「ADR 連結（若有）」|
| Trade-offs | 列出本功能採用的方案 vs 替代方案 + 取捨理由 |
| External Dependencies | 此功能依賴的外部系統 / 資料表 / Tier 2A 常數的明確指針 |

### tasks.md

| 區塊 | 內容 |
|------|------|
| ✅ 已完成 | 按 `YYYY-MM-DD series` 分組，每條一行：commit hash + AC ID + 1 句事實 |
| 🔄 待辦 | 試點期可選做（不卡 sprint） |
| 🔮 未來（v2）| 變更前不啟動 |
| Spec Pack 維護紀律 | 重申更新時機 + D0 紅線提醒 |

---

## D0 反模式紅線（**強制規則**，違反即試點失敗）

❌ **禁止**：
1. 複製 PRD/SRS/RTM 既有 AC、FR、ADR 內容到 spec pack（雙源 = 必漂移）
2. 把 spec pack 當「設計討論草稿」（討論寫在 plans/ 或對話）
3. 寫完不更新（每次該功能 commit 必同步 tasks.md）

✅ **只准放**：
- 指針（PRD#L117-126、`contracts/thresholds.yml#rag.hybrid_filter_min`）
- 功能級特殊規則（PRD/SRS 不該寫但開發必知）
- 踩坑歷史指針 + 1-2 句脈絡

**自我檢查**：每段內容自問「**這個資訊在 PRD/SRS/ARCH/RTM/CLAUDE_LESSONS 有了嗎？**」
- 有 → 改寫成指針
- 沒有 → 寫進 spec pack

---

## 切分原則（spec pack 怎麼劃分？）

**正確顆粒：business function group**（業務功能群），不是 NestJS 模組。

| 切法 | 範例 | 評價 |
|------|------|------|
| ✅ 按 business function | `chat-rag`（包含檢索 + LLM 編排 + 串流 + 信心度，但不含 feedback CRUD）| 對應 PR 異動範圍、CLAUDE_LESSONS 集中、AI context 聚焦 |
| ❌ 按 NestJS 模組 | `chat`（涵蓋 RAG + prompts + feedback + confidence）| 過於寬泛，變迷你 SRS |
| ❌ 按單一 controller method | `chat-stream-endpoint` | 過於細碎，維護成本 > 效益 |
| ❌ 按 user story | `US-02-01` | RTM 已涵蓋，重複 |

**判斷指引**：spec pack 應該對應「PR 通常涉及的範圍」+「CLAUDE_LESSONS 自然聚集的範圍」。
若同一個 PR 經常跨多個 spec pack → 表示切太細；若同一 spec pack 內容過長 → 表示切太粗。

---

## 候選清單與試點狀態（2026-04-29）

| 候選功能 | 觸發條件評分 | 狀態 | 備註 |
|---------|-------------|------|------|
| **chat-rag** | 4/4（4/24 IDOR + 4/22 dual-track + 4/18 漂移；NestJS+Python；安全；複雜）| ✅ 試點通過 | 2026-05-02 修正版 D5 dry-run 全 ✅；補強觀察期至 2026-05-06 |
| **cleaning-maker-checker** | 4/4（4/18 UUID bug；NestJS+Python；Maker-Checker；多狀態流轉）| ✅ 試點中 | 2026-05-02 啟動；D5 +7 天補強觀察期至 2026-05-09 |
| **auth-credential-rotation** | 2/4（無近期事件；單服務；安全敏感；中度複雜）| 🟡 暫緩 | RTM v1.4.0 ③ 已完整記錄，邊際價值低 |
| prompts-crud | 0/4 | ❌ 不建 | 純 CRUD |
| audit-logs | 0/4 | ❌ 不建 | 已 SSoT 化（`audit-actions.ts`） |
| websearch-config | 0/4 | ❌ 不建 | 純 CRUD |
| health-check | 0/4 | ❌ 不建 | 簡單端點 |

**推廣節奏**：chat-rag D5 通過（3 題全 ✅，2026-05-02 修正版 dry-run）→ cleaning-maker-checker 啟動第 2 試點（2026-05-02）→ 第 2 試點 D5 通過（最早 2026-05-09）→ 再評估 auth-credential-rotation。

**修正版 D5 機制**（取代「+7 天等真實 PR」假設）：
- Q1 對人 self-review：選既有 commit dry-run（不等真實事件），檢查 spec pack 規則是否能 30 秒鎖定 PR 觸碰範圍
- Q2 對 AI 協作：開新 session 給虛構 bug 任務，觀察是否 30-50 行 spec pack 建立完整 context
- Q3 D0 紅線：`grep` 抽 5 段，自問「這資訊在 PRD/SRS/RTM/CLAUDE_LESSONS 已有嗎？」
- +7 天為**補強觀察期**（被動記錄真實 commit 是否使用 spec pack），不是必要等待

---

## 與 contracts/ 和 ~/.claude/skills/ 的關係

| 場景 | 該放哪 |
|------|--------|
| 7 個 RAG 閾值（跨 TS+Python，PRD AC 引用）| `contracts/thresholds.yml`（Tier 2A）|
| 7 個 user roles（DTO + PRD）| `contracts/enums.yml`（Tier 2A）|
| IDOR 4 項檢查（跨所有功能）| `~/.claude/skills/domain-code-review/SKILL.md` line 33（Tier 2A）|
| 「chat-rag 採 contracts/thresholds.yml 為唯一閾值來源」事實 | `chat-rag/design.md` KD-3（指針，Tier 2B）|
| 「user A 試讀 user B 對話 → 403」negative test 必補 | `chat-rag/tasks.md` T-NEG-01（Tier 2B）|
| 跨多功能的「Maker-Checker UUID 比較陷阱」 | `~/.claude/skills/process-ai-native-sdlc/references/qg3-defense-lines.md`（Tier 2A）|

**反模式**：把 IDOR 4 項清單複製到每個 spec pack → DRY 違反，規則漂移。**正解**：spec pack 寫「依 `domain-code-review` IDOR 4 項清單」一行指針。

---

## RTM 整合方式

`docs/01-specs/RTM.md` 既有寬表格 `| US | API | NestJS 模組 | NestJS 測試 | Python 測試 | TC | Screen | Security | 狀態 |`，**不在試點期擴欄**，避免為單一試點功能改動整張 RTM。

試點期採輕量整合：
- 在對應 FR 區塊開頭加入 `Spec Pack` 連結，例如 FR-02 指向 `docs/04-features/chat-rag/`
- 若 D5 驗收通過且第 2 個 spec pack 建立，再評估是否新增 RTM 欄位
- 若 D5 驗收失敗，移除 FR 區塊連結與 `docs/04-features/chat-rag/`

詳見 RTM v1.6.0 變更記錄。

---

## 自動驗證

每次新增或修改 spec pack 後執行：

```bash
pnpm lint:spec-pack
```

檢查項目：
- 三件套完整性：每個功能目錄都必須有 `requirements.md` / `design.md` / `tasks.md`
- Markdown 相對連結是否存在
- RTM 是否有對應功能的 Spec Pack 連結
- D0 紅線：不得精準複製 Tier 1 長段落
- D5 gate：`chat-rag` 試點中時，不允許提前新增第二個功能級 spec pack 目錄

壞案例自測：

```bash
node scripts/spec-pack-lint.mjs --self-test
```

---

## 維護紀律

| 時機 | 動作 |
|------|------|
| 該功能 commit（fix / feature）| tasks.md ✅ 段加一行（commit hash + AC ID）|
| PRD AC 變更 / contracts 變更 | 同步檢視三件套是否仍正確（指針失效要修），並跑 `pnpm lint:spec-pack` |
| spec pack 試點結果（D5 +7 天）| 加「## 驗收結果」段；❌ 整個 spec pack 撤除（連同 RTM FR 區塊連結）|
| CLAUDE_LESSONS 新增該功能教訓 | requirements.md Pitfall History 段補一行指針 |

---

## Cross-References

- **Spec pack 通用設計（跨專案）**：`~/.claude/skills/process-ai-native-sdlc/references/three-tier-ssot-architecture.md`
- **QG-3 三道防線**：`~/.claude/skills/process-ai-native-sdlc/references/qg3-defense-lines.md`
- **contracts/ 邊界**：`../../contracts/README.md`
- **試點任務追蹤**：`../../CLAUDE_TASK.md` 持續追蹤段
- **試點實例**：`./chat-rag/`
