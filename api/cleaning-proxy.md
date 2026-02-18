# Cleaning Module - NestJS Proxy Layer

## 概述

Cleaning 模組是 NestJS API 的代理層，負責將前端的去識別化請求轉發至 FastAPI 後端 (port 8000)，並確保所有操作僅限 Admin 使用者。

## 模組架構

```
cleaning/
├── cleaning.module.ts                    # 模組定義與依賴注入
├── services/
│   └── cleaning-proxy.service.ts         # 核心代理服務
└── controllers/
    ├── upload.controller.ts              # 檔案上傳 API
    ├── clean.controller.ts               # 清洗任務 API
    ├── tasks.controller.ts               # 任務管理 API
    ├── rules.controller.ts               # 規則設定檔 API
    └── download.controller.ts            # 檔案下載 API
```

## 設計模式

### 1. Proxy Pattern (代理模式)

所有 Controller 不直接實作業務邏輯，而是透過 `CleaningProxyService` 轉發請求至 FastAPI。

**優點**：
- 單一職責：NestJS 負責驗證與授權，FastAPI 負責業務邏輯
- 易於維護：FastAPI 更新時無需修改 NestJS
- 統一入口：所有請求經過相同的驗證流程

### 2. Dependency Injection (依賴注入)

透過 NestJS 的 DI 系統注入 `CleaningProxyService`：

```typescript
@Controller('api/v1/upload')
export class UploadController {
  constructor(private readonly proxy: CleaningProxyService) {}
}
```

### 3. Guard Pattern (守衛模式)

使用 `JwtAuthGuard` 和 `RolesGuard` 確保安全性：

```typescript
@UseGuards(JwtAuthGuard, RolesGuard)
@Roles('admin')
export class UploadController {
  // 所有方法皆受保護
}
```

## 核心服務

### CleaningProxyService

提供 6 種代理方法：

| 方法 | 簽章 | 用途 |
|------|------|------|
| `proxyGet` | `(path: string, query?: Record<string, string>): Promise<unknown>` | GET 請求 |
| `proxyPost` | `(path: string, body?: unknown): Promise<unknown>` | POST 請求 |
| `proxyPut` | `(path: string, body?: unknown): Promise<unknown>` | PUT 請求 |
| `proxyDelete` | `(path: string): Promise<unknown>` | DELETE 請求 |
| `proxyUpload` | `(path: string, formData: Buffer, headers: Record<string, string>): Promise<unknown>` | 檔案上傳 |
| `proxyDownload` | `(path: string, res: Response): Promise<void>` | 檔案下載 |

### 關鍵實作細節

#### 1. 檔案上傳 (Raw Body Handling)

```typescript
private getRawBody(req: Request): Promise<Buffer> {
  return new Promise((resolve, reject) => {
    const chunks: Buffer[] = [];
    req.on('data', (chunk: Buffer) => chunks.push(chunk));
    req.on('end', () => resolve(Buffer.concat(chunks)));
    req.on('error', reject);
  });
}
```

**為何使用 Raw Body？**
- 避免 NestJS body parser 干擾 multipart/form-data 格式
- 直接轉發原始二進位資料至 FastAPI
- 保持檔案完整性

#### 2. 檔案下載 (Stream Piping)

```typescript
async proxyDownload(path: string, res: Response): Promise<void> {
  const response = await firstValueFrom(
    this.httpService.get(url, { responseType: 'stream' }),
  );

  // 轉發 headers
  res.setHeader('Content-Type', response.headers['content-type']);
  res.setHeader('Content-Disposition', response.headers['content-disposition']);

  // 串流轉發
  response.data.pipe(res);
}
```

**為何使用串流？**
- 避免大檔案佔用記憶體
- 支援即時下載（不需等待完整檔案載入）
- 提升效能與使用者體驗

#### 3. 錯誤處理

所有代理方法使用 RxJS `firstValueFrom` 處理 Observable：

```typescript
const { data } = await firstValueFrom(this.httpService.get(url, config));
```

若 FastAPI 返回錯誤 (4xx/5xx)，會自動拋出例外並由 NestJS 統一處理。

## Controllers

### 1. UploadController

**路由前綴**: `/api/v1/upload`

| 端點 | 方法 | 說明 |
|------|------|------|
| `/` | POST | 上傳單一或多個檔案 |
| `/:fileId` | GET | 取得檔案中繼資料 |

**特殊處理**：
- POST 端點使用 `@Req()` 取得原始請求
- 手動讀取 raw body 並轉發
- 保留原始 `content-type` header

### 2. CleanController

**路由前綴**: `/api/v1/clean`

| 端點 | 方法 | 說明 |
|------|------|------|
| `/` | POST | 啟動清洗任務 |
| `/preview` | POST | 預覽清洗結果（不實際執行） |
| `/:taskId/result` | GET | 取得清洗結果詳情 |

### 3. TasksController

**路由前綴**: `/api/v1/tasks`

| 端點 | 方法 | 說明 |
|------|------|------|
| `/` | GET | 列出所有任務（支援分頁與篩選） |
| `/:taskId` | GET | 取得單一任務狀態 |
| `/:taskId` | DELETE | 取消執行中任務 |

**查詢參數**：
- `limit`: 每頁筆數
- `offset`: 偏移量
- `status_filter`: 狀態篩選（pending/processing/completed/failed/cancelled）

### 4. RulesController

**路由前綴**: `/api/v1/rules`

