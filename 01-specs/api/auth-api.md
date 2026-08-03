---
audience: both
purpose: reference
status: approved
owner: ODA Cyber Konsult
---

# Auth API 文件

> **版本**: v1.5
> **基礎路徑**: `/api/auth`
> **服務**: NestJS API (Port 3051)

## 概述

> **TL;DR**：系統使用 JWT 驗證身分，並以 PostgreSQL `auth_sessions` 管理各應用程式／裝置的獨立 Refresh Token。Access Token 為 60 分鐘，Refresh Token 為 7 天且每次使用後輪替；暫時性連線錯誤不應清除前端頁面或輸入內容。

Auth API 提供註冊、登入、Token 刷新、目前／全部登出、Cleaner SSO 交換與密碼變更。JWT 負責請求簽章驗證，`auth_sessions` 負責 Refresh Token 輪替、撤銷與工作階段上限，並遵循台灣資通安全「普」級密碼政策。

## 技術規格

- **驗證方式**: JWT (Access Token + Refresh Token)
- **密碼雜湊**: bcrypt (cost factor: 12 for passwords, 10 for refresh tokens)
- **Token 有效期**:
  - Access Token: 60 分鐘
  - Refresh Token: 7 天
- **密碼政策**（資通安全「普」級）:
  - 最少 8 字元
  - 至少包含大寫英文、小寫英文、數字、特殊字元其中 3 種
  - 90 天過期強制變更
  - 前 2 代密碼不可重複使用
- **帳號鎖定**: 連續 5 次登入失敗鎖定 15 分鐘；失敗次數與鎖定時間由 PostgreSQL 單一原子 `UPDATE` 寫入，並行請求不會互相覆蓋
- **工作階段**: 每次登入依 `clientType` 建立獨立 Session，每帳號最多 10 個有效 Session
- **Refresh 輪替**: Refresh JWT 具有 `sid` 與唯一 `jti`，資料庫以條件更新確保單次使用
- **Token 用途**: 新發 Token 以 `tokenUse=access|refresh` 明確隔離；Refresh Token 不能作為 Bearer Access Token
- **支援角色**: `basic_user`, `user`, `consultant`, `data_cleaner`, `data_reviewer`, `admin`

## 端點列表

| 方法 | 路徑 | 說明 | 需驗證 |
|------|------|------|--------|
| POST | `/api/auth/register` | 註冊新使用者 | 否 |
| POST | `/api/auth/login` | 使用者登入 | 否 |
| POST | `/api/auth/refresh` | 刷新 Access Token | 否 |
| POST | `/api/auth/logout` | 使用者登出 | 是 (JWT) |
| POST | `/api/auth/logout-all` | 撤銷帳號全部工作階段 | 是 (JWT) |
| POST | `/api/auth/sso/exchange` | 建立獨立 Cleaner 工作階段 | 是 (JWT) |
| POST | `/api/auth/change-password` | 變更密碼 | 是 (JWT) |

---

## 端點詳細說明

### 1. 使用者註冊

建立新的使用者帳號。

**端點**: `POST /api/auth/register`

#### 請求

```http
POST /api/auth/register HTTP/1.1
Host: localhost:3051
Content-Type: application/json

{
  "email": "user@example.com",
  "password": "SecurePass123!",
  "name": "John Doe",
  "role": "basic_user"
}
```

#### 請求參數

| 欄位 | 類型 | 必填 | 說明 | 驗證規則 |
|------|------|------|------|----------|
| `email` | string | 是 | 使用者電子郵件 | 有效的 Email 格式 |
| `password` | string | 是 | 使用者密碼 | 至少 8 字元 + 3/4 複雜度 (大寫/小寫/數字/特殊字元) |
| `name` | string | 否 | 使用者姓名 | 先移除前後空白，再以 Unicode 字元計算最多 30 個字元；超限回傳 400 |
| `role` | string | 否 | 使用者角色 | 公開註冊固定為 `basic_user`；管理員可於後台指派六種角色 |

