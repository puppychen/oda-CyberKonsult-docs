# 運維與部署文件（Delivery Loop — QG-5）

> ODA Cyber Konsult — 環境建置、部署指南、運維手冊、故障排除

本目錄涵蓋系統從開發環境到正式環境的所有運維文件，對應 AI-Native SDLC 的 Delivery Loop 階段（Quality Gate 5）。

---

## 文件清單

### 運維手冊

| 文件 | 說明 | 更新日期 |
|------|------|---------|
| [runbook.md](./runbook.md) | 運維手冊 — 服務拓撲、健康檢查端點、故障排除（7 種情境）、備份還原、安全事件應變 | 2026-03-01 |

### 環境建置

| 文件 | 說明 | 適用對象 | 更新日期 |
|------|------|---------|---------|
| [getting-started.md](./getting-started.md) | 快速開始 — 5 分鐘建立開發環境 | 新成員 | 2026-02-04 |
| [development.md](./development.md) | 開發環境完整說明 — Node.js/Python/Docker 設定、環境變數、啟動指令 | 開發者 | 2026-02-11 |
| [deployment.md](./deployment.md) | 部署與維運指南 — Docker Compose、GCP、VM 部署方式、Nginx 設定 | DevOps | 2026-02-11 |

### 模組設定

| 文件 | 說明 | 適用對象 | 更新日期 |
|------|------|---------|---------|
| [auth-deps.md](./auth-deps.md) | 認證模組依賴安裝 — bcrypt、JWT、Passport 套件 | 開發者 | 2026-02-09 |
| [cleaning-proxy-setup.md](./cleaning-proxy-setup.md) | 清洗代理層設定 — NestJS 代理轉發 FastAPI 設定 | 開發者 | 2026-02-09 |
| [gdrive-setup.md](./gdrive-setup.md) | Google Drive 整合設定 — OAuth 憑證、Webhook、同步設定 | 管理員 | 2026-02-13 |
| [linting-setup.md](./linting-setup.md) | ESLint + Prettier + Husky 設定 — 程式碼品質工具鏈 | 開發者 | 2026-02-11 |

---

## 快速導引

| 我想要... | 前往 |
|----------|------|
| 第一次建立環境 | [getting-started.md](./getting-started.md) → [development.md](./development.md) |
| 部署到正式環境 | [deployment.md](./deployment.md) |
| 排除線上故障 | [runbook.md](./runbook.md) |
| 設定 Google Drive 同步 | [gdrive-setup.md](./gdrive-setup.md) |
| 設定程式碼品質工具 | [linting-setup.md](./linting-setup.md) |

---

## 相關文件

| 文件 | 位置 | 說明 |
|------|------|------|
| system-overview.md | [../01-specs/architecture/system-overview.md](../01-specs/architecture/system-overview.md) | 系統架構與服務拓撲 |
| health-api.md | [../01-specs/api/health-api.md](../01-specs/api/health-api.md) | 健康檢查端點規格 |
| test-guide.md | [../02-testing/test-guide.md](../02-testing/test-guide.md) | 測試啟動與驗證 |
