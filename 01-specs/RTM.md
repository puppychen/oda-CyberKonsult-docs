---
audience: ai-primary
purpose: reference
status: approved
owner: ODA Cyber Konsult
---

# 需求追溯矩陣 (RTM)

> **ODA Cyber Konsult - 資安助手 RAG 系統**
>
> 文件版本：1.15.1
> 建立日期：2026-03-01
> 最後更新：2026-07-29
> 文件類型：需求追溯矩陣（Requirements Traceability Matrix）

---

## 版本歷史

| 版本 | 日期 | 變更說明 |
|------|------|----------|
| v1.7.0 | 2026-07-11 | RFC-001：六角色契約、回應層級直接提示詞、新手每日額度、Token 失效與前端驗收追溯 |
| v1.8.0 | 2026-07-12 | RFC-002：新增 FR-24 分類、API、migration 與測試追溯 |
| v1.9.0 | 2026-07-12 | RFC-003：新增任務命名、快速改名、UTC／台灣時間與 migration 012 追溯 |
| v1.10.0 | 2026-07-13 | 新增 US-02-06：議題四態分類、SSE／metadata 契約與前端提醒追溯 |
| v1.11.0 | 2026-07-13 | 認證穩定化：獨立 session、原子 refresh、SSO 交換、同頁重新登入、跨分頁協調與 Prisma 0009 追溯 |
| v1.12.0 | 2026-07-13 | 補 Token 用途隔離、同帳號重新登入、租約續期、重複 401 延後處理與真實併發輪替追溯 |
| v1.13.0 | 2026-07-14 | 新增 US-02-07／TC-02-002：最新回覆 icon-only 重新產出、保留原問答、API 最新訊息驗證、額度與 metadata 追溯 |
| v1.13.1 | 2026-07-14 | 強化 US-02-07：相同提示詞、row lock、attempt 冪等、pending／completed 重試與 PostgreSQL 並發驗證 |
| v1.13.2 | 2026-07-14 | US-02-07 加入 processing／failed claim、防重入 409、斷線釋放與續跑驗證 |
| v1.13.3 | 2026-07-14 | US-02-07 加入 5 分鐘 lease、fencing、跨分頁 attempt 定位與 completed claim 驗證 |
| v1.13.4 | 2026-07-14 | US-02-07 加入失敗釋放 lease fencing，以及額度與 claim 同交易回滾保證 |
| v1.13.5 | 2026-07-14 | US-02-07 加入 metadata lease CAS、後端 topicScope 限制與真實額度回滾測試 |
| v1.14.0 | 2026-07-16 | 新增 US-20-02／TC-20-001：管理員待審核來源預檢與邏輯刪除、全關聯阻擋、Qdrant 失敗關閉、交易重驗、歷史保留及 Cleaner 確認流程追溯 |
| v1.14.1 | 2026-07-16 | 補強 FR-20：已刪除專用查詢、來源清單歷史入口、操作欄語意與 Cleaner 台北時區測試追溯 |
| v1.14.2 | 2026-07-17 | 新增 TC-20-003：來源未送審、最新任務狀態、先篩選後計數／穩定分頁與 PostgreSQL rollback 追溯 |
| v1.14.3 | 2026-07-18 | US-20-01 增加來源檔案與最新任務名稱對照、舊任務 ID 備援及名稱／狀態／連結同源測試追溯 |
| v1.15.0 | 2026-07-26 | 強化 US-02-06／新增 TC-02-003：全題 RAG 優先、嚴格證據門檻、縮寫限定網搜推測、單輪原子銜接、RAG 不可用分流、資安建議題與 SSRF／提示注入防線追溯 |
| v1.15.1 | 2026-07-29 | 新增 TC-02-004：一般訊息 attempt 冪等、user row lock、原子額度、lease fencing、完成回放與 PostgreSQL 雙併發追溯 |
| v1.0.0 | 2026-03-01 | 初版建立，FR-01~21 追溯矩陣 |
| v1.1.0 | 2026-03-06 | 新增 FR-18 Maker-Checker 追溯、FR-19~21 追溯、驗收測試更新 |
| v1.2.0 | 2026-03-11 | 新增 US-05-03 ZIP 追溯、更新角色定義為 7 角色、測試覆蓋統計更新（107 test files） |
| v1.3.0 | 2026-03-16 | ML-15：新增 FR-22 回饋機制追溯、FR-23 信心度追溯；驗收案例 TC-05-006/007 確認 |
| v1.4.0 | 2026-04-18 | **Wave 1-5 功能 + 機制修正追溯同步**：<br/>① 新增 19 項 Wave 功能追溯（標籤範本/報告模板/Demo 問題庫/Golden Test/測試執行器/法規版本管理）<br/>② gap-analysis P1 修補映射（GAP-01 ~ GAP-05 全數 closed）<br/>③ 記錄本 session 揭露的 Maker-Checker UUID 比較 bug 修復、changePassword tokenVersion 補齊、Expert maxTokens 規格校正、ZIP 閾值校正、query history 5 輪修正、RejectTaskRequest validator、file content audit log、SELECT FOR UPDATE 行鎖<br/>④ 新增反偽測試覆蓋率指標（舊 15 處 `inspect.getsource()` 已識別，待 Q2 取代）<br/>⑤ 補充 AC-02-05 vs AC-23-01 閾值語意差異說明<br/>⑥ 明確標註 3 項 NFR 未驗證項目（PII 95% 召回率、P95 < 10s、10 files/分吞吐量） |
| v1.6.4 | 2026-05-04 | **dev 機制穩定性修復 + 假環境拆除（plan dazzling-wiggling-hopper 一次完整修法）**：<br/>① **contracts CJS 修法**（commit `98e2b94`）：`packages/contracts/tsconfig.json` 強制 `module: CommonJS, moduleResolution: node` override；根因為 `packages/tsconfig/base.json` 預設 ES2022/bundler 設定（給打包工具用）誤套到 contracts 的 Node runtime 消費者，導致 NestJS bootstrap 載入 contracts/dist/index.js 時 ERR_MODULE_NOT_FOUND（`./loader` 無 .js 副檔名）→ NestJS 從未 listen 3051 → admin vite proxy 把 connection refused 包成假 500 → Jake 嚴重不滿「假東西」「不專業」「commit 完成但系統不能用」<br/>② **vite proxy fail-loud**（commit `514f76d`）：3 個 vite app（admin / cleaner / chatbot）的 `vite.config.ts` proxy 加 `error` handler；backend `:3051` 連不到時改回 502 + JSON `{error: "Bad Gateway", hint: "Run pnpm --filter @oda-cyber/api dev..."}`，取代 vite 預設「假 500」掩蓋真錯誤的行為；終端機 console 同步印明確訊息<br/>③ **dev-smoke 防呆**（commit `2d7431d`）：新建 `scripts/dev-smoke.sh` + `pnpm smoke` 命令；curl 5 個服務（NestJS API / RAG / 3 vite）health endpoint，任一失敗 exit 1 + 訊息「dev 啟動聲稱完成 ≠ 系統實際 work」；強制底層機制 commit 後必跑 smoke，不依賴 unit test（jest.mock 繞過真實載入）<br/>④ **驗證**：3051 listen 確認、admin login 200 + JWT、7 角色 curl 登入 7/7、it_user `/api/chat/modes` 回 `[beginner, standard]` 對齊 SRS §5.17.3、consultant 回 `[beginner, standard, expert]`、vite proxy fail-loud 真實情境（殺 NestJS）回 502 + JSON、dev-smoke 殺 NestJS 後 exit 1 + UNREACHABLE 訊息<br/>⑤ **`CLAUDE_LESSONS.md` 新增段「commit 完成 ≠ 系統 work（2026-05-02 嚴重失職案例）」**：6 條教訓含「jest.mock 繞過真實載入」「commit message 是承諾要兌現」「連 commit 6 個的 cascading failure」「vite proxy 預設掩蓋真實錯誤」「強制 dev-smoke 配套」「快速產出多 commit 是反指標」 |
| v1.6.3 | 2026-05-02 | **it_user 顧問模式 UI 同步（v1.6.1 ③ 後續清理）**：<br/>① **`/api/chat/modes` 端點 it_user 對應修正**：`apps/api/src/modules/chat/controllers/chat.controller.ts:329` 由 `['beginner', 'standard', 'expert']` 改為 `['beginner', 'standard']`，與 SRS §5.17.3 表格對齊<br/>② **`apps/chatbot/src/pages/ChatPage.tsx:49` 註解清除 it_user 角色**（expert 模式報告匯出原條件，雖功能 v1.6.1 ① 已暫時停用，註解仍含 it_user 與撤回不一致）<br/>③ **新建 `apps/api/src/modules/chat/controllers/chat.controller.spec.ts`**：getModes() 7 角色映射 + it_user 不含 expert 防回退測試 + 未知角色 fallback 測試 + 預設模式驗證（10/10 綠）<br/>④ **seed.ts 確認**：grep 後確認原本就無 it_user expert prompt（v1.6.1 ③「待後續清理」事項實際不存在，自然完成）<br/>⑤ **緣由**：5/2 系統運行驗證透過 Chrome 對 7 角色測試時發現 it_user 仍可在 chatbot 模式選單看到「顧問」可選；前端 `RoleSelector.tsx` 動態 fetch `/api/chat/modes` 機制正確，但後端 modeMap 漏對齊 v1.6.1 撤回 |
| v1.6.2 | 2026-05-02 | **Spec Pack 試點全面廢除（Tier 2B 機制移除）**：<br/>① **`docs/04-features/` 整目錄移除**（chat-rag + cleaning-maker-checker 共 9 檔），廢除理由為 audience × 持久化矩陣分析顯示「個人 + AI + 入 git」組合無 fits 位置，14/14 段內容皆能映射既有 SSoT<br/>② **內容回流既有 SSoT**：chat-rag IDOR 服務清單 + cleaning-maker-checker 4 端點清單 + Maker-Checker 4 端點不變式 + 嚴格狀態機流轉 + SELECT FOR UPDATE 並發控制 + 閾值雙語意陷阱 → 全部回寫 `CLAUDE_LESSONS.md`（gitignored AI 記憶）<br/>③ **FR-02 / FR-18 區塊 Spec Pack 連結移除**（line 86 / 209）；本版本記錄為刻意保留的廢除事件追蹤<br/>④ **skill 系統同步**：`process-ai-native-sdlc/SKILL.md` Three-Tier → Two-Tier；`references/three-tier-ssot-architecture.md` → `two-tier-ssot-architecture.md` 改名重寫；`output-identity/SKILL.md` 移除 spec pack 性質整章；個人 `~/.claude/CLAUDE.md` 資料歸屬矩陣移除 spec pack 列<br/>⑤ **lint 改名簡化**：`scripts/spec-pack-lint.mjs` → `scripts/docs-lint.mjs`；`pnpm lint:spec-pack` → `pnpm lint:docs`；移除 feature triplet completeness / D5 pilot gate / RTM Spec Pack column 檢查；保留 broken-link + Tier 1 long-line copy 並擴大掃描至整個 `docs/`，順帶修正 3 處 pre-existing broken link 與 1 處 typo<br/>⑥ **CLAUDE_LESSONS 新增「Spec Pack 試點失敗（2026-05-02）」段**，記錄 audience × 持久化矩陣自檢、集中 context 成立條件、plan 內部編號禁外溢、業界術語檢核、沉沒成本偏見辨識訊號 5 條教訓<br/>⑦ **plan 內部編號清除**：工作區 14 檔移除 plan §B / §D5 / D0 紅線 / Tier 2B / 04-features 引用；`ARCH.md` ADR-011 試點期描述還原為 Accepted；`PROJECT_CONTEXT.md` SDLC Governance 表移除 Spec Pack Policy 列 |
| v1.6.1 | 2026-05-02 | **Spec pack 第 2 試點 + 修正版 D5 機制 + SRS 撤回**：<br/>① **cleaning-maker-checker spec pack（第 2 試點）**：`docs/04-features/cleaning-maker-checker/{README,requirements,design,tasks}.md` 四件套；含 7 條功能特殊規則（TYPE-COMPARE-01 / MC-INVARIANT-01 / ACTOR-INJECTION-01 / STATE-FLOW-01 / ROW-LOCK-01 / AUDIT-01 / PSEUDO-TEST-01），FR-18 區塊加 Spec Pack 連結<br/>② **修正版 D5 機制**：原 D5「+7 天 PR review 變快」假設不適用個人 commit-based 工作流；改為 dry-run 立刻測（不等真實事件）+ +7 天為補強觀察期；Q1 用既有 commit 演練 self-review 加速、Q2 用新 session 給虛構 bug 任務、Q3 grep 抽樣自查 D0 紅線；對 chat-rag 第 1 試點同步重做<br/>③ **SRS_TECHNICAL §5.17.3 it_user expert mode 撤回**（v1.5.0 ⑥ 撤回）：經 product owner 5/2 確認，it_user 不開放 expert mode；SRS 表格還原為 `beginner, standard`；seed.ts 是否含 it_user expert prompt 待後續清理<br/>④ **ai-native-sdlc skill PR 假設修正**：QG-3 三道防線（references/qg3-defense-lines.md）+ SKILL.md mypy patch-strict 描述「PR diff」改為「branch diff vs main」，對應 commit-based 個人工作流<br/>⑤ **資料正確性 7 項 cross-validation 通過**：規格-實作 / 術語 / Bounded Context / RTM 行號 / PROJECT_CONTEXT 索引 / README broken-link / chat-rag D5 自我驗收全綠 |
| v1.6.0 | 2026-04-29 | **SDD contracts SSoT + spec pack 試點（B+D 任務批次）**：<br/>① **`contracts/` 跨層級 SSoT 建立**：root `contracts/thresholds.yml` 收 7 個 RAG 閾值常數；`contracts/enums.yml` 收 user_roles 7 角色；`packages/contracts` workspace 提供 TS 載入；`python/shared/contracts.py` Python stub<br/>② **`chat.service.ts` 7 處 magic number 替換**為 `RAG_THRESHOLDS.*`（line 326/341/385-386）；`create-user.dto.ts` 改用 `USER_ROLES`<br/>③ **anti-drift 行為驗證**：`chat.service.contract.spec.ts` 用 jest.mock 注入 0.999 極端值（5/5 綠）；`python/tests/test_contract_parity.py` 跨語言 schema parity（7/7 綠）<br/>④ **chat-rag 試點 spec pack**：`docs/04-features/chat-rag/{requirements,design,tasks}.md` 守 D0 紅線（指針 + 功能特殊規則 + 踩坑歷史，禁止複製 PRD）<br/>⑤ **+7 天驗收期**：CLAUDE_TASK.md 持續追蹤條目（D5 三題：人審查/AI context/D0 紅線；任一 ❌ 即撤除試點）<br/>⑥ 對應 ai-native-sdlc skill QG-3 強化（references/qg3-defense-lines.md）+ pattern-anti-pseudo-test 自動載入 + domain-code-review IDOR 4-checklist |
| v1.5.1 | 2026-04-29 | **文件對齊（A 任務批次）**：<br/>① **PRD AC-23-01-04 補閾值語意交叉註腳**（PRD line 582）：明示「過濾閾值 vs 信心度判定閾值」兩層不同語意，引用 line 127-131 閾值語意說明<br/>② **system-cross-validation-2026-04-18.md §1.2「矛盾 1」+ §3.2「飄移 #6」校正撤回**：經 PRD line 127-131 確認 0.005/0.003 為兩層獨立閾值（filterLowScoreResults vs determineConfidenceLevel 兩函式），原稽核判斷錯誤已標註撤回；保留軌跡供後續 reviewer 追溯<br/>③ **新增 AC-01-01-05 tokenVersion 失效規格**：補 fd20f53 修復對應的規格條文（密碼變更後 tokenVersion 遞增、既發 JWT 立即失效）。CLAUDE_LESSONS「Credential Rotation 事件窮舉」對應 |
| v1.5.0 | 2026-04-22 | **codex 稽核 4 項 + 自我稽核 2 項「文件完成 ≠ 實作完成」缺陷修正**：<br/>① **變數注入**（AC-03-01-03）：context-builder 僅處理 `{context}`，補齊 `{user_name}` / `{query}` 替換，新增共用 `prompt-renderer.util.ts`，chat.service fetch `users.getDisplayName` 注入 user_name<br/>② **提示詞測試 vs 正式執行格式統一**（US-03-03）：prompts.service.testPrompt 原用 `{{key}}` 雙括號與正式 `{key}` 單括號不符，統一改用單括號 renderer，新增 `prompts.service.spec.ts` 等價性 golden test 保證未來兩路徑不再分歧<br/>③ **三層模式 topK 差異化**（AC-16-02-01~03）：chatbot 前端固定送 `topK: 5` 覆蓋後端 mode-aware fallback (3/5/8)，移除硬編碼，新增 `useChat.test.ts` smoke test<br/>④ **Prompt UI 角色擴充**（AC-01-02-03）：admin PromptsPage 補齊 `basic_user` / `it_user` 角色（colors / labels / filters / Select options 四處），使 SRS §4.4 規定的「it_user 專屬提示詞」得以透過 UI 建立<br/>⑤ **UpdatePromptDto `it_user` 遺漏補齊**（自我稽核發現）：create-prompt.dto.ts 已列 5 角色，但兄弟 update-prompt.dto.ts 只列 4 角色——意即 it_user 提示詞可建立、不可編輯；同時兩份 DTO spec 的 it.each 也遺漏了 it_user。修復方案：提煉 `PROMPT_ROLES` / `PROMPT_MODES` 共用常數於 create-prompt.dto.ts，create/update DTO 與兩份 spec 全部引用同一來源，根除列舉雙源漂移<br/>⑥ **SRS §5.17.3 表格校正**：原「it_user | standard | beginner, standard」與 PRD v1.5.0 版本歷史 line 21「it_user 開放 expert 模式」矛盾，seed.ts 已有 it_user expert 提示詞；表格更新為「beginner, standard, expert」<br/>⑦ 本次修正對應的測試新增：`prompt-renderer.util.spec`、`prompts.service.spec`、`context-builder.service.spec`（新增變數替換案例）、`useChat.test`；既有 `create-prompt.dto.spec` / `update-prompt.dto.spec` 改為引用 PROMPT_ROLES 常數驅動<br/>⑧ CLAUDE_LESSONS.md 新增「規格漂移與驗證漏洞」段落記錄四項通用教訓（含新增的「Create/Update DTO 兄弟對稱 + 列舉覆蓋測試漂移」） |

