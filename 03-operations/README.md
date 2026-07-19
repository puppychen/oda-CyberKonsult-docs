---
audience: human-primary
purpose: reference
status: approved
owner: ODA Cyber Konsult
---

# 運維與部署文件（Delivery Loop — QG-5）

> ODA Cyber Konsult — 環境建置、部署指南、運維手冊、故障排除

本目錄涵蓋系統從開發環境到正式環境的所有運維文件，對應 AI-Native SDLC 的 Delivery Loop 階段（Quality Gate 5）。

---

## 文件清單

### 運維手冊

| 文件 | 說明 | 更新日期 |
|------|------|---------|
| [runbook.md](./runbook.md) | 運維手冊 — 服務拓撲、健康檢查、Token 用途／認證 Session 與故障排除 | 2026-07-13 |
| [qg5-readiness.md](./qg5-readiness.md) | QG-5 交付準備檢核 — 部署、監控、安全、回滾、交接缺口 | 2026-04-29 |

### 環境建置

| 文件 | 說明 | 適用對象 | 更新日期 |
|------|------|---------|---------|
| [getting-started.md](./getting-started.md) | 快速開始 — 5 分鐘建立開發環境 | 新成員 | 2026-02-04 |
| [development.md](./development.md) | 開發環境完整說明 — Node.js/Python/Docker 設定、環境變數、啟動指令 | 開發者 | 2026-02-11 |
| [dev-environment-setup.md](./dev-environment-setup.md) | 本機測試環境 — 以 `.env` 為 PostgreSQL 連線單一來源、服務啟動與排錯 | 開發者／AI 代理 | 2026-07-17 |
| [deployment.md](./deployment.md) | 部署與維運指南 — Docker Compose、GCP、VM、Prisma 0009 與 Alembic 011／012 驗證／回復 | DevOps | 2026-07-13 |
| [staging-setup.md](./staging-setup.md) | Staging 環境設置、migration 備份與驗證 | DevOps | 2026-07-12 |
| [backup-strategy.md](./backup-strategy.md) | 備份策略與保存規則 | DevOps | 2026-04-17 |
| [dr-checklist.md](./dr-checklist.md) | Disaster Recovery 檢核表 | DevOps | 2026-04-17 |
| [data-retention-policy.md](./data-retention-policy.md) | 資料保存與刪除政策 | DevOps / Compliance | 2026-04-17 |

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
| 確認是否可進 QG-5 | [qg5-readiness.md](./qg5-readiness.md) |
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
| diagrams index | [../02-design/diagrams/README.md](../02-design/diagrams/README.md) | 部署圖與系統圖索引 |
| ARCH.md ADR-011 | [../01-specs/architecture/ARCH.md](../01-specs/architecture/ARCH.md) | Contracts as Code 架構決策（影響 Delivery Loop 部署期環境變數與設定載入）|
