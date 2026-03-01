# E2E 測試說明

## 概述

本目錄包含 NestJS API 的端對端（E2E）測試，涵蓋系統核心業務流程。

## 測試範圍

### 1. Health Check 測試
- `GET /health` - 整體健康狀態
- `GET /health/live` - 存活性檢查
- `GET /health/ready` - 就緒性檢查

### 2. 認證流程測試
- `POST /api/auth/register` - 使用者註冊（一般使用者與管理員）
- `POST /api/auth/login` - 使用者登入
- `POST /api/auth/refresh` - Token 刷新
- `POST /api/auth/logout` - 使用者登出

### 3. 聊天流程測試（需 JWT）
- `GET /api/chat/modes` - 取得可用模式
- `POST /api/chat` - 發送訊息（mock RAG service）
- `GET /api/chat/conversations` - 取得對話列表
- `GET /api/chat/conversations/:id` - 取得對話詳情
- `DELETE /api/chat/conversations/:id` - 刪除對話

### 4. Admin 流程測試（需 admin JWT）

#### 使用者管理
- `GET /api/users` - 使用者列表
- `PUT /api/users/:id/role` - 變更角色
- `PATCH /api/users/:id/status` - 停用/啟用使用者

#### 稽核日誌
- `GET /api/audit-logs` - 稽核日誌查詢

#### 提示詞管理
- `GET /api/prompts` - 提示詞列表
- `POST /api/prompts` - 建立提示詞
- `GET /api/prompts/:id` - 提示詞詳情
- `PUT /api/prompts/:id` - 更新提示詞
- `DELETE /api/prompts/:id` - 刪除提示詞

## 執行測試

### 前置需求

1. **安裝 supertest 依賴**（首次執行前）

```bash
# 在專案根目錄執行
pnpm add -D --filter @oda-cyber/api supertest @types/supertest
```

2. **確保環境變數正確配置**

E2E 測試會使用 mock PrismaService，因此不需要真實資料庫連線。但如果需要測試與外部服務的整合，請確保環境變數正確設定。

### 執行指令

```bash
# 在 apps/api 目錄下執行
pnpm test:e2e

# 或從專案根目錄執行
pnpm --filter @oda-cyber/api test:e2e
```

### 測試輸出

測試成功時會顯示類似以下輸出：

```
PASS  test/app.e2e-spec.ts
  App E2E Tests
    Health Check
      ✓ GET /health - 應回傳健康狀態 (50ms)
      ✓ GET /health/live - 應回傳 ok 狀態 (10ms)
      ✓ GET /health/ready - 應回傳就緒狀態 (15ms)
    Authentication Flow
      ✓ POST /api/auth/register - 應成功註冊一般使用者 (100ms)
      ✓ POST /api/auth/register - 應成功註冊管理員 (80ms)
      ...
```

## Mock 策略

### PrismaService Mock

E2E 測試使用完整的 mock PrismaService，避免真實資料庫連線。Mock 實現包括：

- **In-memory 資料儲存**：使用 Map 結構儲存測試資料
- **基本 CRUD 操作**：findUnique、findMany、create、update、delete
- **Transaction 支援**：簡化版 `$transaction` 實現
- **關聯查詢**：支援基本的 include 查詢

### RAG Service Mock

聊天流程測試中，RAG service 的呼叫會被 mock，回傳固定的測試資料。

## 測試架構

```
test/
├── jest-e2e.json          # E2E Jest 設定
├── app.e2e-spec.ts        # 主要 E2E 測試檔案
└── README.md              # 本說明文件
```

## 測試設計原則

1. **測試隔離**：每個測試案例獨立，不依賴其他測試的執行結果
2. **順序執行**：同一 describe block 內的測試按順序執行，模擬真實使用流程
3. **資料清理**：測試結束後自動清理 mock 資料
4. **錯誤場景**：涵蓋正常流程與錯誤處理（401、403、409 等）
5. **權限檢查**：驗證 JWT 驗證與角色權限控制

## 擴展測試

如需新增更多 E2E 測試場景：

1. **新增測試檔案**：在 `test/` 目錄下建立新的 `.e2e-spec.ts` 檔案
2. **更新 Mock**：如需測試新的 Prisma Model，請擴展 `createMockPrismaService()` 函數
3. **保持一致**：遵循現有測試的命名與結構規範

## 注意事項

- E2E 測試執行時間較長，建議在提交前執行
- 如需測試真實資料庫整合，請建立獨立的測試資料庫環境
- Mock 策略適用於快速驗證業務邏輯，不取代整合測試
- 測試中的 token 有效期限為測試期間，不會過期

## 疑難排解

### 測試執行卡住

如果測試執行後無法結束，檢查：
- `--forceExit` flag 是否已加入 test:e2e script
- WebSocket 或 HTTP 連線是否正確關閉

### 測試失敗

檢查：
- supertest 依賴是否已安裝
- ValidationPipe 設定是否與 main.ts 一致
- Mock PrismaService 是否正確實現所需的資料庫操作

### 權限錯誤

確認：
- JWT token 正確產生並傳遞
- RolesGuard 正確驗證使用者角色
- 測試中的角色設定（user/admin）符合預期

## 參考資源

- [NestJS Testing Documentation](https://docs.nestjs.com/fundamentals/testing)
- [Supertest GitHub](https://github.com/visionmedia/supertest)
- [Jest Documentation](https://jestjs.io/docs/getting-started)