---

## 文件目的

本文件建立從 User Story 到測試案例的完整追溯鏈路，確保每個需求都有對應的實作與驗證。

### 追溯鏈路

```
User Story (PRD.md)
  → 功能需求 (SRS_TECHNICAL.md FR-XX)
    → API 端點 (docs/api/)
      → 實作模組 (NestJS / Python)
        → 測試案例 (Unit / Integration / E2E)
          → 驗收測試 (SRS_TECHNICAL.md §11 TC-XX)
```

### 狀態圖例

| 圖示 | 意義 | 說明 |
|------|------|------|
| ✅ | 已實作 | 功能完成且有測試覆蓋 |
| 🔄 | 規劃中 | Phase 2 排程實作 |
| 🔮 | 未來願景 | Phase 3 以後 |

### 文件引用

| 縮寫 | 全名 | 路徑 |
|------|------|------|
| PRD | 產品需求文件 | [specs/PRD.md](./PRD.md) |
| SRS-B | 業務需求規格 | [specs/SRS_BUSINESS.md](./SRS_BUSINESS.md) |
| SRS-T | 技術需求規格 | [specs/SRS_TECHNICAL.md](./SRS_TECHNICAL.md) |

---

## 1. 追溯矩陣 — 已實作功能

### FR-01：使用者認證與授權 🔄

