# Chatbot 前端實作完成

## 已建立的檔案清單

### 1. 配置修改
- `/Users/puppychen/Job/EcMap/zProjectsSource/oda-cyber-konsult/apps/chatbot/vite.config.ts`
  - 已將 proxy 簡化為單一 `/api` 路由指向 NestJS (port 4000)

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
  - 顯示來源檔案、預覽內容、相關度分數

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
- NestJS API (port 4000)
- PostgreSQL (port 5432)
- Qdrant (port 6333)

## 啟動方式

```bash
cd /Users/puppychen/Job/EcMap/zProjectsSource/oda-cyber-konsult
pnpm dev --filter @oda-cyber/chatbot
```

瀏覽器開啟：http://localhost:5174

## 後續可擴充功能

1. 歷史對話列表 (側邊欄)
2. 對話標題自動生成
3. 匯出對話記錄
4. 引用來源點擊開啟完整內容
5. 複製訊息內容
6. 深色模式切換
7. 使用者設定頁面
8. 使用配額顯示