#### 回應

**成功 (201 Created)**

```json
{
  "success": true,
  "data": {
    "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "user": {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "email": "user@example.com",
      "name": "John Doe",
      "role": "basic_user"
    }
  },
  "timestamp": "2026-02-09T13:45:30.123Z"
}
```

**錯誤 (409 Conflict) - Email 已存在**

```json
{
  "statusCode": 409,
  "message": "Email already registered",
  "error": "Conflict",
  "timestamp": "2026-02-09T13:45:30.123Z",
  "path": "/api/auth/register"
}
```

**錯誤 (400 Bad Request) - 驗證失敗**

```json
{
  "statusCode": 400,
  "message": [
    "email must be an email",
    "密碼至少 8 字元，且須包含大寫英文、小寫英文、數字、特殊字元中的至少 3 種"
  ],
  "error": "Bad Request",
  "timestamp": "2026-02-09T13:45:30.123Z",
  "path": "/api/auth/register"
}
```

---

### 2. 使用者登入

使用 Email 與密碼進行身份驗證。

**端點**: `POST /api/auth/login`

#### 請求

```http
POST /api/auth/login HTTP/1.1
Host: localhost:3051
Content-Type: application/json

{
  "email": "user@example.com",
  "password": "SecurePass123!",
  "clientType": "chatbot"
}
```

#### 請求參數

| 欄位 | 類型 | 必填 | 說明 | 驗證規則 |
|------|------|------|------|----------|
| `email` | string | 是 | 使用者電子郵件 | 有效的 Email 格式 |
| `password` | string | 是 | 使用者密碼 | 至少 8 字元 |
| `clientType` | enum | 否 | 呼叫端類型 | `admin`、`cleaner`、`chatbot`、`api`；預設 `api` |

#### 回應

**成功 (200 OK)**

```json
{
  "success": true,
  "data": {
    "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "sessionId": "7a6e5c4d-1234-4a9b-8123-abcdef123456",
    "clientType": "chatbot",
    "user": {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "email": "user@example.com",
      "name": "John Doe",
      "role": "user"
    },
    "requirePasswordChange": false
  },
  "timestamp": "2026-02-09T13:45:30.123Z"
}
```

> **注意**: 當 `requirePasswordChange` 為 `true` 時,表示密碼已過期或管理員要求強制變更。前端應導向密碼變更頁面,且除了 change-password、logout、me、refresh 外的端點都會回傳 403。

**錯誤 (401 Unauthorized) - 憑證無效**

```json
{
  "statusCode": 401,
  "message": "Invalid credentials",
  "error": "Unauthorized",
  "timestamp": "2026-02-09T13:45:30.123Z",
  "path": "/api/auth/login"
}
```

**錯誤 (401 Unauthorized) - 帳號已停用**

```json
{
  "statusCode": 401,
  "message": "Account is deactivated",
  "error": "Unauthorized",
  "timestamp": "2026-02-09T13:45:30.123Z",
  "path": "/api/auth/login"
}
```

**錯誤 (423 Locked) - 帳號已鎖定**

```json
{
  "statusCode": 423,
  "message": "帳號已暫時鎖定，請稍後再試",
  "error": "Unauthorized"
}
```

---

### 3. 刷新 Token

使用 Refresh Token 取得新的 Access Token 與 Refresh Token。每次成功刷新都維持相同 `sessionId`，但產生新的 `jti` 與 Token 對；舊 Refresh Token 再次使用會撤銷該工作階段。

**端點**: `POST /api/auth/refresh`

#### 請求