> **角色定義**（6 角色）：basic_user / user / consultant / data_cleaner / data_reviewer / admin

| US | 說明 | API 端點 | NestJS 模組 | NestJS 測試 | Python 測試 | TC | Screen | Security | 狀態 |
|----|------|----------|------------|-------------|-------------|-----|--------|----------|------|
| US-01-01 | 帳號密碼登入 | `POST /api/auth/login` | auth/ | auth.service.spec, token.service.spec, login.dto.spec, password-strength.validator.spec, password-change-required.guard.spec、`auth-login-lock-postgres.integration.ts` | — | TC-01-001, TC-01-002 | Admin `/login`, Cleaner `/login`, Chatbot `/login` | JWT, bcrypt, RBAC, Rate Limit, PostgreSQL 原子帳號鎖定, 密碼政策 | ✅ |
| US-01-01 | 註冊 | `POST /api/auth/register` | auth/ | auth.service.spec, register.dto.spec | — | — | Chatbot `/login` | JWT, bcrypt, RBAC, Rate Limit, 帳號鎖定, 密碼政策 | ✅ |
| US-01-01 | Token 刷新與獨立工作階段 | `POST /api/auth/refresh` | auth/ | auth.service.spec、token.service.spec、auth-session.repository.spec、jwt.strategy.spec | `scripts/verify-migration-0009.sh`、`auth-session-postgres.integration.ts` | TC-01-003、TC-01-004、TC-01-007 | Admin／Cleaner／Chatbot 自動刷新；Web Locks／Bakery-style 競爭者租約協調 | JWT `sid`/`jti`/`tokenUse`、bcrypt refresh hash、原子輪替、最多 10 個 session | ✅ |
| US-01-01 | 密碼變更 | `POST /api/auth/change-password` | auth/ | auth.service.spec, change-password.dto.spec | — | — | Admin `/change-password`, Cleaner `/change-password`, Chatbot `/change-password` | JWT, bcrypt, RBAC, Rate Limit, 帳號鎖定, 密碼政策 | ✅ |
| US-01-02 | 帳號管理 | `GET/POST/PUT /api/users` | users/ | users.service.spec, update-role.dto.spec, update-status.dto.spec | — | — | Admin `/users` | JWT, bcrypt, RBAC, Rate Limit, 帳號鎖定, 密碼政策 | ✅ |
| US-01-03 | 目前／全部登出 | `POST /api/auth/logout`、`POST /api/auth/logout-all` | auth/ | auth.service.spec、auth.controller.spec、jwt.strategy.spec | — | TC-01-003 | 全應用；一般登出不影響其他 session | JWT `sid`、`tokenVersion`、Session 撤銷 | ✅ |
| US-01-03 | Cleaner SSO 獨立 Session | `POST /api/auth/sso/exchange` | auth/ | auth.service.spec、auth.controller.spec、Cleaner useAuth.test | — | TC-01-005 | Admin 開啟 Cleaner；只傳 Access Token | JWT、RBAC、獨立 Cleaner Refresh Token | ✅ |
| US-01-03 | 前端不中斷復原 | 自動 Refresh／同頁重新登入 | Admin／Cleaner／Chatbot | 三端 client.test、useAuth.test、SessionRecovery.test；Chatbot ChatInput.test、useChat.test | — | TC-01-004、TC-01-006、TC-01-007 | 保留頁面、草稿、對話 ID；鎖定原帳號；不自動 reload | 僅 Refresh 400／401 失效；網路／429／5xx 保留憑證；跨帳號不覆寫 | ✅ |

**E2E 覆蓋**：`app.e2e-spec.ts`（登入/登出/Token 流程）

---

### FR-02：RAG 智慧問答 ✅

| US | 說明 | API 端點 | NestJS 模組 | NestJS 測試 | Python 測試 | TC | Screen | Security | 狀態 |
|----|------|----------|------------|-------------|-------------|-----|--------|----------|------|
| US-02-01 | 自然語言資安查詢 | `POST /api/chat` | chat/ | chat.service.spec, rag-proxy.service.spec, context-builder.service.spec | test_rag_chain, test_retriever | TC-02-001 | Chatbot 主畫面 | JWT, Input Validation, Rate Limit | ✅ |
| US-02-01 | SSE 串流 | `POST /api/chat`（SSE） | chat/ | chat.service.spec | — | — | Chatbot 主畫面 | JWT, Input Validation, Rate Limit | ✅ |
| US-02-02 | 引用來源顯示 | `POST /api/chat`（response.sources） | chat/ | context-builder.service.spec | test_retriever | — | Chatbot 主畫面 | JWT, Input Validation, Rate Limit | ✅ |
| US-02-03 | 歷史對話管理 | `GET /api/chat/conversations` | chat/ | chat.service.spec | — | — | Chatbot 主畫面 | JWT, Input Validation, Rate Limit | ✅ |
| US-02-04 | 多輪對話 + Query 改寫 | `POST /api/chat`（history injection） | chat/ | query-preprocessor.service.spec | test_prompts | — | Chatbot 主畫面 | JWT, Input Validation, Rate Limit | ✅ |
| US-02-05 | 分數過濾與品質保障 | `POST /api/v1/rag/retrieve`（閾值） | chat/ + rag-service | rag-proxy.service.spec | test_retriever, test_rag_chain | — | Chatbot 主畫面 | JWT, Input Validation, Rate Limit | ✅ |
| US-02-06 | 資安議題解析、提醒與延續脈絡 | `POST /api/chat`, `POST /api/chat/stream`（`topicScope`） | chat/ + prompts/ + websearch/ | topic-classifier.service.spec、topic-classifier.live-eval.spec、chat.dto.spec、query-preprocessor.service.spec、context-builder.service.spec、rag-proxy.service.spec、message.repository.spec、chat.service.spec、chat.controller.spec、prompts.service.spec、prompt-renderer.util.spec、public-url-safety.service.spec、web-fetcher.service.spec、Chatbot `MessageList.test` | `test:chat-topic-bridge-integration` | AC-02-06-01~08；TC-02-003 | Chatbot `MessageBubble`／建議問題按鈕；Admin 測試提示詞 | JWT、Input Validation、Prompt Injection Boundary、SSRF 防護、conversation row lock | ✅ |
| US-02-07 | 最新回覆重新產出與可靠重試 | `POST /api/chat/stream`（`messageAttemptId`；重新產出另用 `regenerateFromMessageId`、`regenerationAttemptId`） | chat/ | chat.dto.spec、message.repository.spec、context-builder.service.spec、chat.service.spec、chat.controller.spec、Chatbot `useChat.test`／`MessageList.test` | `test:chat-regeneration-integration`、`test:chat-topic-bridge-integration` | TC-02-002、TC-02-004 | Chatbot 最新 `MessageBubble`、失敗重試 | JWT、Input Validation、user／conversation row lock、attempt 冪等、lease fencing、原子額度 | ✅ |
| — | 網路搜尋補充 | `GET /api/websearch/config` | websearch/ | searxng.service.spec, websearch-config.service.spec, web-fetcher.service.spec | — | — | Chatbot 主畫面, Admin `/settings` | JWT, Input Validation, Rate Limit | ✅ |
| — | 三層回應層級切換 | `GET /api/chat/modes` | chat/ | chat.dto.spec | — | — | Chatbot 主畫面, Admin `/settings` | JWT, Input Validation, Rate Limit | ✅ |

