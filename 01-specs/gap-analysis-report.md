# 系統功能完備性差距分析報告

> **ODA Cyber Konsult - 資安助手 RAG 系統**
>
> 稽核日期：2026-03-28
> 稽核範圍：PRD v1.4.0 (17 Epic / 137+ AC) + RBAC 權限矩陣 + 安全基線
> 稽核方法：靜態程式碼分析（Phase A）

---

## 執行摘要

| 指標 | 數量 |
|------|------|
| 稽核 AC 總數 | 112 |
| ✅ 通過 | 93（83%）|
| ⚠️ 部分符合 | 17（15%）|
| ❌ 缺失 | 2（2%）|
| 安全差距 | 4 項（含 1 項 P0）|

**整體評估**：系統核心功能完備度達 83%，RAG 問答、三層模式、Maker-Checker 審核等關鍵功能全數到位。主要差距集中在稽核日誌覆蓋不完整、前端 UI 部分缺漏、以及一項嚴重的安全問題（註冊端點可指定任意角色）。

---

## P0：安全漏洞（必須立即修正）

### GAP-SEC-01：註冊端點允許指定任意角色（含 admin）

| 項目 | 說明 |
|------|------|
| **嚴重度** | **P0 Critical** |
| **位置** | `register.dto.ts:17` + `auth.service.ts:135` |
| **問題** | RegisterDto 接受 `role` 參數，允許 `admin`/`data_cleaner`/`data_reviewer` 等特權角色。`auth.service.ts:135` 直接使用 `dto.role || 'user'`。任何人可自行註冊為 admin。|
| **修正** | 移除 RegisterDto 的 role 欄位，或限制為 `basic_user`/`user` 二選一，特權角色僅限 admin 指派 |

---

## P1：合規風險差距（建議優先修正）

### GAP-01：Chatbot UI 缺少角色攔截

| 項目 | 說明 |
|------|------|
| AC | RBAC 矩陣（SRS_BUSINESS §3.3）|
| 位置 | `chatbot/src/App.tsx` + `chat.controller.ts:18` |
| 問題 | data_cleaner 和 data_reviewer 可登入 Chatbot 使用聊天功能，與權限矩陣「✗」不符 |
| 修正 | Chatbot useAuth 加入角色白名單過濾；後端 ChatController 加入 @Roles 排除 data_cleaner/data_reviewer |

### GAP-02：稽核日誌覆蓋不完整（AC-07-01-01 / AC-07-01-04）

| 項目 | 說明 |
|------|------|
| AC | AC-07-01-01（12 種操作類型）、AC-07-01-04（100% 完整性）|
| 位置 | `auth.controller.ts`、`users.controller.ts`、`chat.controller.ts`、`prompts.controller.ts` |
| 問題 | (1) 未定義明確的 12 種操作類型 enum (2) auth（登入/登出）、users（角色變更）、prompts（CRUD）、chat（回饋）等 Controller 未套用 AuditLogInterceptor |
| 修正 | 定義 AuditActionType enum；對 auth/users/prompts/chat Controller 加掛 Interceptor |

### GAP-03：回饋事件無稽核記錄（AC-22-01-04）

| 項目 | 說明 |
|------|------|
| AC | AC-22-01-04 |
| 位置 | `chat.controller.ts`（無 AuditLogInterceptor）|
| 問題 | submitFeedback 操作未寫入稽核日誌 |
| 修正 | ChatController 加掛 AuditLogInterceptor，或 submitFeedback 方法內手動記錄 |

### GAP-04：Admin Dashboard 無法建立新使用者（AC-01-02-01）

| 項目 | 說明 |
|------|------|
| AC | AC-01-02-01 |
| 位置 | `users.controller.ts`（無 @Post）+ `UsersPage.tsx`（無新增表單）|
| 問題 | 使用者僅能透過公開的 /api/auth/register 自行註冊，Admin 無法從後台建立使用者 |
| 修正 | Users Controller 新增 @Post 端點 + Admin UI 新增「新增使用者」表單 |

### GAP-05：帳號鎖定回傳 401 非 423（AC-01-01-03）