```http
POST /api/auth/refresh HTTP/1.1
Host: localhost:3051
Content-Type: application/json
X-ODA-Auth-Session-Protocol: 2

{
  "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

#### 請求參數

| 欄位 | 類型 | 必填 | 說明 |
|------|------|------|------|
| `refreshToken` | string | 否 | 先前取得的 Refresh Token | 前端應優先放在 body；未提供時可使用相容性 httpOnly Cookie |

Header `X-ODA-Auth-Session-Protocol` 必須為 `2`。未提供或版本不符時 API 回 426，且不驗證、不雜湊、也不輪替 Refresh Token；此 rollout fence 用來阻止仍開啟的舊版頁面與新版頁面同時操作同一 Session。

#### 回應

**成功 (200 OK)**

```json
{
  "success": true,
  "data": {
    "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "sessionId": "7a6e5c4d-1234-4a9b-8123-abcdef123456",
    "clientType": "chatbot"
  },
  "timestamp": "2026-02-09T13:45:30.123Z"
}
```

**錯誤 (401 Unauthorized) - Token 無效**

```json
{
  "statusCode": 401,
  "message": "Invalid refresh token",
  "error": "Unauthorized",
  "timestamp": "2026-02-09T13:45:30.123Z",
  "path": "/api/auth/refresh"
}
```

---

### 4. 使用者登出

撤銷 Bearer Access Token 內 `sid` 對應的目前工作階段，不影響同帳號的其他裝置或應用程式。

**端點**: `POST /api/auth/logout`

**認證**: 需要 JWT Bearer Token

#### 請求

```http
POST /api/auth/logout HTTP/1.1
Host: localhost:3051
Authorization: Bearer YOUR_ACCESS_TOKEN
```

#### 回應

**成功 (200 OK)**

```json
{
  "success": true,
  "data": { "message": "登出成功" },
  "timestamp": "2026-02-09T13:45:30.123Z"
}
```

---

### 4A. 全部登出

`POST /api/auth/logout-all` 需要 Bearer Access Token。成功後撤銷該帳號所有 `auth_sessions`、清除舊版 `users.refresh_token`，並遞增 `tokenVersion`，使既有 Access Token 立即失效。

### 4B. Cleaner SSO 交換

`POST /api/auth/sso/exchange` 需要 Admin／Cleaner 授權角色的 Bearer Access Token。成功後建立 `clientType=cleaner` 的獨立工作階段並回傳新的 Access／Refresh Token；Admin Refresh Token 不會透過 `postMessage` 或 URL 傳給 Cleaner。

---

### 5. 變更密碼

變更當前使用者的密碼。密碼過期或管理員強制變更時,使用者必須透過此端點更新密碼。

**端點**: `POST /api/auth/change-password`

**認證**: 需要 JWT Bearer Token（即使密碼已過期,此端點仍可存取）

#### 請求

```http
POST /api/auth/change-password HTTP/1.1
Host: localhost:3051
Authorization: Bearer YOUR_ACCESS_TOKEN
Content-Type: application/json