**E2E 覆蓋**：`app.e2e-spec.ts`（聊天流程）

---

### FR-03：提示詞管理 🔄

| US | 說明 | API 端點 | NestJS 模組 | NestJS 測試 | Python 測試 | TC | Screen | Security | 狀態 |
|----|------|----------|------------|-------------|-------------|-----|--------|----------|------|
| US-03-01 | 三層提示詞管理 | `GET/PUT /api/prompts` | prompts/ | create-prompt.dto.spec, update-prompt.dto.spec, prompt-renderer.util.spec | — | — | Admin `/prompts` 固定三筆 | JWT, RBAC(admin), Input Validation | ✅ |
| US-03-02 | 回應層級差異化 | （提示詞+chat 配合） | prompts/ + chat/ | chat.service.spec, context-builder.service.spec | test_prompts | — | Admin `/prompts` | JWT, RBAC(admin), Input Validation | ✅ |
| US-03-03 | 提示詞測試 | `POST /api/prompts/test` | prompts/ | test-prompt.dto.spec, prompts.service.spec（含等價性 golden test） | — | — | Admin `/prompts` | JWT, RBAC(admin), Input Validation | ✅ |
| US-03-04 | mode 直接對應提示詞 | `GET /api/prompts?mode=` | prompts/ + chat/ | prompts.service.spec, chat.service.spec | — | — | Admin `/prompts` | JWT, RBAC(admin), 每 mode 唯一啟用索引 | ✅ |

---

### FR-04：資料去識別化 ✅

| US | 說明 | API 端點 | NestJS 模組 | NestJS 測試 | Python 測試 | TC | Screen | Security | 狀態 |
|----|------|----------|------------|-------------|-------------|-----|--------|----------|------|
| US-04-01 | PII 自動偵測 | `POST /api/v1/clean` | cleaning/ → FastAPI | clean.controller.spec, cleaning-proxy.service.spec | test_detector, test_recognizers, test_name_context_integration | TC-04-001 | Admin `/` | JWT, RBAC(admin), X-Internal-Token, PII 處理 | ✅ |
| US-04-02 | 去識別化策略 | `POST /api/v1/clean`（strategy 參數） | cleaning/ → FastAPI | clean.controller.spec | test_anonymizer, test_strategies, test_anonymizer_overlap, test_encrypt_key_rotation | — | Admin `/` | JWT, RBAC(admin), X-Internal-Token, PII 處理 | ✅ |
| US-04-03 | 固定規則套用 | `POST /api/v1/clean`（自動套用 `get_default_rules()`） | cleaning/ → FastAPI | clean.controller.spec | test_clean_api, test_default_rules | — | Admin `/` | JWT, RBAC(admin), X-Internal-Token, PII 處理 | ✅ |
| US-04-04 | 批次處理 | `POST /api/v1/clean`（多檔） | cleaning/ → FastAPI | clean.controller.spec | test_clean_api | — | Admin `/` | JWT, RBAC(admin), X-Internal-Token, PII 處理 | ✅ |

**Python data-pipeline 測試**：test_parsers, test_text_parser, test_excel_parser, test_loaders_binary, test_language_detector, test_relation_keeper

---

### FR-05：檔案上傳與下載 ✅

| US | 說明 | API 端點 | NestJS 模組 | NestJS 測試 | Python 測試 | TC | Screen | Security | 狀態 |
|----|------|----------|------------|-------------|-------------|-----|--------|----------|------|
| US-05-01 | 多格式檔案上傳 | `POST /api/v1/upload` | cleaning/ → FastAPI | upload.controller.spec、Admin `FileUploader.test`、`client.test`、`HomePage.test`、`LoginPage.test`、`useAuth.test`、`uploadFeedback.test` | test_upload_api, test_upload_integration | TC-03-001 | Admin `/`；格式、單檔／整批 50 MB 前置檢查，失敗原因持續以中文顯示；Refresh 失效時同頁重新登入，暫時性驗證錯誤保留已選檔案 | JWT, RBAC(admin), 50MB 檔案批次／51MB multipart 封裝限制, Input Validation | ✅ |
| US-05-02 | 清洗結果下載 | `GET /api/v1/download/:taskId` | cleaning/ → FastAPI | download.controller.spec、Admin `client.test`、`TaskDownloadButton.test` | `test_download_api`（含 report 固定路由優先序） | — | Admin 首頁完成步驟、`/tasks`；Bearer fetch → Blob 下載，與 Cleaner 審核分離 | JWT, RBAC(admin/data_cleaner), Input Validation | ✅ |
| US-05-03 | ZIP 上傳 | `POST /api/v1/upload`（ZIP 自動解壓+auto_tags） | cleaning/ → FastAPI | upload.controller.spec、Admin `client.test`、`HomePage.test`、`uploadFeedback.test` | test_upload_api, test_upload_integration | AC-05-03-01~04 | Admin `/`；完全未匯入停在步驟 1，部分成功進入步驟 2 並顯示中文警告 | JWT, RBAC(admin), 上傳檔 50MB／解壓總量環境限制, Input Validation | ✅ |

---

### FR-06：即時通知 ✅

| US | 說明 | API 端點 | NestJS 模組 | NestJS 測試 | Python 測試 | TC | Screen | Security | 狀態 |
|----|------|----------|------------|-------------|-------------|-----|--------|----------|------|
| US-06-01 | WebSocket 任務進度 | `ws://` WebSocket gateway | websocket/ | — | test_ws_manager | — | Admin `/tasks` | JWT, WebSocket 認證 | ✅ |

---

### FR-07：稽核日誌 ✅

| US | 說明 | API 端點 | NestJS 模組 | NestJS 測試 | Python 測試 | TC | Screen | Security | 狀態 |
|----|------|----------|------------|-------------|-------------|-----|--------|----------|------|
| US-07-01 | 操作記錄查詢 | `GET /api/audit-logs` | audit/ | audit-query.dto.spec, audit-export.dto.spec, audit-log.interceptor.spec | test_audit_repo | — | Admin `/audit-logs` | JWT, RBAC(admin) | ✅ |
| US-07-01 | CSV/JSON 匯出 | `GET /api/audit-logs/export` | audit/ | audit-export.dto.spec | — | — | Admin `/audit-logs` | JWT, RBAC(admin) | ✅ |

---

### FR-08：去識別化規則管理 — 已簡化

> **變更說明**：原規劃為自訂規則 CRUD API，已簡化為固定規則模式（`get_default_rules()`）。原有的 `rules.controller.spec`、`test_rules_api` 測試已移除，由 `test_default_rules` 覆蓋固定規則邏輯。

| US | 說明 | API 端點 | NestJS 模組 | NestJS 測試 | Python 測試 | TC | Screen | Security | 狀態 |
|----|------|----------|------------|-------------|-------------|-----|--------|----------|------|
| US-08-01 | 固定去識別化規則 | `POST /api/v1/clean`（自動套用） | cleaning/ → FastAPI | clean.controller.spec | test_default_rules, test_clean_api | — | Admin `/`（自動套用） | JWT, RBAC(admin), X-Internal-Token | ✅ |

