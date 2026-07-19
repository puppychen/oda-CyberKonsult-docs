---
audience: human-primary
purpose: reference
status: review
owner: Jake
---

# 原型索引

## TL;DR

本目錄收錄正式開發前的靜態原型，先確認畫面、名稱與互動，再進入實作。

## 給 Jake 看（決策 / 簽收）

| 檔名 | audience | purpose | status | 摘要 |
|------|----------|---------|--------|------|
| [`source-file-pending-review-delete-mockup.html`](./source-file-pending-review-delete-mockup.html) | human-primary | decision | approved | 沿用現有 Cleaner 風格，呈現管理員邏輯刪除、已刪除篩選檢視、台灣時間、影響確認與操作結果 |
| [`chat-response-regeneration-mockup.html`](./chat-response-regeneration-mockup.html) | human-primary | decision | review | 沿用現有 Chatbot 風格，保留舊回覆並在對話末端產生新回覆的完整狀態 |
| [`web-reference-sources-mockup.html`](./web-reference-sources-mockup.html) | human-primary | decision | review | 沿用現有 Chatbot 風格的網路／知識庫參考來源收合、展開與無來源狀態 |
| [`auth-session-recovery-mockup.html`](./auth-session-recovery-mockup.html) | human-primary | decision | review | 連線暫時異常與登入確定失效狀態；保留目前對話及未送出內容 |
| [`task-name-taipei-time-mockup.html`](./task-name-taipei-time-mockup.html) | human-primary | decision | review | 上傳任務名稱必填、任務中心快速改名、欄位順序與台灣時間顯示 |
| [`regulation-knowledge-type-required-mockup.html`](./regulation-knowledge-type-required-mockup.html) | human-primary | decision | review | 法規／知識類型必填化、整批單一類型與 SOP 對齊 |
| [`customer-role-chatbot-current-style-mockup.html`](./customer-role-chatbot-current-style-mockup.html) | human-primary | decision | review | 沿用現有 Chatbot 風格的角色與額度調整 |
| [`customer-role-admin-current-style-mockup.html`](./customer-role-admin-current-style-mockup.html) | human-primary | decision | review | 沿用現有 Ant Design 風格的角色、提示詞與設定調整 |
| [`customer-role-chatbot-mockup.html`](./customer-role-chatbot-mockup.html) | human-primary | decision | superseded | 初版功能構圖，因風格不符現況而停用 |
| [`customer-role-admin-mockup.html`](./customer-role-admin-mockup.html) | human-primary | decision | superseded | 初版功能構圖，因風格不符現況而停用 |

## 給 AI 引用（細節記錄）

（無）

## 雙用

（無）

## 放置規則

本目錄只放正式開發前的設計驗證原型；正式 React 程式仍位於 `apps/*`。