{
  "oldPassword": "OldPass123!",
  "newPassword": "NewSecure456@"
}
```

#### 請求參數

| 欄位 | 類型 | 必填 | 說明 | 驗證規則 |
|------|------|------|------|----------|
| `oldPassword` | string | 是 | 目前使用的密碼 | 至少 8 字元 |
| `newPassword` | string | 是 | 新密碼 | 至少 8 字元 + 3/4 複雜度、不可與目前密碼相同、不可與前 2 代密碼相同 |

#### 回應

**成功 (200 OK)**

```json
{
  "success": true,
  "data": {
    "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "user": {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "email": "user@example.com",
      "name": "John Doe",
      "role": "user"
    },
    "requirePasswordChange": false
  },
  "timestamp": "2026-02-09T13:45:30.123Z"
}
```

> **注意**: 密碼變更成功後會回傳新的 Token 對,前端應更新儲存的 Token。

**錯誤 (400 Bad Request) - 舊密碼錯誤**

```json
{
  "statusCode": 400,
  "message": "舊密碼錯誤",
  "error": "Bad Request"
}
```

**錯誤 (400 Bad Request) - 新密碼與目前密碼相同**

```json
{
  "statusCode": 400,
  "message": "新密碼不可與目前密碼相同",
  "error": "Bad Request"
}
```

**錯誤 (400 Bad Request) - 密碼歷史重複**

```json
{
  "statusCode": 400,
  "message": "新密碼不可與前 2 次使用過的密碼相同",
  "error": "Bad Request"
}
```

---

## JWT Payload 結構

Access Token 和 Refresh Token 包含以下 Payload:

```json
{
  "sub": "550e8400-e29b-41d4-a716-446655440000",
  "email": "user@example.com",
  "role": "user",
  "tokenVersion": 0,
  "sid": "7a6e5c4d-1234-4a9b-8123-abcdef123456",
  "tokenUse": "access 或 refresh",
  "jti": "僅 Refresh Token 具有的唯一識別碼",
  "iat": 1707484730,
  "exp": 1707485630
}
```

| 欄位 | 說明 |
|------|------|
| `sub` | 使用者 ID (UUID) |
| `email` | 使用者 Email |
| `role` | 使用者角色 |
| `tokenVersion` | 憑證版本；密碼、Email、角色或狀態變更與全部登出時遞增 |
| `sid` | `auth_sessions.id`；Access／Refresh Token 共用 |
| `tokenUse` | Token 用途；受保護端點只接受 `access`，刷新端點只接受 `refresh` |
| `jti` | Refresh Token 單次輪替識別碼；Access Token 不含此欄 |
| `iat` | Token 簽發時間 (Unix timestamp) |
| `exp` | Token 過期時間 (Unix timestamp) |

---

## 受保護端點的使用方式

### 使用 JWT Guard

在需要驗證的端點加上 `@UseGuards(JwtAuthGuard)`:

```typescript
import { Controller, Get, UseGuards } from '@nestjs/common';
import { JwtAuthGuard } from '../../common/guards/jwt-auth.guard';
import { CurrentUser } from '../../common/decorators/current-user.decorator';

@Controller('profile')
@UseGuards(JwtAuthGuard)
export class ProfileController {
  @Get()
  getProfile(@CurrentUser() user: any) {
    return {
      message: 'This is a protected endpoint',
      user,
    };
  }
}
```

### 請求範例

```http
GET /profile HTTP/1.1
Host: localhost:3051
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

### 使用角色控制

同時使用 `JwtAuthGuard` 和 `RolesGuard`:

```typescript
import { Controller, Get, UseGuards } from '@nestjs/common';
import { JwtAuthGuard } from '../../common/guards/jwt-auth.guard';
import { RolesGuard } from '../../common/guards/roles.guard';
import { Roles } from '../../common/decorators/roles.decorator';

@Controller('admin')
@UseGuards(JwtAuthGuard, RolesGuard)
export class AdminController {
  @Get('users')
  @Roles('admin')
  listUsers() {
    return { message: 'Admin only endpoint' };
  }

  @Get('stats')
  @Roles('admin', 'consultant')
  getStats() {
    return { message: 'Admin and consultant can access' };
  }
}
```

---

## 錯誤碼彙整

| HTTP Status | Error Code | 說明 | 解決方式 |
|-------------|------------|------|----------|
| 400 | Bad Request | 請求參數驗證失敗 | 檢查請求格式與必填欄位 |
| 401 | Unauthorized | Access Token 或 Refresh Token 已過期、撤銷或重播 | Access 過期先 Refresh；Refresh 401 才要求同頁重新登入 |
| 423 | Locked | 登入失敗達上限，帳號暫時鎖定 | 等待鎖定時間結束或由管理員處理 |
| 429 | Too Many Requests | 登入／Refresh 或一般 API 超過速率限制 | 保留頁面與 Token，依 `Retry-After` 重試 |
| 403 | Forbidden | 權限不足 | 確認使用者角色是否符合要求 |
| 403 | PASSWORD_CHANGE_REQUIRED | 密碼已過期或需強制變更 | 呼叫 `/api/auth/change-password` 變更密碼 |
| 409 | Conflict | Email 已存在 | 使用其他 Email 或直接登入 |
| 426 | AUTH_SESSION_PROTOCOL_UPGRADE_REQUIRED | 前端認證協定過舊 | 保留目前頁面與輸入，重新整理載入新版前端後再登入 |
| 500 | Internal Server Error | 伺服器內部錯誤 | 聯繫系統管理員 |