---

### FR-09：任務管理 ✅

| US | 說明 | API 端點 | NestJS 模組 | NestJS 測試 | Python 測試 | TC | Screen | Security | 狀態 |
|----|------|----------|------------|-------------|-------------|-----|--------|----------|------|
| US-09-01 | 任務生命週期 | `GET/POST /api/v1/tasks` | cleaning/ → FastAPI | tasks.controller.spec | test_tasks_api | — | Admin `/tasks` | JWT, RBAC(admin), X-Internal-Token | ✅ |
| US-09-02 | 任務命名、快速改名與台灣時間 | `POST /api/v1/clean`、`PATCH /api/v1/tasks/:taskId/name` | cleaning/ → FastAPI | clean/tasks controller specs、Admin TaskManager/HomePage tests | task API integration、clean API、migration 012 verification | TC-09-001 | Admin `/`、`/tasks` | JWT、Admin-only RBAC、Input Validation、Audit、UTC contract | ✅ |

---

### FR-16：三層式回應機制 🔄

| US | 說明 | API 端點 | NestJS 模組 | NestJS 測試 | Python 測試 | TC | Screen | Security | 狀態 |
|----|------|----------|------------|-------------|-------------|-----|--------|----------|------|
| US-16-01 | 角色預設回應層級 | `GET /api/chat/modes` | chat/ | chat.controller.spec, chat-mode.guard.spec | — | — | Chatbot 主畫面 | JWT, fail-closed role matrix | ✅ |
| US-16-02 | 模式切換+差異化 | `POST /api/chat`（mode 參數） | chat/ + llm/ | chat.service.spec, llm.service.spec, chatbot useChat.test（前端不覆蓋 topK） | test_llm_temperature, test_retriever | — | Chatbot 主畫面, Admin `/prompts` | JWT, Input Validation | ✅ |
| US-16-03 | 回應層級提示詞管理 | `GET/PUT /api/prompts` | prompts/ | prompts.service.spec | test_prompts | — | Admin `/prompts` | JWT, Input Validation | ✅ |
| US-16-05 | 新手每日訊息額度 | `GET /api/chat/modes`, `GET/PUT /api/chat/usage-config`, `POST /api/chat/stream` | chat/ | chat-usage.service.spec, chat.service.spec, ChatInput.test.tsx | — | — | Chatbot 輸入區, Admin `/settings` | JWT, RBAC(admin), 429, 原子計數 | ✅ |

---

### FR-17：在地化法規知識庫 ✅/🔄

| US | 說明 | API 端點 | NestJS 模組 | NestJS 測試 | Python 測試 | TC | Screen | Security | 狀態 |
|----|------|----------|------------|-------------|-------------|-----|--------|----------|------|
| US-17-01 | 法規知識查詢 | `POST /api/v1/rag/retrieve` + `POST /api/chat` | chat/ + rag-service | rag-proxy.service.spec | test_retriever, test_rag_chain, test_chunker, test_preprocessor | — | Chatbot 主畫面 | JWT, X-Internal-Token | ✅ |

**RAG 基礎建設測試**：test_bm25_store, test_embedding_cache, test_embedding_factory, test_llm_factory, test_qdrant_store_browse, test_config, test_models, test_schemas

---

### FR-18：清洗審核管理 — Cleaner App ✅

| US | 說明 | API 端點 | NestJS 模組 | NestJS 測試 | Python 測試 | TC | Screen | Security | 狀態 |
|----|------|----------|------------|-------------|-------------|-----|--------|----------|------|
| US-18-01 | 待審核任務瀏覽 | `GET /api/v1/review/:taskId` | cleaning/ → FastAPI | review.controller.spec, review.dto.spec | test_review_api | TC-05-001 | Cleaner `/tasks` | JWT, RBAC(admin/cleaner/reviewer), X-Internal-Token, PII 隔離 | ✅ |
| US-18-02 | 檔案內容檢視 | `GET /api/v1/review/:taskId/files/:fileId/content` | cleaning/ → FastAPI | review.controller.spec | test_review_api | — | Cleaner `/tasks/:taskId/files/:fileId` | JWT, RBAC(admin/cleaner/reviewer), X-Internal-Token, PII 隔離 | ✅ |
| US-18-03 | 內容手動修正 | `PUT /api/v1/review/:taskId/files/:fileId/content` | cleaning/ → FastAPI | review.controller.spec | test_review_api | TC-05-002 | Cleaner `/tasks/:taskId/files/:fileId` | JWT, RBAC(admin/cleaner/reviewer), X-Internal-Token, PII 隔離 | ✅ |
| US-18-04 | 標籤管理 | `PUT /api/v1/review/:taskId/files/:fileId/tags` | cleaning/ → FastAPI | review.controller.spec | test_review_api | TC-05-003 | Cleaner `/tasks/:taskId/files/:fileId` | JWT, RBAC(admin/cleaner/reviewer), X-Internal-Token, PII 隔離 | ✅ |
| US-18-05 | 任務批准/駁回 | `POST /api/v1/review/:taskId/approve` | cleaning/ → FastAPI | review.controller.spec | test_review_api, test_review_integration | TC-05-004 | Cleaner `/tasks/:taskId` | JWT, RBAC(admin/cleaner/reviewer), X-Internal-Token, PII 隔離, Maker-Checker | ✅ |
| US-18-06 | 送入 RAG 知識庫 | `POST /api/v1/review/:taskId/ingest` | cleaning/ → FastAPI → rag-service | review.controller.spec | test_review_integration、test_regulation_versioning、test_regulations_integration、test_qdrant_indexed_at、test_retriever | TC-05-005 | Cleaner `/tasks/:taskId/ingest` | JWT, RBAC(admin/reviewer), X-Internal-Token, PII 隔離、任務級補償、法規有效期跨儲存同步、可重試 | ✅ |
| US-18-07 | 來源資料瀏覽 | `GET /api/v1/files` | cleaning/ → FastAPI | files.controller.spec, Cleaner FilesPage tests | test_upload_integration, test_source_file_filter_postgres_integration | TC-20-003 | Cleaner `/files` | JWT, RBAC(admin/cleaner/reviewer), X-Internal-Token, PII 隔離 | ✅ |
| US-18-08 | 知識庫文件瀏覽 | `GET /api/v1/knowledge-base/documents/*` | cleaning/ → FastAPI → rag-service | knowledge-base.controller.spec | test_knowledge_base_api, test_knowledge_base_integration | — | Cleaner `/knowledge-base` | JWT, RBAC(admin/cleaner/reviewer), X-Internal-Token, PII 隔離 | ✅ |
| US-18-09 | 知識庫文件刪除 | `DELETE /api/v1/knowledge-base/documents/by-source` | cleaning/ → FastAPI → rag-service | knowledge-base.controller.spec | test_knowledge_base_api | — | Cleaner `/knowledge-base` | JWT, RBAC(admin/cleaner/reviewer), X-Internal-Token, PII 隔離 | ✅ |
| US-18-10 | 送審任務 | `POST /api/v1/review/:taskId/submit` | cleaning/ → FastAPI | review.controller.spec | test_review_api | TC-05-006 | Cleaner `/tasks/:taskId` | JWT, RBAC(admin/cleaner), X-Internal-Token, Maker-Checker | ✅ |
| US-18-11 | 退回任務 | `POST /api/v1/review/:taskId/reject` | cleaning/ → FastAPI | review.controller.spec | test_review_api | TC-05-007 | Cleaner `/tasks/:taskId` | JWT, RBAC(admin/reviewer), X-Internal-Token, Maker-Checker | ✅ |
| US-18-12 | Maker-Checker 職責分離 | submit/approve/reject 端點聯動 | cleaning/ → FastAPI | review.controller.spec | test_review_api, test_review_integration | TC-05-006, TC-05-007 | Cleaner `/tasks/:taskId` | JWT, RBAC, Maker-Checker（submitted_by ≠ approved_by） | ✅ |

---

### FR-19：資料分析儀表板 ✅

| US | 說明 | API 端點 | NestJS 模組 | NestJS 測試 | Python 測試 | TC | Screen | Security | 狀態 |
|----|------|----------|------------|-------------|-------------|-----|--------|----------|------|
| US-19-01 | 清洗統計 | `GET /api/v1/analytics/cleaning` | cleaning/ → FastAPI | analytics.controller.spec, analytics.dto.spec | test_analytics_api | TC-06-001 | Cleaner `/` | JWT, RBAC(admin/cleaner), X-Internal-Token | ✅ |
| US-19-02 | 知識庫統計 | `GET /api/v1/analytics/knowledge-base` | cleaning/ → FastAPI | analytics.controller.spec | test_analytics_api | — | Cleaner `/` | JWT, RBAC(admin/cleaner), X-Internal-Token | ✅ |
| US-19-03 | 時間軸統計 | `GET /api/v1/analytics/timeline` | cleaning/ → FastAPI | analytics.controller.spec | test_analytics_api | TC-06-002 | Cleaner `/` | JWT, RBAC(admin/cleaner), X-Internal-Token | ✅ |

