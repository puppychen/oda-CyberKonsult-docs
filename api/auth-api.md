# Auth API 文件

> **版本**: v1
> **基礎路徑**: `/api/auth`
> **服務**: NestJS API (Port 4000)

## 概述

Auth API 提供使用者身份驗證與授權功能,包含註冊、登入、Token 刷新等核心功能。系統採用 JWT (JSON Web Token) 進行無狀態驗證。

## 技術規格

- **驗證方式**: JWT (Access Token + Refresh Token)
- **密碼雜湊**: bcrypt (cost factor: 10)
- **Token 有效期**:
  - Access Token: 15 分鐘
  - Refresh Token: 7 天
- **支援角色**: `user`, `it_user`, `consultant`, `admin`

## 端點列表

| 方法 | 路徑 | 說明 | 需驗證 |
|------|------|------|--------|
| POST | `/api/auth/register` | 註冊新使用者 | 否 |
| POST | `/api/auth/login` | 使用者登入 | 否 |
| POST | `/api/auth/refresh` | 刷新 Access Token | 否 |

---

## 端點詳細說明

### 1. 使用者註冊

建立新的使用者帳號。

**端點**: `POST /api/auth/register`

#### 請求

```http
POST /api/auth/register HTTP/1.1
Host: localhost:4000
Content-Type: application/json

{
  "email": "user@example.com",
  "password": "SecurePass123",
  "name": "John Doe",
  "role": "user"
}
```

#### 請求參數

| 欄位 | 類型 | 必填 | 說明 | 驗證規則 |
|------|------|------|------|----------|
| `email` | string | 是 | 使用者電子郵件 | 有效的 Email 格式 |
| `password` | string | 是 | 使用者密碼 | 至少 6 字元 |
| `name` | string | 否 | 使用者姓名 | - |
| `role` | string | 否 | 使用者角色 | `user`, `it_user`, `consultant`, `admin` (預設: `user`) |

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
      "role": "user"
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
    "password must be longer than or equal to 6 characters"
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
Host: localhost:4000
Content-Type: application/json

{
  "email": "user@example.com",
  "password": "SecurePass123"
}
```

#### 請求參數

| 欄位 | 類型 | 必填 | 說明 | 驗證規則 |
|------|------|------|------|----------|
| `email` | string | 是 | 使用者電子郵件 | 有效的 Email 格式 |
| `password` | string | 是 | 使用者密碼 | 至少 6 字元 |

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
    }
  },
  "timestamp": "2026-02-09T13:45:30.123Z"
}
```

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

---

### 3. 刷新 Token

使用 Refresh Token 取得新的 Access Token。

**端點**: `POST /api/auth/refresh`

#### 請求

