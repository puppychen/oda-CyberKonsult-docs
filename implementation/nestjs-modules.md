# NestJS API 模組建置總結

**日期:** 2026-02-09
**專案:** ODA Cyber Konsult - NestJS Backend API

---

## 已建立的模組

本次建置完成三個核心模組：**Audit**、**Users**、**WebSocket**，遵循 NestJS 分層架構標準。

---

## 1. Audit Module (稽核日誌模組)

### 目錄結構

```
apps/api/src/modules/audit/
├── audit.module.ts
├── controllers/
│   └── audit.controller.ts
├── services/
│   └── audit.service.ts
└── repositories/
    └── audit.repository.ts
```

### 功能說明

- **權限要求:** 僅限 Admin 角色存取
- **主要功能:**
  - 取得稽核日誌清單（支援分頁與多維度篩選）
  - 取得單一稽核日誌詳情
  - 自動關聯使用者資訊

### API 端點

| 方法 | 路徑 | 說明 |
|------|------|------|
| GET | `/api/audit` | 取得稽核日誌清單 |
| GET | `/api/audit/:id` | 取得單一日誌詳情 |

### 篩選參數

- `limit` / `offset` - 分頁控制
- `action` - 操作類型過濾（部分比對）
- `userId` - 使用者 ID 過濾
- `startDate` / `endDate` - 時間範圍過濾

---

## 2. Users Module (使用者管理模組)

### 目錄結構

```
apps/api/src/modules/users/
├── users.module.ts
├── controllers/
│   └── users.controller.ts
├── services/
│   └── users.service.ts
└── repositories/
    └── users.repository.ts
```

### 功能說明

- **權限區分:**
  - Admin 可存取所有端點
  - 一般使用者僅可存取 `/me` 端點
- **主要功能:**
  - 使用者清單管理（分頁）
  - 使用者角色更新
  - 使用者停用
  - 當前登入使用者資訊查詢

### API 端點

| 方法 | 路徑 | 權限 | 說明 |
|------|------|------|------|
| GET | `/api/users` | Admin | 取得使用者清單 |
| GET | `/api/users/me` | Any | 取得當前使用者資訊 |
| GET | `/api/users/:id` | Admin | 取得指定使用者 |
| PUT | `/api/users/:id/role` | Admin | 更新使用者角色 |

### 資料回傳欄位

- `id`, `email`, `name`, `role`, `isActive`
- `createdAt`, `updatedAt`（列表與詳情端點）

---

## 3. WebSocket Module (即時通訊模組)

### 目錄結構

```
apps/api/src/modules/websocket/
├── websocket.module.ts
└── gateways/
    └── task.gateway.ts
```

### 功能說明

- **Namespace:** `/cleaning/ws`
- **主要功能:**
  - 轉發 FastAPI 清洗任務進度更新
  - 支援單一任務或全域任務訂閱
  - 自動處理連線管理與清理
- **CORS 支援:**
  - `http://localhost:5173` (Admin Dashboard)
  - `http://localhost:5174` (Chatbot UI)

### WebSocket 連線模式

1. **連線時指定任務 ID**
   ```javascript
   const socket = io('http://localhost:4000/cleaning/ws', {
     query: { taskId: 'task-uuid' }
   });
   ```

2. **連線後訂閱所有任務**
   ```javascript
   const socket = io('http://localhost:4000/cleaning/ws');
   ```

3. **動態訂閱特定任務**
   ```javascript
   socket.emit('subscribe_task', 'task-uuid');
   ```

### 訊息格式

**事件名稱:** `task_update`

**訊息結構:**
```json
{
  "type": "progress" | "status" | "complete" | "error",
  "taskId": "uuid",
  "status": "processing" | "completed" | "failed",
  "progress": 75,
  "message": "處理中...",
  "timestamp": "2026-02-09T10:30:00Z"
}
```

---

## 架構設計原則

### 分層架構（符合 NestJS 最佳實踐）

```
Controller → Service → Repository → PrismaService
```

#### 各層職責

| 層級 | 職責 | 禁止事項 |
|------|------|----------|
| **Controller** | HTTP 路由、參數驗證、回應格式 | 不得包含商業邏輯、不得直接存取資料庫 |
| **Service** | 商業邏輯、資料驗證、錯誤處理 | 不得直接使用 HTTP 相關物件 |
| **Repository** | 資料存取、查詢封裝 | 不得包含商業邏輯 |

### 為何採用 Repository 模式？

依據 NestJS Framework Skill 的評估清單，本專案符合以下進階模式觸發條件（5 項指標中命中 3+ 項）：

1. ✅ 查詢含 3 層以上巢狀 include/select（如 AuditLog 關聯 User）
2. ✅ 專案要求 Service 層單元測試（可注入 Mock Repository）
3. ✅ 團隊多人共同開發（依專案背景判斷）

因此採用 **Controller → Service → Repository** 進階架構。

---

## 統一回應格式

所有 API 端點統一使用以下格式：

### 成功回應

```typescript
{
  success: true,
  data: T,
  timestamp: string
}
```

### 錯誤回應

```typescript
{
  success: false,
  error: string,
  timestamp: string
}
```

---

## AppModule 整合

