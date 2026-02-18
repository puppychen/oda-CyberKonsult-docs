# ODA Cyber Konsult - 文件總覽

三層式 AI 資安顧問系統，基於 RAG 技術搭配在地化法規知識庫，服務中小企業。

## 需求規格

| 文件 | 說明 |
|------|------|
| [specs/SRS_BUSINESS.md](./specs/SRS_BUSINESS.md) | 商業需求規格書 |
| [specs/SRS_TECHNICAL.md](./specs/SRS_TECHNICAL.md) | 技術需求規格書 |
| [SRS_BUSINESS.docx](./SRS_BUSINESS.docx) | 商業需求 Word 原始版本 |

## 系統架構

| 文件 | 說明 |
|------|------|
| [architecture/system-overview.md](./architecture/system-overview.md) | 整體架構圖與服務拓撲 |
| [architecture/rag-system-blueprint.md](./architecture/rag-system-blueprint.md) | RAG 系統 9 階段建構指南 |
| [architecture/rag-pipeline.md](./architecture/rag-pipeline.md) | RAG 管線流程（載入/切塊/索引/檢索） |

## API 文件

| 文件 | 說明 |
|------|------|
| [api/README.md](./api/README.md) | API 總索引（含認證規範、流程圖） |
| [api/auth-api.md](./api/auth-api.md) | 認證與授權 `/api/auth` |
| [api/chat-api.md](./api/chat-api.md) | 聊天與對話 `/api/chat` |
| [api/audit-users-api.md](./api/audit-users-api.md) | 使用者管理 `/api/users` |
| [api/prompts-api.md](./api/prompts-api.md) | 提示詞管理 `/api/prompts` |
| [api/health-api.md](./api/health-api.md) | 健康檢查 `/health` |
| [api/cleaning-proxy.md](./api/cleaning-proxy.md) | NestJS 清洗代理 `/api/v1/*` |
| [api/cleaning-api.md](./api/cleaning-api.md) | FastAPI 去識別化 `/api/v1/*` |
| [api/rag-api.md](./api/rag-api.md) | RAG 查詢與文件管理 `/api/v1/rag` |
| [api/gdrive-api.md](./api/gdrive-api.md) | Google Drive 資料來源 `/api/datasources/gdrive` |
| [api/data-pipeline-loaders-api.md](./api/data-pipeline-loaders-api.md) | 文件解析器 API（PDF/DOCX/XLSX/PPTX 等） |

## 環境建置與部署

| 文件 | 說明 |
|------|------|
| [setup/getting-started.md](./setup/getting-started.md) | 快速開始（5 分鐘上手） |
| [setup/development.md](./setup/development.md) | 開發環境完整說明 |
| [setup/deployment.md](./setup/deployment.md) | 部署與維運指南 |
| [setup/auth-deps.md](./setup/auth-deps.md) | 認證模組依賴安裝 |
| [setup/cleaning-proxy-setup.md](./setup/cleaning-proxy-setup.md) | 清洗代理層設定 |
| [setup/gdrive-setup.md](./setup/gdrive-setup.md) | Google Drive 整合設定 |
| [setup/linting-setup.md](./setup/linting-setup.md) | ESLint + Prettier + Husky 設定 |

## 實作說明

| 文件 | 說明 |
|------|------|
| [implementation/nestjs-modules.md](./implementation/nestjs-modules.md) | NestJS 模組架構總覽 |
| [implementation/chat-module.md](./implementation/chat-module.md) | 聊天模組實作細節 |
| [implementation/chatbot-frontend.md](./implementation/chatbot-frontend.md) | Chatbot UI 前端實作 |

## 操作指南

| 文件 | 說明 |
|------|------|
| [guides/chat-quick-start.md](./guides/chat-quick-start.md) | 聊天功能快速上手 |
| [guides/rag-cookbook.md](./guides/rag-cookbook.md) | RAG + 清洗整合食譜 |

## 測試

| 文件 | 說明 |
|------|------|
| [testing/api-e2e-testing.md](./testing/api-e2e-testing.md) | E2E 測試架構與執行方式 |