| 項目 | 說明 |
|------|------|
| AC | AC-01-01-03 |
| 位置 | `auth.service.ts:41` |
| 問題 | 帳號鎖定時拋出 UnauthorizedException (401)，AC 要求 423 (Locked) |
| 修正 | 改用 `throw new HttpException('帳號已鎖定', HttpStatus.LOCKED)` |

---

## P2：功能差距（體驗優化）

### GAP-06：Admin UI 角色 Select 不完整（AC-01-02-03）

| 項目 | 說明 |
|------|------|
| 位置 | `admin/src/pages/UsersPage.tsx:93-97` |
| 問題 | 前端 Select 僅顯示 3 種角色（user/consultant/admin），缺少 basic_user/it_user/data_cleaner/data_reviewer |
| 修正 | Select options 擴充為 7 種角色 |

### GAP-07：ZIP 防護參數過於寬鬆（AC-05-03-02）

| 項目 | 說明 |
|------|------|
| 位置 | `python/rag-service/src/rag_service/config.py:54-56` |
| 問題 | 壓縮比 100:1（AC 要求 20:1）、總大小 500MB（AC 要求 200MB）|
| 修正 | `max_zip_ratio=20`, `max_zip_total_size_mb=200` |

### GAP-08：WebSocket 進度缺少檔案名稱與實體數（AC-06-01-02）

| 項目 | 說明 |
|------|------|
| 位置 | `processor.py:112-115` |
| 問題 | WS progress 推送 `current_file` 為索引數字，非檔案名稱；未含偵測實體數 |
| 修正 | 推送增加 `filename` 和 `entities_found` 欄位 |

### GAP-09：送審缺少備註輸入 UI（AC-18-10-02）

| 項目 | 說明 |
|------|------|
| 位置 | `cleaner/src/pages/TaskReviewPage.tsx:86` |
| 問題 | SubmitForReviewDto 有 optional note 欄位，但前端送審時傳空物件，未提供備註輸入框 |
| 修正 | 送審確認對話框加入 TextArea 備註欄位 |

### GAP-10：PII 實體未在文字中高亮標示（AC-18-02-02）

| 項目 | 說明 |
|------|------|
| 位置 | `cleaner/src/pages/FileReviewPage.tsx:225-238` |
| 問題 | 僅顯示 PII 統計摘要，未在文字內容中高亮標示各實體位置 |
| 修正 | 利用後端回傳的 entities 陣列中的 start/end 位置資訊，在文字渲染時加入高亮 span |

### GAP-11：趨勢圖前端未實作（AC-19-03-01 / AC-19-03-02）

| 項目 | 說明 |
|------|------|
| 位置 | `cleaner/src/pages/DashboardPage.tsx` |
| 問題 | 後端 `/analytics/timeline` API 已就位（支援 days 參數 1~365），但前端未呼叫此 API，無趨勢折線圖 |
| 修正 | DashboardPage 加入 Timeline 區塊，呼叫 timeline API 並以折線圖呈現 |

### GAP-12：退回後狀態為 rejected 非 pending（AC-18-11-03）

| 項目 | 說明 |
|------|------|
| 位置 | `python/rag-service/.../review.py:406` |
| 問題 | AC 描述「退回後狀態回 pending」，實際為 `rejected`。功能等效（submit 允許從 rejected 重新送審）但狀態名稱不一致 |
| 修正 | 建議更新 AC 措辭為「退回後狀態變為 rejected，可重新編輯並送審」（實作合理，AC 需修正）|

---

## P3：非功能需求待驗證

| AC | 項目 | 說明 |
|------|------|------|
| AC-02-01-02 | P95 回應 < 10 秒 | 未建立效能測試基準（k6 壓測待建）|
| AC-04-01-02 | PII 偵測準確率 > 95% | 未見基準測試數據 |
| AC-04-04-02 | 處理速度 > 10 檔/分鐘 | 未見基準測試數據 |

---

## 安全基線檢查結果