---

## 安全機制

### 1. 密碼政策（資通安全「普」級合規）

- **最少長度**: 8 字元
- **複雜度**: 至少包含大寫英文、小寫英文、數字、特殊字元其中 3 種
- **過期機制**: 90 天自動過期（可透過 `PASSWORD_MAX_AGE_DAYS` 環境變數調整）
- **歷史限制**: 前 2 代密碼不可重複（可透過 `PASSWORD_HISTORY_COUNT` 調整）
- **強制變更**: 管理員可設定 `forcePasswordChange` 旗標要求使用者下次登入時變更密碼
- **後端強制**: `PasswordChangeRequiredGuard` 全域攔截,密碼過期時除了白名單端點外一律回傳 403

### 2. 帳號鎖定

- 連續 5 次登入失敗,帳號鎖定 15 分鐘
- 成功登入後自動重置失敗計數器

### 3. NestJS → FastAPI 內部 API 認證

- 使用 `X-Internal-Token` Shared Secret header 認證
- 環境變數: `INTERNAL_API_KEY` (NestJS) / `RAG_INTERNAL_API_KEY` (FastAPI)
- 選用 IP 白名單: `RAG_ALLOWED_HOSTS`

### 4. JWT_SECRET 設定

在正式環境務必使用強度足夠的隨機字串:

```bash
# .env
JWT_SECRET=use-a-strong-random-string-at-least-32-characters-long
```

> 非 production 環境未設定 JWT_SECRET 時會顯示警告。production 環境未設定將拒絕啟動。

### 5. HTTPS 傳輸

正式環境務必使用 HTTPS,避免 Token 在傳輸過程中被攔截。

### 6. Token 儲存

- 目前 Admin、Cleaner、Chatbot 各自使用不同 `localStorage` key 保存 Access／Refresh Token，後端另設定相容性 httpOnly Cookie
- 所有前端都必須維持 CSP、React 輸出轉義與相依套件安全更新，降低 localStorage 遭 XSS 讀取的風險
- Cleaner SSO 僅透過可信任 origin 的 `postMessage` 傳 Admin Access Token，再由後端交換獨立 Cleaner Token；不得把 Refresh Token 放入 URL 或跨應用傳遞

### 7. Token 刷新策略

- Access Token 過期後，下一次 API 請求收到 401 時自動刷新並重送原請求
- Refresh 請求固定帶 `X-ODA-Auth-Session-Protocol: 2`；API 在進入 Token 驗證與輪替前，以 426 拒絕缺少或版本不符的請求。部署時必須先讓新版 API 接管 100% 流量並排空舊 replica，再發布三個前端
- 同分頁共用一個 Refresh Promise；跨分頁使用 Web Locks，無 Web Locks 時使用 Bakery-style 多鍵競爭者互斥與 5 秒有效期、每 2 秒續期的 localStorage 租約。競爭者被選出後仍須最後檢查舊版單鍵 lease；執行前寫入 lease 並讀回確認 owner，若其他分頁剛取得 lease 或 claim 未成功，移除本分頁 contender 並等待後重新競爭
- 登入、同頁重新驗證、Cleaner SSO、密碼變更、登出與 Refresh 的 Token 寫入／清除都使用同一跨分頁互斥鎖，避免非 Refresh 流程與正在落地的 Refresh 回應互相覆寫
- 只在 Refresh 端點回 400／401或本機無 Refresh Token 時清除 Token，並在原頁要求重新登入
- 網路錯誤、429、5xx、格式異常或新 Access Token 重試仍為 401 時，保留 Token、頁面與輸入內容並顯示暫時性錯誤
- 同頁重新登入的帳號欄唯讀，回傳使用者 ID 必須等於原工作階段；不同帳號登入會被拒絕並撤銷該次新工作階段。Refresh 在 HTTP 回應抵達後及 JSON 解析完成後都重驗 Access、Refresh 與使用者快照；若另一分頁已切換帳號，不清除或覆寫新帳號 Token，成功但被捨棄的舊 Session 會立即撤銷
- 使用者選擇「稍後處理」後，重複背景 401 只更新提示，不自行重開視窗；提示列仍可手動開啟重新登入