已更新 `/Users/puppychen/Job/EcMap/zProjectsSource/oda-cyber-konsult/apps/api/src/app.module.ts`：

```typescript
@Module({
  imports: [
    PrismaModule,
    AuthModule,
    CleaningModule,
    ChatModule,
    AuditModule,      // ← 新增
    UsersModule,      // ← 新增
    WebSocketModule,  // ← 新增
  ],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

---

## 套件依賴需求

WebSocket 功能需要安裝以下套件（尚未執行）：

```bash
cd /Users/puppychen/Job/EcMap/zProjectsSource/oda-cyber-konsult/apps/api
pnpm add @nestjs/websockets @nestjs/platform-socket.io socket.io ws
pnpm add -D @types/ws
```

**執行時機:** 請在啟動 API 伺服器前完成安裝。

---

## 檔案清單

### Audit Module (4 個檔案)

1. `/Users/puppychen/Job/EcMap/zProjectsSource/oda-cyber-konsult/apps/api/src/modules/audit/audit.module.ts`
2. `/Users/puppychen/Job/EcMap/zProjectsSource/oda-cyber-konsult/apps/api/src/modules/audit/controllers/audit.controller.ts`
3. `/Users/puppychen/Job/EcMap/zProjectsSource/oda-cyber-konsult/apps/api/src/modules/audit/services/audit.service.ts`
4. `/Users/puppychen/Job/EcMap/zProjectsSource/oda-cyber-konsult/apps/api/src/modules/audit/repositories/audit.repository.ts`

### Users Module (4 個檔案)

5. `/Users/puppychen/Job/EcMap/zProjectsSource/oda-cyber-konsult/apps/api/src/modules/users/users.module.ts`
6. `/Users/puppychen/Job/EcMap/zProjectsSource/oda-cyber-konsult/apps/api/src/modules/users/controllers/users.controller.ts`
7. `/Users/puppychen/Job/EcMap/zProjectsSource/oda-cyber-konsult/apps/api/src/modules/users/services/users.service.ts`
8. `/Users/puppychen/Job/EcMap/zProjectsSource/oda-cyber-konsult/apps/api/src/modules/users/repositories/users.repository.ts`

### WebSocket Module (2 個檔案)

9. `/Users/puppychen/Job/EcMap/zProjectsSource/oda-cyber-konsult/apps/api/src/modules/websocket/websocket.module.ts`
10. `/Users/puppychen/Job/EcMap/zProjectsSource/oda-cyber-konsult/apps/api/src/modules/websocket/gateways/task.gateway.ts`

### 文件

11. `/Users/puppychen/Job/EcMap/zProjectsSource/oda-cyber-konsult/docs/api/audit-users-api.md`
12. `/Users/puppychen/Job/EcMap/zProjectsSource/oda-cyber-konsult/docs/NESTJS_MODULES_SUMMARY.md` (本文件)

### 更新檔案

13. `/Users/puppychen/Job/EcMap/zProjectsSource/oda-cyber-konsult/apps/api/src/app.module.ts` (已更新)

---

## 後續步驟

### 1. 安裝依賴套件

```bash
cd apps/api
pnpm add @nestjs/websockets @nestjs/platform-socket.io socket.io ws
pnpm add -D @types/ws
```

### 2. 確認資料庫 Schema

確認 Prisma Schema 包含以下 Model：

- `User` (含 `id`, `email`, `name`, `role`, `isActive`, `createdAt`, `updatedAt`)
- `AuditLog` (含 `id`, `action`, `userId`, `details`, `timestamp`，並關聯 `User`)

### 3. 執行 TypeScript 編譯檢查

```bash
cd apps/api
pnpm tsc --noEmit
```

### 4. 啟動開發伺服器

```bash
pnpm dev
```

### 5. 測試 API 端點

使用 Postman 或 curl 測試：

```bash
# 取得當前使用者資訊（需 JWT Token）
curl -H "Authorization: Bearer <token>" http://localhost:4000/api/users/me

# 取得稽核日誌（需 Admin 權限）
curl -H "Authorization: Bearer <admin-token>" http://localhost:4000/api/audit?limit=10

# WebSocket 連線測試（使用前端框架）
```

---

## 程式碼品質檢查

### TypeScript 嚴格模式

- ✅ 所有函式具備明確回傳型別
- ✅ 使用 `readonly` 修飾器保護依賴注入
- ✅ 查詢參數型別安全（使用 `parseInt` 轉換）

### NestJS 最佳實踐

- ✅ 遵循模組化架構
- ✅ 使用 Dependency Injection
- ✅ Guard 與 Decorator 實作權限控制
- ✅ Repository 層封裝資料存取

### 程式碼精簡度

- ✅ Controller 僅負責路由與回應格式
- ✅ Service 集中商業邏輯
- ✅ Repository 避免重複查詢邏輯
- ✅ 使用 `Promise.all` 最佳化並行查詢

---

## 參考文件

- [NestJS 官方文件](https://docs.nestjs.com/)
- [Prisma Client API](https://www.prisma.io/docs/reference/api-reference/prisma-client-reference)
- [Socket.IO 文件](https://socket.io/docs/v4/)
- [專案 API 文件](./api/audit-users-api.md)

---

**建置完成時間:** 2026-02-09
**維護者:** ODA Cyber Konsult Development Team