| # | 項目 | 狀態 | 備註 |
|---|------|------|------|
| 1 | JWT + RBAC Guards | ⚠️ | Guards 非全域掛載，依賴手動配置 |
| 2 | 密碼政策（普級） | ✅ | 8碼+複雜度+90天+歷史2代 |
| 3 | Input Validation | ✅ | ValidationPipe 全域 whitelist+transform |
| 4 | Rate Limiting | ⚠️ | ThrottlerModule 已註冊但 Guard 可能未全域生效 |
| 5 | X-Internal-Token | ✅ | NestJS→FastAPI hmac 驗證 + IP 白名單 |
| 6 | Maker-Checker | ✅ | 4 處 submitted_by ≠ approved_by 驗證 |
| 7 | SAST | ✅ | eslint-plugin-security 7 條規則 |
| 8 | PasswordChangeRequired | ✅ | APP_GUARD 全域掛載 |
| 9 | 上傳限制 50MB | ✅ | 串流讀取即時檢查 |
| 10 | Swagger 文件 | ✅ | /api/docs BearerAuth |

---

## 修正優先排序

| 優先級 | 差距 | 工作量 | 影響 |
|--------|------|--------|------|
| **P0** | GAP-SEC-01 註冊角色漏洞 | S（1h） | 安全漏洞，可自行註冊 admin |
| **P1** | GAP-01 Chatbot 角色攔截 | S（1h） | 合規：權限矩陣不符 |
| **P1** | GAP-02 稽核日誌覆蓋 | M（3h） | 合規：稽核完整性不足 |
| **P1** | GAP-03 回饋稽核 | S（30m） | 合規：稽核遺漏 |
| **P1** | GAP-04 Admin 建立使用者 | M（2h） | 管理功能缺失 |
| **P1** | GAP-05 鎖定狀態碼 | S（15m） | API 規格不符 |
| **P2** | GAP-06 角色 Select | S（30m） | UI 不完整 |
| **P2** | GAP-07 ZIP 參數 | S（15m） | 安全參數 |
| **P2** | GAP-08 WS 進度 | S（1h） | UX 體驗 |
| **P2** | GAP-09 送審備註 | S（30m） | UX 體驗 |
| **P2** | GAP-10 PII 高亮 | M（3h） | UX 體驗 |
| **P2** | GAP-11 趨勢圖 | M（2h） | 功能缺漏 |
| **P2** | GAP-12 退回狀態 | S（文件修正） | AC 措辭調整 |

> S = Small（< 1h）、M = Medium（1-3h）、L = Large（> 3h）

---

## Epic 完備度總覽

| Epic | 名稱 | AC 總數 | ✅ | ⚠️ | ❌ | 完備率 |
|------|------|---------|---|---|---|--------|
| 1 | 認證授權 | 9 | 6 | 2 | 1 | 67% |
| 2 | RAG 問答 | 13 | 12 | 1 | 0 | 92% |
| 3 | 提示詞管理 | 8 | 7 | 1 | 0 | 88% |
| 4 | 資料去識別化 | 10 | 8 | 2 | 0 | 80% |
| 5 | 檔案上傳下載 | 8 | 7 | 1 | 0 | 88% |
| 6 | 即時通知 | 2 | 1 | 1 | 0 | 50% |
| 7 | 稽核日誌 | 4 | 2 | 2 | 0 | 50% |
| 8 | 固定規則 | 3 | 3 | 0 | 0 | 100% |
| 9 | 任務管理 | 4 | 3 | 1 | 0 | 75% |
| 10 | 三層模式 | 7 | 7 | 0 | 0 | 100% |
| 12 | Cleaner App | 30 | 27 | 3 | 0 | 90% |
| 13 | 儀表板 | 8 | 6 | 2 | 0 | 75% |
| 16 | 回饋 | 4 | 3 | 0 | 1 | 75% |
| 17 | 信心度 | 5 | 4 | 1 | 0 | 80% |
| **合計** | | **115** | **96** | **17** | **2** | **83%** |

---

## 下一步建議

1. **立即**：修正 GAP-SEC-01（註冊角色漏洞）— 不超過 1 小時
2. **本週**：修正 P1 差距（GAP-01~05）— 約 7 小時
3. **Phase B**：Chrome 動態驗證修正後的功能
4. **Phase 2-C**：補建 NFR 基準測試（k6 壓測、PII 準確率測試）

---

> 本報告基於靜態程式碼分析產出，Phase B 動態瀏覽器驗證將進一步確認實際行為。