---

## 完整使用流程範例

### 1. 註冊新使用者

```bash
curl -X POST http://localhost:3051/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "email": "john@example.com",
    "password": "SecurePass123!",
    "name": "John Doe",
    "role": "user"
  }'
```

### 2. 使用者登入

```bash
curl -X POST http://localhost:3051/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "email": "john@example.com",
    "password": "SecurePass123!"
  }'
```

回應中取得 `accessToken` 和 `refreshToken`。

### 3. 使用 Access Token 存取受保護資源

```bash
curl http://localhost:3051/protected-endpoint \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

### 4. Token 過期時刷新

```bash
curl -X POST http://localhost:3051/api/auth/refresh \
  -H "Content-Type: application/json" \
  -d '{
    "refreshToken": "YOUR_REFRESH_TOKEN"
  }'
```

---

## 資料模型參考

### User Model (Prisma Schema)

```prisma
model User {
  id                  String    @id @default(uuid())
  email               String    @unique
  password            String?
  name                String?
  role                String    @default("basic_user")
  isActive            Boolean   @default(true) @map("is_active")
  loginAttempts       Int       @default(0) @map("login_attempts")
  lockedUntil         DateTime? @map("locked_until")
  lastLogin           DateTime? @map("last_login")
  refreshToken        String?   @map("refresh_token")
  passwordChangedAt   DateTime? @map("password_changed_at")
  forcePasswordChange Boolean   @default(false) @map("force_password_change")
  createdAt           DateTime  @default(now()) @map("created_at")
  updatedAt           DateTime  @updatedAt @map("updated_at")
  passwordHistories   PasswordHistory[]
  authSessions        AuthSession[]
}

model PasswordHistory {
  id             String   @id @default(uuid())
  userId         String   @map("user_id")
  hashedPassword String   @map("hashed_password")
  createdAt      DateTime @default(now()) @map("created_at")
  user           User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  @@index([userId])
  @@map("password_histories")
}