```http
POST /api/auth/refresh HTTP/1.1
Host: localhost:4000
Content-Type: application/json

{
  "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

#### 請求參數

| 欄位 | 類型 | 必填 | 說明 |
|------|------|------|------|
| `refreshToken` | string | 是 | 先前取得的 Refresh Token |

#### 回應

**成功 (200 OK)**

```json
{
  "success": true,
  "data": {
    "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
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

## JWT Payload 結構

Access Token 和 Refresh Token 包含以下 Payload:

```json
{
  "sub": "550e8400-e29b-41d4-a716-446655440000",
  "email": "user@example.com",
  "role": "user",
  "iat": 1707484730,
  "exp": 1707485630
}
```

| 欄位 | 說明 |
|------|------|
| `sub` | 使用者 ID (UUID) |
| `email` | 使用者 Email |
| `role` | 使用者角色 |
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
Host: localhost:4000
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
| 401 | Unauthorized | 驗證失敗 (密碼錯誤、Token 過期等) | 重新登入取得新 Token |
| 403 | Forbidden | 權限不足 | 確認使用者角色是否符合要求 |
| 409 | Conflict | Email 已存在 | 使用其他 Email 或直接登入 |
| 500 | Internal Server Error | 伺服器內部錯誤 | 聯繫系統管理員 |

---

## 安全性建議

### 1. JWT_SECRET 設定

在正式環境務必使用強度足夠的隨機字串:

```bash
# .env
JWT_SECRET=use-a-strong-random-string-at-least-32-characters-long
```

### 2. HTTPS 傳輸

正式環境務必使用 HTTPS,避免 Token 在傳輸過程中被攔截。

### 3. Token 儲存

- 前端應將 Token 儲存於 `httpOnly` Cookie 或記憶體中
- 避免儲存於 localStorage (易受 XSS 攻擊)

### 4. Token 刷新策略

- Access Token 過期前自動刷新
- Refresh Token 過期時要求使用者重新登入

### 5. 帳號安全

- 實作帳號鎖定機制 (多次登入失敗)
- 記錄登入歷史與異常行為
- 提供雙因素驗證 (2FA) 選項

---

## 完整使用流程範例

### 1. 註冊新使用者

```bash
curl -X POST http://localhost:4000/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "email": "john@example.com",
    "password": "SecurePass123",
    "name": "John Doe",
    "role": "user"
  }'
```

### 2. 使用者登入

```bash
curl -X POST http://localhost:4000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "email": "john@example.com",
    "password": "SecurePass123"
  }'
```

回應中取得 `accessToken` 和 `refreshToken`。

### 3. 使用 Access Token 存取受保護資源

```bash
curl http://localhost:4000/protected-endpoint \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

### 4. Token 過期時刷新

```bash
curl -X POST http://localhost:4000/api/auth/refresh \
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
  id        String   @id @default(uuid())
  email     String   @unique
  password  String?
  name      String?
  role      String   @default("user")
  isActive  Boolean  @default(true)
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}
```

### 支援的角色

| 角色 | 代碼 | 說明 | 權限範圍 |
|------|------|------|----------|
| 一般使用者 | `user` | 新手模式使用者 | 基礎諮詢功能 |
| IT 使用者 | `it_user` | 一般模式使用者 | 進階諮詢功能 |
| 顧問 | `consultant` | 顧問模式使用者 | 專業分析與報告 |
| 管理員 | `admin` | 系統管理員 | 完整系統管理權限 |

---

## 相關檔案

### 核心檔案

- `apps/api/src/modules/auth/auth.module.ts` - Auth 模組定義
- `apps/api/src/modules/auth/services/auth.service.ts` - 身份驗證邏輯
- `apps/api/src/modules/auth/services/token.service.ts` - Token 生成與驗證
- `apps/api/src/modules/auth/controllers/auth.controller.ts` - HTTP 端點
- `apps/api/src/modules/auth/strategies/jwt.strategy.ts` - JWT 驗證策略

### DTO

- `apps/api/src/modules/auth/dto/login.dto.ts` - 登入請求 DTO
- `apps/api/src/modules/auth/dto/register.dto.ts` - 註冊請求 DTO

### Guards & Decorators

- `apps/api/src/common/guards/jwt-auth.guard.ts` - JWT 驗證 Guard
- `apps/api/src/common/guards/roles.guard.ts` - 角色權限 Guard
- `apps/api/src/common/decorators/current-user.decorator.ts` - 取得當前使用者
- `apps/api/src/common/decorators/roles.decorator.ts` - 角色標記裝飾器

---

## 變更歷史

| 版本 | 日期 | 說明 |
|------|------|------|
| v1.0 | 2026-02-09 | 初始版本,包含註冊、登入、Token 刷新功能 |

---

## 常見問題 (FAQ)

### Q1: Access Token 過期後要怎麼處理?

使用 Refresh Token 呼叫 `/api/auth/refresh` 端點取得新的 Token 對。建議在前端實作自動刷新機制。

### Q2: 如何實作登出功能?

由於採用無狀態 JWT,登出功能需在前端實作:
1. 清除儲存的 Token
2. (選用) 呼叫後端 API 將 Refresh Token 加入黑名單

### Q3: 忘記密碼怎麼辦?

目前版本尚未實作密碼重置功能,後續版本將加入:
- `POST /api/auth/forgot-password` - 發送重置郵件
- `POST /api/auth/reset-password` - 使用 Token 重置密碼

### Q4: 可以同時登入多個裝置嗎?

可以。每次登入都會產生新的 Token 對,不會使舊的 Token 失效。如需單一裝置登入,需在資料庫記錄 Token 並實作驗證邏輯。

### Q5: 如何提升密碼安全性?

建議實作以下機制:
- 密碼複雜度要求 (大小寫、數字、特殊字元)
- 密碼歷史記錄 (避免重複使用舊密碼)
- 定期提醒使用者更換密碼
- 雙因素驗證 (2FA)