---

### FR-20：來源資料瀏覽 ✅

| US | 說明 | API 端點 | NestJS 模組 | NestJS 測試 | Python 測試 | TC | Screen | Security | 狀態 |
|----|------|----------|------------|-------------|-------------|-----|--------|----------|------|
| US-20-01 | 來源檔案列表、最新任務名稱、審核狀態、穩定分頁與台北時間 | `GET /api/v1/files` | cleaning/ → FastAPI | files.controller.spec, Cleaner FilesPage tests | test_upload_integration, test_source_file_filter_postgres_integration | TC-20-002, TC-20-003 | Cleaner `/files`、`/tasks`、審核頁、知識庫 | JWT, RBAC(admin/cleaner), X-Internal-Token；UTC 傳輸 | ✅ |
| US-20-02 | 待審核來源影響確認、邏輯刪除與刪除歷史查閱 | `GET /api/v1/files?pipeline_status=deleted`；`GET /api/v1/files/:id/deletion-impact`；`DELETE /api/v1/files/:id` | cleaning/ → FastAPI `SourceFileDeletionService` | files.controller.spec, cleaning-authz.spec, source-file.dto.spec | test_source_file_deletion_service, test_upload_integration, test_review_integration, test_source_file_deletion_postgres_integration | TC-20-001 | Cleaner `/files`, `/tasks/:taskId` | JWT Admin-only、操作者注入、File/Task/TaskFile row lock、全關聯檢查、Qdrant fail-closed、理由與確認文字、操作紀錄、禁止實體刪除 | ✅ |

---

### FR-21：知識庫文件瀏覽 ✅

| US | 說明 | API 端點 | NestJS 模組 | NestJS 測試 | Python 測試 | TC | Screen | Security | 狀態 |
|----|------|----------|------------|-------------|-------------|-----|--------|----------|------|
| US-21-01 | 知識庫文件聚合 | `GET /api/v1/knowledge-base/documents/grouped` | cleaning/ → FastAPI → rag-service | knowledge-base.controller.spec | test_knowledge_base_api, test_knowledge_base_integration | — | Cleaner `/knowledge-base` | JWT, RBAC(admin/cleaner), X-Internal-Token | ✅ |
| US-21-01 | 依來源篩選 | `GET /api/v1/knowledge-base/documents/by-source` | cleaning/ → FastAPI → rag-service | knowledge-base.controller.spec | test_knowledge_base_api | — | Cleaner `/knowledge-base` | JWT, RBAC(admin/cleaner), X-Internal-Token | ✅ |

---

### FR-22：回答品質回饋 ✅

| US | 說明 | API 端點 | NestJS 模組 | NestJS 測試 | Python 測試 | TC | Screen | Security | 狀態 |
|----|------|----------|------------|-------------|-------------|-----|--------|----------|------|
| US-22-01 | 回答品質回饋 | `POST /api/chat/messages/:messageId/feedback` | chat/ | chat.service.spec | — | — | Chatbot 主畫面 | JWT, Input Validation | ✅ |

---

### FR-23：RAG 信心度指示器 ✅

| US | 說明 | API 端點 | NestJS 模組 | NestJS 測試 | Python 測試 | TC | Screen | Security | 狀態 |
|----|------|----------|------------|-------------|-------------|-----|--------|----------|------|
| US-23-01 | 信心度彩色標籤 | `POST /api/chat`（response.confidenceLevel） | chat/ | chat.service.spec | — | — | Chatbot 主畫面 | JWT | ✅ |

---

### FR-24：法規／知識分類治理 ✅

| US | 說明 | API 端點 | NestJS 模組 | NestJS 測試 | Python 測試 | TC | Screen | Security | 狀態 |
|----|------|----------|------------|-------------|-------------|-----|--------|----------|------|
| US-24-01 | 批次必選法規／知識類型 | `PATCH /api/v1/upload/:id/metadata`、`POST /api/v1/clean` | cleaning/ | cleaning controllers 既有代理／DTO測試、Admin `regulationTypes.test` | `test_clean_api`、`test_upload_integration`、`test_regulation_type_contract`、`test_contract_parity`、`verify-migration-011.sh` | AC-24-01-01～08 | Admin 首頁步驟 2 | JWT、RBAC、雙層輸入驗證、跨語言契約 parity、檔案列鎖、任務建立後分類不可變 | ✅ |

---

## 2. 追溯矩陣 — 未來功能

| FR | 說明 | 狀態 | 目標階段 |
|----|------|------|---------|
| FR-10 | 進階 RBAC 權限管理 | 🔮 | Phase 3 |
| FR-11 | 多會員架構 | 🔮 | Phase 3 |
| FR-12 | 多知識庫管理 | 🔮 | Phase 3 |
| FR-13 | 智慧分析儀表板 | 🔮 | Phase 3 |
| FR-14 | 第三方整合 | 🔮 | Phase 3 |
| FR-15 | 運維監控與告警 | 🔮 | Phase 3 |

---

## 3. 驗收測試追溯（SRS_TECHNICAL.md §11）

