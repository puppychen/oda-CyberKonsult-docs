---
audience: both
---

# Chatbot 前端實作完成

## TL;DR

Chatbot 以 SSE 顯示回答，並在回答下方提供預設收合的「N 個引用來源」。最新完成回覆可用 icon 重新產出，原問答保留並在底部追加新問答；知識庫與網路搜尋來源沿用同一區塊，不改變既有聊天介面風格。

## 已建立的檔案清單

## 參考來源收合機制

- `MessageBubble` 只在助理訊息有來源時顯示 `SourceList`；沒有來源時不保留空白區塊。
- 收合按鈕顯示來源總數，預設不展開。知識庫來源顯示文件名稱、摘要與相關度；網路來源顯示標題、原始 URL、摘要與「網路搜尋」標籤。
- 網路 URL 以新分頁開啟，並設定 `noopener noreferrer`，避免新頁面控制原 Chatbot 分頁。
- SSE `done.sources` 會寫入目前訊息；切換歷史對話時從 `messages.sources` 還原，所以重新載入後仍能查看來源。
- 來源列表沿用既有色彩與間距，手機寬度下由訊息區塊約束，不產生水平捲動。

## 非資安議題提醒

- SSE `done.topicScope` 會保存到助理訊息；切換歷史對話時從 `metadata.topicScope` 還原。
- `non_cybersecurity` 顯示「此問題與資安議題無直接相關」提醒與服務範圍。
- `mixed` 顯示「此問題包含非資安內容」，說明只回答資安部分。
- `cybersecurity` 與 `unclear` 不顯示非資安提醒；串流完成前也不提前顯示。
- 提醒沿用現有琥珀色資訊列，不變更聊天頁整體視覺風格。

## 最新回覆重新產出

- `ChatWindow` 只配對最後一則使用者提示詞與其後最新完成的助理回覆；回答必須具有後端 UUID，且 `topicScope` 為 `cybersecurity` 或 `mixed`，非最新、分類未知或未持久化回覆不傳入重新產出 callback。
- `MessageBubble` 在既有讚／不讚右側顯示 `RefreshCw` icon；沒有可見文字，hover `title` 為「重新產出」，並保留鍵盤與螢幕閱讀器名稱。
- 點擊後 `useChat` 以相同問題、目前 mode、conversation ID、原助理訊息 ID 與新 UUID attempt 建立 SSE；舊問答不刪除，新來源與回饋獨立保存。
- `sendingRef` 同步鎖在 React 狀態更新前阻擋第二條串流；串流中、失敗、固定回覆、唯讀狀態不顯示 icon，額度為 0 時停用。
- 重新產出失敗沿用既有「重試」流程；只有最新失敗氣泡顯示重試，且重試保留 `regenerateFromMessageId` 與 `regenerationAttemptId`。串流進行中先拒絕重試，不先刪除 UI 訊息；後端有效 lease 的 `processing` claim 會回 409，5 分鐘逾時或 `failed` claim 可換發 lease 續跑，已落盤回答則直接回放。逾時接手後，舊連線的寫入與失敗回報都不能影響新 lease；額度保留與 claim 建立同交易提交或回滾。前端只接受 UUID 格式的 `serverId`／持久化 message ID，格式錯誤時不顯示 icon。

### 1. 配置修改
- `/Users/puppychen/Job/EcMap/zProjectsSource/oda-cyber-konsult/apps/chatbot/vite.config.ts`
  - 已將 proxy 簡化為單一 `/api` 路由指向 NestJS (port 3051)

### 2. API 層
- `/Users/puppychen/Job/EcMap/zProjectsSource/oda-cyber-konsult/apps/chatbot/src/api/client.ts`
  - Token 管理 (localStorage)
  - 統一的 request 函數
  - 完整的 API 方法 (login, register, getMe, sendMessage, conversations)

### 3. Hooks
- `/Users/puppychen/Job/EcMap/zProjectsSource/oda-cyber-konsult/apps/chatbot/src/hooks/useAuth.ts`
  - 使用者認證狀態管理
  - login, register, logout 方法
  - 自動檢查 token 有效性

