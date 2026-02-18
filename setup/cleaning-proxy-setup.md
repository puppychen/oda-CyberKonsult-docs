# Cleaning Proxy 模組安裝與啟動指南

## 已建立的檔案

### 1. 核心服務
- `/apps/api/src/modules/cleaning/services/cleaning-proxy.service.ts`
  - 提供所有 HTTP 代理方法（GET/POST/PUT/DELETE/Upload/Download）

### 2. Controllers (5 個)
- `/apps/api/src/modules/cleaning/controllers/upload.controller.ts`
- `/apps/api/src/modules/cleaning/controllers/clean.controller.ts`
- `/apps/api/src/modules/cleaning/controllers/tasks.controller.ts`
- `/apps/api/src/modules/cleaning/controllers/rules.controller.ts`
- `/apps/api/src/modules/cleaning/controllers/download.controller.ts`

### 3. 模組定義
- `/apps/api/src/modules/cleaning/cleaning.module.ts`

### 4. 文件
- `/docs/api/cleaning-proxy.md`

## 已修改的檔案

### 1. AppModule
- `/apps/api/src/app.module.ts`
  - 已加入 `CleaningModule` 至 imports

### 2. Package.json
- `/apps/api/package.json`
  - 已加入 `@nestjs/axios: ^3.1.2`
  - 已加入 `axios: ^1.7.9`

### 3. 環境變數範例
- `/.env.example`
  - 已加入 `FASTAPI_BASE_URL=http://localhost:8000`

## 安裝步驟

### 1. 安裝依賴

```bash
# 在專案根目錄執行
pnpm install
```

### 2. 設定環境變數

複製 `.env.example` 至 `.env`（如尚未建立）：

```bash
cp .env.example .env
```

確認 `.env` 中包含：

```env
FASTAPI_BASE_URL=http://localhost:8000
```

### 3. 啟動服務

確保以下服務已啟動：

```bash
# 1. PostgreSQL (Docker)
docker ps | grep postgres

# 2. FastAPI (data-pipeline)
cd python/data-pipeline
uv run uvicorn data_pipeline.api.main:app --host 0.0.0.0 --port 8000 --reload

# 3. NestJS API
cd apps/api
pnpm dev
```

## 驗證安裝

### 1. 檢查模組載入

NestJS 啟動時應顯示：

```
[Nest] INFO [RoutesResolver] UploadController {/api/v1/upload}:
[Nest] INFO [RouterExplorer] Mapped {/api/v1/upload, POST} route
[Nest] INFO [RouterExplorer] Mapped {/api/v1/upload/:fileId, GET} route
[Nest] INFO [RoutesResolver] CleanController {/api/v1/clean}:
...
```

### 2. 測試端點 (需先登入取得 JWT Token)

```bash
# 取得 Token (假設已有 admin 使用者)
TOKEN=$(curl -X POST http://localhost:4000/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@example.com","password":"password"}' \
  | jq -r '.access_token')

# 測試取得任務列表
curl http://localhost:4000/api/v1/tasks \
  -H "Authorization: Bearer $TOKEN"
```

### 3. 檢查代理轉發

NestJS 應正確轉發請求至 FastAPI，並返回相同格式的回應。

## 架構總覽

```
Admin Dashboard (5173)
    ↓
NestJS API (4000)
    ├── JWT 驗證
    ├── Admin 角色檢查
    └── CleaningProxyService
            ↓
        FastAPI (8000)
            ├── De-identification Engine
            ├── File Processing
            └── Database (PostgreSQL)
```

## 權限說明

所有 Cleaning API 端點皆需滿足：

1. 有效的 JWT Token
2. 使用者角色為 `admin`

若不符合，將返回 401 Unauthorized 或 403 Forbidden。

## 錯誤排查

### 1. 模組找不到錯誤

```
Error: Cannot find module '@nestjs/axios'
```

**解決方式**：執行 `pnpm install`

### 2. FastAPI 連線失敗

```
Error: connect ECONNREFUSED 127.0.0.1:8000
```

**解決方式**：確認 FastAPI 已啟動於 port 8000

```bash
cd python/data-pipeline
uv run uvicorn data_pipeline.api.main:app --host 0.0.0.0 --port 8000 --reload
```

### 3. 權限檢查失敗

```
403 Forbidden
```

**解決方式**：確認使用者角色為 `admin`

```sql
-- 檢查使用者角色
SELECT id, email, role FROM users WHERE email = 'admin@example.com';

-- 更新為 admin
UPDATE users SET role = 'admin' WHERE email = 'admin@example.com';
```

## 後續開發

此代理層已完成基礎功能，後續可擴展：

1. 加入 Swagger API 文件 (`@ApiTags`, `@ApiOperation`)
2. 實作錯誤處理中介層（統一格式化 FastAPI 錯誤）
3. 加入請求/回應日誌
4. 實作快取機制（規則設定檔等靜態資料）
5. 加入 Rate Limiting 防止 API 濫用

## 相關文件

- [Cleaning API 完整文件](/docs/api/cleaning-api.md)
- [Cleaning Proxy 架構說明](/docs/api/cleaning-proxy.md)