| TC 編號 | 對應 FR | 對應 US | 說明 | NestJS 測試 | Python 測試 | Screen | Security | 狀態 |
|---------|---------|---------|------|-------------|-------------|--------|----------|------|
| TC-01-001 | FR-01 | US-01-01 | 正常登入 | auth.service.spec + E2E | — | — | JWT, bcrypt, 帳號鎖定 | ✅ |
| TC-01-002 | FR-01 | US-01-01 | 登入失敗（帳密錯誤/鎖定）；5 個並行失敗請求不遺失計數 | auth.service.spec、`pnpm --filter @oda-cyber/api test:auth-lock-integration` | PostgreSQL 實際驗證 `login_attempts=5` 與未來 `locked_until` | — | JWT, bcrypt, Rate Limit、原子帳號鎖定 | ✅ |
| TC-01-003 | FR-01 | US-01-01／US-01-03 | 兩個 session 獨立、Refresh 維持 `sid`、目前登出不影響另一 session | auth.service.spec、auth-session.repository.spec、jwt.strategy.spec | `scripts/verify-migration-0009.sh` | 本機 API 雙 session 實測 | JWT `sid`/`jti`、原子輪替 | ✅ |
| TC-01-004 | FR-01 | US-01-03 | Refresh 401 才清 Token；5xx／網路錯誤保留頁面與使用者資料 | Admin／Cleaner／Chatbot `client.test`、`useAuth.test` | — | — | 錯誤分類、 bounded retry | ✅ |
| TC-01-005 | FR-01 | US-01-03 | Admin Access Token 交換為獨立 Cleaner Session | auth.service.spec、auth.controller.spec、Cleaner `useAuth.test` | — | — | SSO、RBAC、Refresh Token 不跨應用傳遞 | ✅ |
| TC-01-006 | FR-01 | US-01-03 | Chatbot 草稿與目前對話 ID 在同頁重新登入期間保留 | `ChatInput.test`、`useChat.test`、`useAuth.test` | — | — | `sessionStorage`、寫入停用 | ✅ |
| TC-01-007 | FR-01 | US-01-01／US-01-03 | Refresh Token 不可作為 Access Token；併發輪替僅一個成功；舊認證協定 Refresh 回 426 且不消耗 Token；所有認證寫入共用跨分頁互斥鎖；請求或 JSON 解析途中跨帳號切換不得覆寫；末段插入舊版租約或 lease claim 未讀回自己 owner 時重新競爭；誤登入新工作階段撤銷、稍後處理與唯讀原帳號 | `token.service.spec`、`jwt.strategy.spec`、`auth.controller.spec`、三端 `client.test`／`crossTabMutex.test`／`useAuth.test`／`SessionRecovery.test` | `pnpm --filter @oda-cyber/api test:auth-session-integration` 查驗舊協定 426、PostgreSQL `current_jti`、hash、`revoked_at`；Chromium 禁用 Web Locks 的雙分頁互斥測試 | 桌面／375px 手機 Playwright 6 組情境 | `tokenUse`、協定 fencing、CAS 輪替、同帳號防護 | ✅ |
| TC-02-001 | FR-02 | US-02-01 | RAG 查詢回應 | chat.service.spec, rag-proxy.service.spec | test_rag_chain, test_retriever | — | JWT, Input Validation | ✅ |
| TC-02-002 | FR-02 | US-02-07 | 重新產出只作用於相同最後提示詞、資安／混合主題與最新完成回覆；雙 attempt 並發僅一個成功；有效 lease processing 回 409，逾時／failed／completed 重試不重複計費 | API chat DTO／repository／context／service／controller／usage specs；Chatbot `useChat.test`、`MessageList.test` | `pnpm --filter @oda-cyber/api test:chat-regeneration-integration` 實際驗證 PostgreSQL row lock、提示詞防竄改、`chat_daily_usages` 與 claim 同交易回滾、單調時間、並發 loser、lease 逾時接手、舊 worker metadata／寫入／失敗釋放 fencing、跨分頁 failed 釋放、claim completed 與完成回放 | icon-only、hover 提示、額度停用、最新失敗重試、UUID 正向驗證 | JWT、topicScope、相同提示詞、row lock、attempt 冪等、lease fencing、原子額度、in-flight lock | ✅ |
| TC-02-003 | FR-02 | US-02-06 | 非資安／不明確初判先查 RAG；門檻等值可依知識庫回答、門檻減 ε 不可；`SGS`／`TUV`／`TAF` 可由知識庫或安全且非空的網搜內容恢復；網搜回答由 API 固定加推測前綴；`ODA` 後接「資安服務」只合併一輪且併發只消耗一次；帶一般訊息 attempt 時 bridge 必須在 user claim 建立前於同交易消耗並保存，failed／逾時重試沿用；無問號完整敘述不合併；RAG payload 異常與服務不可用都禁止網搜推測；已完成 SSE 回答先落盤再更新建議題，`done` 遺失時復原伺服器回答且不可重送；建議題失敗時仍回資安預設題；執行期資料不進 system message | `chat.service.spec`、`chat.controller.spec`、`chat.dto.spec`、`rag-proxy.service.spec`、`message.repository.spec`、`query-preprocessor.service.spec`、`context-builder.service.spec`、`prompts.service.spec`、`prompt-renderer.util.spec`、`public-url-safety.service.spec`、`web-fetcher.service.spec`、`llm.service.spec`、Chatbot `useChat.test`、`MessageList.test` | `pnpm --filter @oda-cyber/api run test:chat-topic-bridge-integration` 建立隔離 fixture 並以 PostgreSQL blocker 確認兩個 consume 同時等待 row lock，釋放後僅一個成功並落盤 `topicBridgeConsumedAt`；一般訊息 claim 另驗證 bridge 於建立 user message 前消耗並保存；外部 LLM／SearXNG smoke 須明確啟用，不列入預設閘門 | 建議問題按鈕送出完整文字；SSE 缺少 `done` 時先復原伺服器回答，狀態查詢失敗時 fail-closed；既有版面不變；後台 `rendered`／`runtimeData` 等於正式 system／user data message | Input Validation、Prompt Injection Boundary、SSRF／DNS rebinding／redirect 防護、Node `all=true` lookup、row lock 單次消耗 | ✅ |
| TC-02-004 | FR-02 | US-02-07 | 一般訊息帶 `messageAttemptId`；首次請求原子建立必要對話、消耗並保存 topic bridge、建立 user claim 與額度；相同 attempt 有效 lease 回 409，failed／逾時接手不重複扣額且沿用 bridge，completed 回放同一回答；舊 lease 不得寫入或釋放新 claim | `chat.dto.spec`、`message.repository.spec`、`chat.service.spec`、`chat.controller.spec`、Chatbot `useChat.test` | `pnpm --filter @oda-cyber/api run test:chat-topic-bridge-integration` 以隔離使用者、兩個 Prisma client 與 PostgreSQL blocker 驗證兩個 claim 同時等待 user row lock；釋放後僅一個在同交易消耗 bridge、建立訊息並寫入一次真實 `chat_daily_usages`，另一個回處理中衝突；強制 callback 失敗時驗證訊息與額度一併回滾 | 每次一般送出產生 UUID；重試與歷史復原依同一 UUID 配對，不以問題文字猜測；狀態查詢失敗時 fail-closed | JWT、Input Validation、user／conversation row lock、attempt 冪等、lease fencing、原子額度 | ✅ |
| TC-03-001 | FR-05 | US-05-01 | 檔案上傳（含登入失效、批次容量、完全失敗、部分成功與中文原因） | upload.controller.spec、Admin `FileUploader.test`、`client.test`、`HomePage.test`、`LoginPage.test`、`useAuth.test`、`uploadFeedback.test` | test_upload_api | — | JWT, RBAC(admin), 50MB 檔案批次／51MB multipart 封裝限制 | ✅ |
| TC-04-001 | FR-04 | US-04-01 | 清洗任務執行 | clean.controller.spec | test_clean_api, test_detector | — | JWT, RBAC(admin), X-Internal-Token | ✅ |
| TC-05-001 | FR-18 | US-18-01 | 清洗審核瀏覽 | review.controller.spec | test_review_api | — | JWT, RBAC(admin/cleaner), X-Internal-Token | ✅ |
| TC-05-002 | FR-18 | US-18-03 | 內容手動編輯 | review.controller.spec | test_review_api | — | JWT, RBAC(admin/cleaner), X-Internal-Token | ✅ |
| TC-05-003 | FR-18 | US-18-04 | 標籤管理 | review.controller.spec | test_review_api | — | JWT, RBAC(admin/cleaner), X-Internal-Token | ✅ |
| TC-05-004 | FR-18 | US-18-05 | 任務批准 | review.controller.spec | test_review_integration | — | JWT, RBAC(admin/cleaner), X-Internal-Token | ✅ |
| TC-05-005 | FR-18 | US-18-06 | 送入 RAG；部分檔案或 DB commit 失敗時完整補償 Qdrant/BM25、還原舊法規向量有效期並維持 approved 可重試；歷史回補不重疊，舊／同日法規不取代現行版；人工取代同步 Qdrant；hybrid BM25 不繞過有效期 | review.controller.spec | test_review_integration、test_regulation_versioning、test_regulations_integration、test_qdrant_indexed_at、test_retriever | Cleaner 失敗時留在送入頁可重試 | JWT, RBAC(admin/reviewer), X-Internal-Token、跨儲存補償 | ✅ |
| TC-05-006 | FR-18 | US-18-10, US-18-12 | Maker-Checker 送審與職責分離 | review.controller.spec | test_review_api, test_review_integration | — | JWT, RBAC, Maker-Checker | ✅ |
| TC-05-007 | FR-18 | US-18-11 | 退回任務流程 | review.controller.spec | test_review_api | — | JWT, RBAC(admin/reviewer), Maker-Checker | ✅ |
| TC-06-001 | FR-19 | US-19-01 | 清洗統計 | analytics.controller.spec | test_analytics_api | — | JWT, RBAC(admin/cleaner), X-Internal-Token | ✅ |
| TC-06-002 | FR-19 | US-19-03 | 時間軸統計 | analytics.controller.spec | test_analytics_api | — | JWT, RBAC(admin/cleaner), X-Internal-Token | ✅ |
| TC-09-001 | FR-09 | US-09-02 | 建立命名任務、所有狀態快速改名、舊任務 fallback 與台灣時間 | clean/tasks controller specs、Admin TaskManager/HomePage tests | task/clean integration、migration 012 verification | Admin `/tasks` | Admin-only RBAC、Audit、UTC contract | ✅ |
| TC-20-001 | FR-20 | US-20-02 | 管理員預檢並邏輯刪除未送審或待審核來源；阻擋非管理員、其他審核狀態、任一關聯已批准／已入庫／處理中、逐檔未通過、Qdrant 有切塊或不可用；刪除後由專用篩選查閱 | files.controller.spec、cleaning-authz.spec、source-file.dto.spec、Cleaner FilesPage／TaskReviewPage／client tests | test_source_file_deletion_service、test_upload_integration、test_review_integration、test_source_file_deletion_postgres_integration | Cleaner `/files` 預檢、已刪除篩選與 `/tasks/:taskId` 歷史標示 | Admin-only、交易 row lock、JWT actor、Qdrant fail-closed、操作紀錄、邏輯刪除 | ✅ |
| TC-20-002 | FR-20 | US-20-01 | UTC、無 offset UTC 與跨日時間轉換；任務列表與各 Cleaner 頁面共用格式 | Cleaner dateTime、FilesPage、TaskListPage、TaskReviewPage、FileReviewPage、KnowledgeBasePage tests | — | Cleaner 來源、任務、審核、首頁活動、知識庫頁面 | UTC API contract、明確 `Asia/Taipei` 顯示 | ✅ |
| TC-20-003 | FR-20 | US-18-07, US-20-01 | `pipeline_status` 代理、無效值 422、SQL 狀態矩陣、25 筆資料先篩選後計數／分頁、逐檔未通過不覆寫未送審、同時間任務以 `id DESC` 決勝 | files.controller.spec、Cleaner FilesPage tests | test_upload_integration、test_source_file_deletion_service、test_source_file_filter_postgres_integration | Cleaner `/files` 選擇「未送審」並跨頁查閱 | JWT、列舉驗證、外層交易 rollback、零 fixture 殘留 | ✅ |