| 端點 | 方法 | 說明 |
|------|------|------|
| `/` | GET | 列出所有規則設定檔 |
| `/` | POST | 建立新規則設定檔 |
| `/:profileId` | GET | 取得單一規則設定檔 |
| `/:profileId` | PUT | 更新規則設定檔 |
| `/:profileId` | DELETE | 刪除規則設定檔 |
| `/:profileId/export` | GET | 匯出單一規則為 JSON |
| `/export` | POST | 批次匯出多個規則 |
| `/import` | POST | 匯入規則 JSON |

### 5. DownloadController

**路由前綴**: `/api/v1/download`

| 端點 | 方法 | 說明 |
|------|------|------|
| `/:taskId` | GET | 下載所有清洗後檔案 (ZIP) |
| `/:taskId/report` | GET | 下載清洗報告 (JSON) |
| `/:taskId/:fileId` | GET | 下載單一清洗後檔案 |

**特殊處理**：
- 所有方法使用 `@Res()` 取得 Express Response
- 透過 `proxyDownload` 串流轉發
- 自動設定正確的 Content-Type 和 Content-Disposition

## 安全性

### 1. 雙重驗證

```typescript
@UseGuards(JwtAuthGuard, RolesGuard)
@Roles('admin')
```

**流程**：
1. `JwtAuthGuard` 驗證 JWT Token 有效性
2. `RolesGuard` 檢查使用者角色是否為 `admin`
3. 若驗證失敗，返回 401 Unauthorized 或 403 Forbidden

### 2. 環境變數隔離

FastAPI URL 透過環境變數設定：

```typescript
this.baseUrl = process.env.FASTAPI_BASE_URL || 'http://localhost:8000';
```

**優點**：
- 避免硬編碼
- 支援不同環境（開發/測試/生產）
- 方便更換後端服務

## 模組設定

### HttpModule 設定

```typescript
HttpModule.register({
  timeout: 300000,      // 5 分鐘
  maxRedirects: 5,
})
```

**為何設定 5 分鐘 Timeout？**
- 去識別化任務可能處理大量檔案
- 預留足夠時間進行複雜清洗邏輯
- 避免任務執行中被中斷

## 型別安全

所有程式碼符合 TypeScript Strict Mode：

- ✅ 明確的回傳型別
- ✅ 避免 `any` 型別（代理回應使用 `unknown`）
- ✅ 完整的 null 檢查
- ✅ 參數型別定義

範例：

```typescript
async proxyGet(path: string, query?: Record<string, string>): Promise<unknown> {
  const url = `${this.baseUrl}${path}`;
  const config: AxiosRequestConfig = { params: query };
  const { data } = await firstValueFrom(this.httpService.get(url, config));
  return data;
}
```

## 測試建議

### 1. 單元測試

測試 `CleaningProxyService` 的每個方法：

```typescript
describe('CleaningProxyService', () => {
  let service: CleaningProxyService;
  let httpService: HttpService;

  beforeEach(() => {
    // Mock HttpService
  });

  it('should proxy GET request', async () => {
    // Test proxyGet
  });

  it('should handle errors from FastAPI', async () => {
    // Test error handling
  });
});
```

### 2. 整合測試

測試 Controller 與 Guards 整合：

```typescript
describe('UploadController (e2e)', () => {
  it('should reject requests without JWT token', () => {
    return request(app.getHttpServer())
      .post('/api/v1/upload')
      .expect(401);
  });

  it('should reject non-admin users', () => {
    return request(app.getHttpServer())
      .post('/api/v1/upload')
      .set('Authorization', `Bearer ${userToken}`)
      .expect(403);
  });

  it('should allow admin to upload files', () => {
    return request(app.getHttpServer())
      .post('/api/v1/upload')
      .set('Authorization', `Bearer ${adminToken}`)
      .attach('files', '/path/to/file.pdf')
      .expect(200);
  });
});
```

## 效能考量

### 1. 串流處理

檔案上傳與下載皆使用串流，避免記憶體溢出：

- ✅ 上傳：raw body chunks 逐步讀取
- ✅ 下載：pipe 方式直接轉發至 Response

### 2. HTTP 連線池

`@nestjs/axios` 預設使用 Axios 的連線池機制，重複使用 TCP 連線。

### 3. Timeout 設定

避免長時間佔用連線資源，設定合理的 timeout。

## 維護注意事項

### 1. 保持路徑一致

NestJS 代理層路徑必須與 FastAPI 完全一致：

- ❌ NestJS: `/api/v1/tasks`, FastAPI: `/api/v1/task`
- ✅ NestJS: `/api/v1/tasks`, FastAPI: `/api/v1/tasks`

### 2. 同步更新

若 FastAPI 新增端點，需同步在對應 Controller 新增方法。

### 3. 錯誤訊息轉發

確保 FastAPI 的錯誤訊息能正確傳遞至前端，避免資訊遺失。

## 擴展方向

1. **API 文件**
   - 加入 Swagger decorators (`@ApiTags`, `@ApiOperation`)
   - 自動生成 OpenAPI 規範

2. **日誌記錄**
   - 記錄所有代理請求與回應
   - 方便除錯與稽核

3. **快取機制**
   - 對靜態資料（如規則設定檔列表）實作快取
   - 減少對 FastAPI 的重複請求

4. **Rate Limiting**
   - 防止 API 濫用
   - 保護 FastAPI 後端

5. **健康檢查**
   - 定期檢查 FastAPI 可用性
   - 返回服務狀態至監控系統

## 相關文件

- [Cleaning API 完整規範](/docs/api/cleaning-api.md)
- [Cleaning Proxy 使用說明](/docs/api/cleaning-proxy.md)
- [安裝與設定指南](/docs/setup/cleaning-proxy-setup.md)
