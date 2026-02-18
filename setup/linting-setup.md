# ESLint 9 + Prettier + Husky + GitHub Actions CI 設定指南

## 概述

本專案已完成 Phase 2-C-13/14/15 的 Linting 與 CI/CD 設定，包含：

- **ESLint 9** (flat config) - TypeScript + React 程式碼檢查
- **Prettier** - 程式碼格式化
- **Husky + lint-staged** - Git pre-commit 自動檢查
- **GitHub Actions CI** - 自動化測試與建置

## 安裝步驟

### 1. 安裝依賴套件

```bash
cd /Users/puppychen/Job/EcMap/zProjectsSource/oda-cyber-konsult
pnpm install
```

### 2. 初始化 Husky

```bash
npx husky init
chmod +x .husky/pre-commit
```

## 設定檔案說明

### ESLint 設定 (`eslint.config.mjs`)

使用 ESLint 9 的 flat config 格式，整合：
- `@eslint/js` - JavaScript 基礎規則
- `typescript-eslint` - TypeScript 規則
- `eslint-plugin-react-hooks` - React Hooks 規則
- `eslint-plugin-react-refresh` - React Refresh 規則
- `eslint-config-prettier` - 停用與 Prettier 衝突的規則

**關鍵規則：**
- 未使用變數警告（允許 `_` 前綴忽略）
- `any` 類型警告（測試檔除外）
- 空物件類型關閉檢查
- React 前端專案啟用 Hooks 與 Refresh 規則

### Prettier 設定 (`.prettierrc`)

```json
{
  "semi": true,
  "singleQuote": true,
  "trailingComma": "all",
  "printWidth": 120,
  "tabWidth": 2,
  "endOfLine": "lf"
}
```

### Lint-staged 設定 (`package.json`)

Git commit 前自動執行：
- `*.{ts,tsx}` → `eslint --fix` + `prettier --write`
- `*.{json,md,yaml,yml}` → `prettier --write`

## NPM Scripts

| 指令 | 說明 |
|------|------|
| `pnpm lint` | 檢查所有程式碼（根目錄 ESLint） |
| `pnpm lint:fix` | 自動修正 ESLint 錯誤 |
| `pnpm lint:workspaces` | 執行各 workspace 的 lint（透過 Turbo） |
| `pnpm format` | 格式化所有檔案 |
| `pnpm format:check` | 檢查格式（CI 用） |

## GitHub Actions CI

位於 `.github/workflows/ci.yml`，包含兩個 Job：

### Job 1: `lint-and-test` (TypeScript)

1. 啟動 PostgreSQL 17 服務
2. 安裝 Node.js 22 + pnpm 9
3. 執行 `pnpm lint` 與 `pnpm format:check`
4. 建置所有專案（shared-types → api → admin → chatbot）
5. 執行 API 測試

### Job 2: `python-test`

1. 安裝 Python 3.12 + uv
2. 測試 `data-pipeline` 與 `rag-service`

## Git Workflow

### Pre-commit Hook

當執行 `git commit` 時，Husky 會自動：
1. 檢查暫存檔案的 ESLint 錯誤
2. 自動格式化程式碼
3. 僅在通過檢查後允許 commit

### 手動跳過（不建議）

```bash
git commit --no-verify -m "message"
```

## 常見問題

### Q1: ESLint 與 Prettier 衝突？
A: 已透過 `eslint-config-prettier` 停用衝突規則，Prettier 只負責格式，ESLint 負責邏輯檢查。

### Q2: 為何使用 ESLint 9 flat config？
A: ESLint 9 改用 `eslint.config.mjs` 取代舊的 `.eslintrc`，更靈活且支援 TypeScript import。

### Q3: CI 失敗如何除錯？
A: 本地執行相同指令：
```bash
pnpm install --frozen-lockfile
pnpm lint
pnpm format:check
pnpm build
pnpm test
```

### Q4: 如何在編輯器中啟用自動修正？
**VS Code：**
```json
{
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": true
  }
}
```

## 檔案清單

```
/Users/puppychen/Job/EcMap/zProjectsSource/oda-cyber-konsult/
├── .github/
│   └── workflows/
│       └── ci.yml                  # GitHub Actions CI
├── .husky/
│   └── pre-commit                  # Pre-commit hook
├── eslint.config.mjs               # ESLint 9 設定
├── .prettierrc                     # Prettier 設定
├── .prettierignore                 # Prettier 忽略清單
└── package.json                    # 新增 scripts 與 lint-staged
```

## 驗證設定

### 測試 ESLint

```bash
pnpm lint
```

### 測試 Prettier

```bash
pnpm format:check
```

### 測試 Husky + lint-staged

```bash
# 修改任意 .ts 檔案
git add .
git commit -m "test: husky hook"
# 應該會自動執行 lint-staged
```

### 測試 CI

推送至 GitHub 後，在 Actions 頁面查看執行結果。

## 後續維護

- **升級依賴**：`pnpm update -r`
- **新增規則**：編輯 `eslint.config.mjs`
- **修改格式**：編輯 `.prettierrc`
- **調整 CI**：編輯 `.github/workflows/ci.yml`

---

**完成日期**: 2026-02-11
**負責人**: Claude Code
**階段**: Phase 2-C-13/14/15