---

## 4. 跨切面需求追溯

以下為非功能性需求與架構決策的追溯。

### 4.1 安全性需求

| 需求 | 實作機制 | 測試覆蓋 | ADR 參考 |
|------|---------|----------|----------|
| JWT 認證 | JwtAuthGuard 全域啟用 | auth.service.spec, E2E | — |
| RBAC 授權 | RolesGuard + @Roles 裝飾器 | roles.guard.spec | — |
| 密碼政策（資通安全「普」級） | IsStrongPassword validator | password-strength.validator.spec | [ADR-007](./architecture/ARCH.md#adr-007) |
| 內部 API 認證 | X-Internal-Token 中介軟體 | test_internal_auth_middleware | — |
| Rate Limiting | ThrottlerModule 60req/min | E2E | — |
| 輸入驗證 | ValidationPipe + class-validator | DTO spec 檔案（15+） | — |
| XSS 防護 | React 自動轉義 + helmet headers | — | — |
| SQL Injection | Prisma + SQLAlchemy 參數化查詢 | — | [ADR-003](./architecture/ARCH.md#adr-003) |

### 4.2 效能需求

| 需求 | 目標值 | 相關模組 | 測試狀態 |
|------|--------|---------|---------|
| RAG 查詢回應 | P95 < 10s | chat/ + rag-service | 規劃中（k6） |
| 檔案上傳速度 | 10MB/s | cleaning/ | 規劃中 |
| 清洗處理速度 | > 10 files/min | FastAPI | 規劃中 |
| 並發使用者 | 100 同時在線 | 全系統 | 規劃中 |

### 4.3 架構決策追溯

| ADR | 決策 | 影響的 FR | 參考 |
|-----|------|----------|------|
| ADR-001 | Monorepo (pnpm + Turborepo) | 全部 | [ARCH.md](./architecture/ARCH.md#adr-001) |
| ADR-002 | NestJS 統一 API Gateway | FR-04~09, FR-18~21 | [ARCH.md](./architecture/ARCH.md#adr-002) |
| ADR-003 | 雙 ORM 策略 | FR-01~09 + FR-04~09 | [ARCH.md](./architecture/ARCH.md#adr-003) |
| ADR-004 | Hybrid Search (Vector+BM25+RRF) | FR-02, FR-17 | [ARCH.md](./architecture/ARCH.md#adr-004) |
| ADR-005 | SSE 串流回應 | FR-02 | [ARCH.md](./architecture/ARCH.md#adr-005) |
| ADR-006 | 三前端應用 | FR-01~03, FR-04~09, FR-18~21 | [ARCH.md](./architecture/ARCH.md#adr-006) |
| ADR-007 | 資通安全「普」級密碼政策 | FR-01 | [ARCH.md](./architecture/ARCH.md#adr-007) |
| ADR-008 | SearXNG 搜尋引擎 | FR-02 | [ARCH.md](./architecture/ARCH.md#adr-008) |
| ADR-009 | Presidio PII 偵測 | FR-04 | [ARCH.md](./architecture/ARCH.md#adr-009) |
| ADR-010 | 階層式 Chunking 策略 | FR-17 | [ARCH.md](./architecture/ARCH.md#adr-010) |

---

## 5. 涵蓋度摘要

### 5.1 功能需求分布

| 類別 | FR 數量 | US 數量 | 狀態 |
|------|---------|---------|------|
| 已實作（✅） | 12 | 29 | Unit + Integration 覆蓋 |
| 規劃中（🔄） | 5 | 12 | 部分實作、持續強化 |
| 未來願景（🔮） | 6 | 6（未展開） | Phase 3 以後 |
| **合計** | **23** | **47** | — |

### 5.2 測試覆蓋統計

| 層級 | 檔案數 | 覆蓋範圍 |
|------|--------|---------|
| NestJS Unit (.spec.ts) | 42 | 全部已實作模組（auth, users, chat, prompts, audit, cleaning, health, websearch, llm, datasources） |
| Python rag-service (test_*.py) | 38 | RAG 核心管線、API 路由、整合測試、Maker-Checker 驗證 |
| Python data-pipeline (test_*.py) | 14 | 解析器、偵測器、去識別化策略 |
| React 前端 (*.test.tsx) | 13 | Admin 4 + Cleaner 5 + Chatbot 4（hooks + pages + components） |
| **合計** | **107** | 含前端測試框架（Vitest + RTL） |

### 5.3 追溯完整性

| 檢核項目 | 狀態 | 說明 |
|----------|------|------|
| 每個已實作 US 有對應 API 端點 | ✅ | 39/39 US 有明確端點（含 US-05-03、US-18-10/11/12） |
| 每個 API 端點有 NestJS 測試 | ✅ | 43 spec 檔案覆蓋全部 Controller/Service |
| 每個 FastAPI 路由有 Python 測試 | ✅ | 38 test 檔案覆蓋 API + 整合 |
| SRS §11 驗收案例有對應測試 | ✅ | 15/15 TC 全數有測試覆蓋（含 TC-02-004 一般訊息冪等、TC-05-006/007 Maker-Checker） |
| 每個 ADR 有對應 FR 引用 | ✅ | 10/10 ADR 標注影響範圍 |

---

## 6. Quality Gate 檢核

### 6.1 Phase 1（當前）

- [x] 所有 FR-01~09 + FR-16~21 有 Unit Test 覆蓋
- [x] 雙語言服務（NestJS + Python）皆有獨立測試
- [x] 驗收案例 TC-01~06 全數有對應測試
- [x] 安全性檢核項全數通過（SRS-T §11.3）
- [x] SEC-IDOR-01：Chat conversation ownership verification（`chat.service.ts:231-234`，防止 IDOR 跨用戶存取對話）
- [x] SEC-RATE-01：ThrottlerGuard APP_GUARD 全域 60req/min + auth 端點 per-endpoint 5req/min（`app.module.ts:52` + `auth.controller.ts`）
- [x] SEC-AUDIT-01：Audit interceptor 增強 — AuditEvent decorator + before/after 變更追蹤 + required 阻斷 + auditRead 敏感讀取（`audit-log.interceptor.ts` + `audit-actions.ts` + `audit-details.ts`）
- [ ] 行覆蓋率量測（NestJS ≥80%、Python ≥80%）— 規劃中

### 6.2 Phase 2（目標）

- [ ] 效能測試基準建立（k6 壓力測試）
- [x] 前端元件測試導入（Vitest + RTL）— 已完成 134 tests
- [x] E2E 測試基礎建設（supertest + rate-limit e2e）— 26/31 通過
- [ ] E2E 自動化擴充（Playwright）
- [ ] CI/CD 整合測試閘門

### 6.3 Phase 3（願景）

- [ ] FR-10~15 User Story 展開與追溯
- [ ] 多租戶架構安全測試
- [ ] 合規性自動化稽核

---

## 參考文件

| 文件 | 說明 | 路徑 |
|------|------|------|
| PRD.md | User Story 定義 | [specs/PRD.md](./PRD.md) |
| SRS_TECHNICAL.md | FR 定義與驗收標準 | [specs/SRS_TECHNICAL.md](./SRS_TECHNICAL.md) |
| ARCH.md | 架構決策紀錄 | [architecture/ARCH.md](./architecture/ARCH.md) |
| test-strategy.md | 測試策略 | [testing/test-strategy.md](../02-testing/test-strategy.md) |
| API 文件索引 | 全部端點規格 | [api/README.md](./api/README.md) |
| RFC-001 | 客戶角色與回應層級簡化 | [RFC-001.md](./RFC-001.md) |
| RFC-003 | 任務命名、快速改名與台灣時間 | [RFC-003.md](./RFC-003.md) |

---

> **文件結束**
>
> 本文件為 ODA Cyber Konsult 的需求追溯矩陣，串連 User Story → 功能需求 → API → 模組 → 測試 的完整鏈路。
> 新增功能時，請同步更新對應的追溯列。
