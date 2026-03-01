# Auth 模組依賴安裝指南

Auth 模組已成功建立,但需要安裝以下依賴套件才能正常運作。

## 必要套件

在專案根目錄執行以下指令:

```bash
cd /Users/puppychen/Job/EcMap/zProjectsSource/oda-cyber-konsult

# 安裝 NestJS Auth 相關套件
pnpm add --filter @oda-cyber/api \
  @nestjs/jwt \
  @nestjs/passport \
  passport \
  passport-jwt \
  bcrypt

# 安裝型別定義 (devDependencies)
pnpm add -D --filter @oda-cyber/api \
  @types/passport-jwt \
  @types/bcrypt
```

## 套件說明

| 套件 | 用途 |
|------|------|
| `@nestjs/jwt` | NestJS JWT 整合 |
| `@nestjs/passport` | NestJS Passport 整合 |
| `passport` | 驗證中介軟體核心 |
| `passport-jwt` | JWT 驗證策略 |
| `bcrypt` | 密碼雜湊 |
| `@types/passport-jwt` | TypeScript 型別定義 |
| `@types/bcrypt` | TypeScript 型別定義 |

## 環境變數設定

在 `.env` 檔案中新增:

```env
JWT_SECRET=your-super-secret-key-change-in-production-min-32-chars
```

> 建議在正式環境使用至少 32 字元的隨機字串

## 驗證安裝

安裝完成後,啟動開發伺服器:

```bash
pnpm --filter @oda-cyber/api dev
```

應該不會出現模組找不到的錯誤。

## API 端點

Auth 模組提供以下端點:

- `POST /api/auth/login` - 使用者登入
- `POST /api/auth/register` - 使用者註冊
- `POST /api/auth/refresh` - 刷新 Token

## 測試範例

### 註冊新使用者

```bash
curl -X POST http://localhost:4000/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "email": "test@example.com",
    "password": "Test123456",
    "name": "Test User",
    "role": "user"
  }'
```

### 登入

```bash
curl -X POST http://localhost:4000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "email": "test@example.com",
    "password": "Test123456"
  }'
```

回應範例:
```json
{
  "success": true,
  "data": {
    "accessToken": "eyJhbGciOiJIUzI1...",
    "refreshToken": "eyJhbGciOiJIUzI1...",
    "user": {
      "id": "uuid-here",
      "email": "test@example.com",
      "name": "Test User",
      "role": "user"
    }
  },
  "timestamp": "2026-02-09T..."
}
```

### 使用 Token 存取受保護端點

```bash
curl http://localhost:4000/protected-endpoint \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

## 使用範例

### 保護特定路由

```typescript
import { Controller, Get, UseGuards } from '@nestjs/common';
import { JwtAuthGuard } from '../../common/guards/jwt-auth.guard';
import { CurrentUser } from '../../common/decorators/current-user.decorator';

@Controller('profile')
@UseGuards(JwtAuthGuard)
export class ProfileController {
  @Get()
  getProfile(@CurrentUser() user: any) {
    return { message: 'This is protected', user };
  }
}
```

### 限制特定角色

```typescript
import { Controller, Get, UseGuards } from '@nestjs/common';
import { JwtAuthGuard } from '../../common/guards/jwt-auth.guard';
import { RolesGuard } from '../../common/guards/roles.guard';
import { Roles } from '../../common/decorators/roles.decorator';

@Controller('admin')
@UseGuards(JwtAuthGuard, RolesGuard)
export class AdminController {
  @Get()
  @Roles('admin')
  adminOnly() {
    return { message: 'Admin only endpoint' };
  }
}
```