- `/Users/puppychen/Job/EcMap/zProjectsSource/oda-cyber-konsult/apps/chatbot/src/hooks/useChat.ts`
  - 聊天訊息管理
  - SSE 串流支援
  - conversationId 追蹤
  - 停止串流功能

### 4. 元件
- `/Users/puppychen/Job/EcMap/zProjectsSource/oda-cyber-konsult/apps/chatbot/src/components/MessageBubble.tsx`
  - 訊息氣泡顯示
  - 支援使用者/助手訊息樣式差異
  - 整合 Markdown 與引用來源

- `/Users/puppychen/Job/EcMap/zProjectsSource/oda-cyber-konsult/apps/chatbot/src/components/MarkdownRenderer.tsx`
  - 簡易 Markdown 解析
  - 支援標題、列表、程式碼區塊、粗體、行內程式碼

- `/Users/puppychen/Job/EcMap/zProjectsSource/oda-cyber-konsult/apps/chatbot/src/components/SourceList.tsx`
  - 引用來源展開/收合
  - 區分知識庫與網路搜尋來源，顯示檔名／網頁標題、預覽內容、相關度或原始連結

- `/Users/puppychen/Job/EcMap/zProjectsSource/oda-cyber-konsult/apps/chatbot/src/components/ChatInput.tsx`
  - 自動高度調整的輸入框
  - 支援 Enter 送出、Shift+Enter 換行
  - 送出/停止按鈕切換

- `/Users/puppychen/Job/EcMap/zProjectsSource/oda-cyber-konsult/apps/chatbot/src/components/RoleSelector.tsx`
  - 三種模式切換 (新手/一般/顧問)
  - Tooltip 說明

- `/Users/puppychen/Job/EcMap/zProjectsSource/oda-cyber-konsult/apps/chatbot/src/components/ChatWindow.tsx`
  - 訊息列表顯示
  - 自動捲動至底部
  - 空狀態歡迎畫面

### 5. 頁面
- `/Users/puppychen/Job/EcMap/zProjectsSource/oda-cyber-konsult/apps/chatbot/src/pages/LoginPage.tsx`
  - 登入/註冊表單切換
  - 錯誤訊息顯示
  - 載入狀態處理

- `/Users/puppychen/Job/EcMap/zProjectsSource/oda-cyber-konsult/apps/chatbot/src/pages/ChatPage.tsx`
  - 完整聊天介面
  - Header 整合 (使用者資訊、新對話、登出)
  - 模式選擇器

### 6. 根元件
- `/Users/puppychen/Job/EcMap/zProjectsSource/oda-cyber-konsult/apps/chatbot/src/App.tsx`
  - 認證狀態路由
  - 載入畫面
  - LoginPage / ChatPage 切換

## 技術特點

1. **TypeScript 嚴格模式**
   - 所有函數皆明確定義返回型別
   - 無 `any` 使用
   - 完整的 Props 型別定義

2. **React 最佳實務**
   - 使用 `useCallback` 記憶化回調
   - 使用 `useRef` 處理 DOM 引用與 AbortController
   - 函數元件命名匯出 (利於除錯)

3. **串流支援**
   - SSE (Server-Sent Events) 實作
   - 支援中斷串流
   - 逐字元顯示效果

4. **錯誤處理**
   - 401 自動清除 token 並重載
   - 網路錯誤友善提示
   - 表單驗證

5. **使用者體驗**
   - 自動捲動至最新訊息
   - 輸入框自動調整高度
   - 載入與串流狀態視覺回饋
   - 引用來源可展開/收合

## 待啟動服務

確保以下服務正在運行：
- NestJS API (port 3051)
- PostgreSQL (port 5432)
- Qdrant (port 6333)

## 啟動方式

```bash
cd /Users/puppychen/Job/EcMap/zProjectsSource/oda-cyber-konsult
pnpm dev --filter @oda-cyber/chatbot
```

瀏覽器開啟：http://localhost:5502

## 後續可擴充功能

1. 歷史對話列表 (側邊欄)
2. 對話標題自動生成
3. 匯出對話記錄
4. 知識庫來源站內全文檢視
5. 複製訊息內容
6. 深色模式切換
7. 使用者設定頁面
8. 使用配額顯示