model AuthSession {
  id               String    @id
  userId           String    @map("user_id")
  clientType       String    @map("client_type")
  refreshTokenHash String    @map("refresh_token_hash")
  currentJti       String    @unique @map("current_jti")
  expiresAt        DateTime  @map("expires_at")
  revokedAt        DateTime? @map("revoked_at")
  lastUsedAt       DateTime  @default(now()) @map("last_used_at")
  createdAt        DateTime  @default(now()) @map("created_at")
  updatedAt        DateTime  @updatedAt @map("updated_at")
  user             User      @relation(fields: [userId], references: [id], onDelete: Cascade)
  @@index([userId, revokedAt])
  @@index([expiresAt])
  @@map("auth_sessions")
}
```

`users.refresh_token` 暫時保留，只用於 migration 0009 上線前既有 Refresh Token 的第一次惰性轉換；新登入與新刷新都寫入 `auth_sessions`。

### 支援的角色

| 角色 | 代碼 | 說明 | 權限範圍 |
|------|------|------|----------|
| 新手 | `basic_user` | 免費方案使用者 | beginner 回應層級；每日額度受後台設定控制 |
| 一般 | `user` | IT/MIS 付費方案使用者 | beginner、standard 回應層級 |
| 顧問 | `consultant` | 專業付費方案使用者 | beginner、standard、expert 回應層級 |
| 資料清洗員 | `data_cleaner` | 清洗審核操作 | 清洗任務審核、知識庫管理 |
| 資料審核員 | `data_reviewer` | Maker-Checker 審批 | 批准、退回與送入知識庫 |
| 管理員 | `admin` | 系統管理員 | 完整系統管理權限 |

---

## 相關檔案

### 核心檔案

- `apps/api/src/modules/auth/auth.module.ts` - Auth 模組定義
- `apps/api/src/modules/auth/services/auth.service.ts` - 身份驗證邏輯
- `apps/api/src/modules/auth/services/token.service.ts` - Token 生成與驗證
- `apps/api/src/modules/auth/controllers/auth.controller.ts` - HTTP 端點
- `apps/api/src/modules/auth/strategies/jwt.strategy.ts` - JWT 驗證策略
- `apps/api/src/modules/auth/repositories/auth-session.repository.ts` - 工作階段建立、輪替與撤銷

### DTO

- `apps/api/src/modules/auth/dto/login.dto.ts` - 登入請求 DTO
- `apps/api/src/modules/auth/dto/register.dto.ts` - 註冊請求 DTO
- `apps/api/src/modules/auth/dto/change-password.dto.ts` - 變更密碼請求 DTO

### Guards & Decorators

- `apps/api/src/common/guards/jwt-auth.guard.ts` - JWT 驗證 Guard
- `apps/api/src/common/guards/roles.guard.ts` - 角色權限 Guard
- `apps/api/src/common/decorators/current-user.decorator.ts` - 取得當前使用者
- `apps/api/src/common/decorators/roles.decorator.ts` - 角色標記裝飾器
- `apps/api/src/common/guards/password-change-required.guard.ts` - 密碼過期強制變更 Guard
- `apps/api/src/common/decorators/skip-password-check.decorator.ts` - 跳過密碼過期檢查
- `apps/api/src/common/validators/password-strength.validator.ts` - 密碼強度驗證器

---

## 變更歷史

| 版本 | 日期 | 說明 |
|------|------|------|
| v1.0 | 2026-02-09 | 初始版本,包含註冊、登入、Token 刷新功能 |
| v1.1 | 2026-02-24 | 資通安全「普」級密碼政策合規,新增變更密碼與登出端點,NestJS→FastAPI 內部 API 認證 |
| v1.2 | 2026-07-13 | 新增獨立工作階段、Refresh 原子輪替、目前／全部登出、Cleaner SSO、同頁重新登入與 migration 0009 |
| v1.3 | 2026-07-13 | 新增 `tokenUse` 用途隔離、同帳號重新登入、租約續期與跨分頁帳號切換保護 |
| v1.4 | 2026-07-13 | Admin、Cleaner、Chatbot 共用的 Access Token 效期延長為 60 分鐘；Refresh Token 維持 7 天 |

---

## 常見問題 (FAQ)

### Q1: Access Token 過期後要怎麼處理?

使用 Refresh Token 呼叫 `/api/auth/refresh` 端點取得新的 Token 對。建議在前端實作自動刷新機制。

### Q2: 如何實作登出功能?

呼叫 `POST /api/auth/logout` 只撤銷目前工作階段；需要讓所有裝置失效時呼叫 `POST /api/auth/logout-all`。前端在後端請求完成或失敗後都清除目前應用程式的本機 Token。

### Q3: 忘記密碼怎麼辦?

目前版本尚未實作密碼重置功能,後續版本將加入:
- `POST /api/auth/forgot-password` - 發送重置郵件
- `POST /api/auth/reset-password` - 使用 Token 重置密碼

### Q4: 可以同時登入多個裝置嗎?

可以。每次登入都會在 `auth_sessions` 建立獨立工作階段，最多保留 10 個有效工作階段；超出時撤銷最久未使用者。一般登出只影響目前工作階段。

### Q5: 如何提升密碼安全性?

系統已實作完整的密碼安全機制:
- 密碼複雜度要求 (至少 3/4 種字元類型)
- 密碼歷史記錄 (前 2 代不可重複)
- 90 天密碼過期自動提醒與強制變更
- 帳號鎖定 (5 次失敗鎖定 15 分鐘)
- 後端 Guard 全域攔截未變更密碼的請求
