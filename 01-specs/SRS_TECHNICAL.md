---
audience: ai-primary
purpose: spec
status: approved
owner: ODA Cyber Konsult
---

# 軟體需求規格書 (SRS) - 完整技術規格

> **ODA Cyber Konsult - 資安助手 RAG 系統**
>
> 文件版本：2.8.1
> 最後更新：2026-07-29
> 文件類型：完整技術 SRS（面向開發人員、系統架構師、QA 工程師）

---

## 目錄

1. [文件概述](#1-文件概述)
2. [專案背景與目標](#2-專案背景與目標)
3. [系統架構](#3-系統架構)
4. [使用者角色與權限](#4-使用者角色與權限)
5. [功能需求 (FR)](#5-功能需求-fr)
6. [非功能需求 (NFR)](#6-非功能需求-nfr)
7. [資料模型](#7-資料模型)
8. [API 規格總覽](#8-api-規格總覽)
9. [介面設計規格](#9-介面設計規格)
10. [安全需求](#10-安全需求)
11. [驗收標準](#11-驗收標準)
12. [附錄](#12-附錄)

---

## 狀態圖例

- ✅ 已實作 (Phase 1 完成)
- 🔄 規劃中 (Phase 2 開發中)
- 🔮 未來願景 (Phase 3 規劃)

---

## 1. 文件概述

### 1.1 文件目的與範圍

本軟體需求規格書定義 **ODA Cyber Konsult 資安助手 RAG 系統** 的完整技術規格，涵蓋所有功能階段。

**本文件涵蓋範圍**：
- ✅ 已實作功能的完整規格
- 🔄 規劃中功能的詳細設計
- 🔮 未來擴充功能的技術藍圖

### 1.2 目標讀者

| 讀者類型 | 關注章節 |
|----------|----------|
| 系統架構師 | 第 3、7、8 章 |
| 後端開發人員 | 第 5、7、8、10 章 |
| 前端開發人員 | 第 5、8、9 章 |
| QA 工程師 | 第 5、6、11 章 |
| DevOps 工程師 | 第 3、6、10 章 |

### 1.3 術語定義

| 術語 | 定義 |
|------|------|
| **RAG** | Retrieval-Augmented Generation，檢索增強生成技術 |
| **PII** | Personally Identifiable Information，個人可識別資訊 |
| **去識別化** | 將敏感資料轉換為無法識別原始資料的形式（Data De-identification） |
| **實體類型** | PII 的分類，如人名、身分證字號、電話號碼等 |
| **固定去識別化規則** | 系統內建的 6 種去識別化策略組合，由 `get_default_rules()` 統一定義，不支援自訂規則 CRUD |
| **知識庫** | 經向量化處理後可供 RAG 檢索的文件集合 |
| **Embedding** | 將文字轉換為向量表示的過程 |
| **多會員** | 單一系統服務多個獨立組織（會員）的架構模式 |
| **RBAC** | Role-Based Access Control，基於角色的存取控制 |
| **SLA** | Service Level Agreement，服務水準協議 |
| **三層式 AI 回應機制** | 提供新手(beginner)、一般(standard)、顧問(expert)三種回應深度的機制 |
| **新手回應層級** | 以白話文、故事化方式回應，適合非技術背景使用者 |
| **一般回應層級** | 提供實務技術建議與操作步驟，適合 IT/MIS 工程師 |
| **顧問回應層級** | 提供深入專業分析，含 ISO 標準引用與法規依據，適合資安顧問 |
| **在地化法規知識庫** | 以台灣資通安全管理法、個資法及 ISO 27001/27701 等標準建構的 RAG 知識庫 |
| **Cleaner App** | 獨立的清洗審核管理前端應用 (Port 5503)，提供完整的審核工作流 |
| **審核工作流** | 查看 → 標籤 → 編輯 → 批准 → 送入 RAG 的完整清洗品質審核流程 |
| **Analytics Dashboard** | 資料分析儀表板，提供清洗統計、知識庫統計與時間軸統計 |

### 1.4 參考文件

| 文件名稱 | 位置 | 說明 |
|----------|------|------|
| PROJECT_CONTEXT.md | `/PROJECT_CONTEXT.md` | 專案全貌與技術架構 |
| SRS_BUSINESS.md | `/docs/01-specs/SRS_BUSINESS.md` | 商業需求規格書 |
| cleaning-api.md | `/docs/01-specs/api/cleaning-api.md` | 去識別化 API 完整規格 |
| rag-api.md | `/docs/01-specs/api/rag-api.md` | RAG 服務 API 完整規格 |
| gdrive-api.md | `/docs/01-specs/api/gdrive-api.md` | Google Drive 整合 API 規格 |

---

## 2. 專案背景與目標

### 2.1 專案願景

建立一個 **企業級智慧型資安助手平台**，提供：
1. **智慧問答**：基於 RAG 的資安知識查詢
2. **三層式 AI 顧問**：提供新手／一般／顧問三種回應層級
3. **在地化法規知識庫**：內建台灣資安法規與國際 ISO 標準
4. **資料保護**：完整的 PII 偵測與去識別化能力
5. **多會員服務**：支援多組織獨立運營 🔮
6. **合規支援**：符合台灣個資法與 GDPR 要求

### 2.2 業務目標

| 目標編號 | 業務目標 | 關鍵指標 |
|----------|----------|----------|
| BG-01 | 提升資安諮詢效率 | 查詢回應時間 < 10 秒 |
| BG-02 | 確保敏感資料安全 | PII 偵測準確率 > 95% |
| BG-03 | 符合法規要求 | 支援台灣個資法規定的資料類型 |
| BG-04 | 降低人工審查成本 | 自動化去識別化處理 |
| BG-05 | 提供分層式資安諮詢 | 三層式 AI 回應機制上線 |
| BG-06 | 建立在地化法規知識庫 | 法規覆蓋率 ≥ 90%（台灣資安相關法規） |

### 2.3 成功指標

| 指標 | 目標值 | 階段 |
|------|--------|------|
| 系統可用性 | ≥ 99.5% | Phase 1-2 |
| 系統可用性 | ≥ 99.9% | Phase 3 |
| 查詢回應時間 | P95 < 10 秒 | Phase 1 |
| 查詢回應時間 | P95 < 7 秒 | Phase 2-3 |
| PII 偵測召回率 | ≥ 95% | Phase 1 |
| PII 偵測召回率 | ≥ 98% | Phase 2 |
| 支援會員數 | - | Phase 1-2 |
| 支援會員數 | ≥ 100 | Phase 3 |
| AI 使用者滿意度 | ≥ 85% | Phase 2 |
| AI 回答準確率 | ≥ 95% | Phase 2 |
| 法規知識庫覆蓋率 | ≥ 90% | Phase 1 |
| 月活躍使用者數 | ≥ 50 | Phase 2 |

---

## 3. 系統架構

### 3.1 完整架構圖

```
                    ┌──────────────────────────────────────────────────┐
                    │                 Load Balancer 🔮                  │
                    └──────────────────────┬───────────────────────────┘
                                           │
     ┌─────────────────┬───────────────────┼───────────────┬──────────────────┐
     │                 │                   │               │                  │
     ▼                 ▼                   ▼               ▼                  ▼
┌──────────┐  ┌───────────────┐  ┌───────────────┐  ┌──────────┐  ┌─────────────────┐
│ Chatbot  │  │    Admin      │  │   Cleaner     │  │   API    │  │   API Gateway   │
│ UI       │  │  Dashboard    │  │   App         │  │ Gateway  │  │   🔮            │
│ React 19 │  │  React 19     │  │  React 19     │  │   🔮     │  │                 │
│ :5502    │  │  :5501        │  │  :5503        │  │          │  │                 │
└────┬─────┘  └───────┬───────┘  └───────┬───────┘  └──────────┘  └────────┬────────┘
     │                │                  │                                  │
     └────────────────┴──────────────────┴──────────────────────────────────┘
                                         │
                          ┌──────────────┴──────────────┐
                          │                             │
                          ▼                             ▼
               ┌─────────────────┐           ┌─────────────────┐
               │   NestJS API    │           │  Auth Service   │
               │   port: 4000    │           │  🔮 Keycloak    │
               │                 │           │                 │
               │  - REST API     │           │  - OAuth2       │
               │  - WebSocket    │           │  - SSO          │
               │  - LLM 編排    │           │  - MFA          │
               │  - GraphQL 🔮   │           │                 │
               └────────┬────────┘           └─────────────────┘
                        │
          ┌─────────────┼─────────────┬──────────────┬──────────────┐
          │             │             │              │              │
          ▼             ▼             ▼              ▼              ▼
    ┌─────────────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐
    │ RAG + Cleaning  │  │ Gemini/  │  │ SearXNG  │  │ Notification │
    │ Service         │  │ OpenAI   │  │ (Docker) │  │ Service      │
    │ FastAPI         │  │ LLM API  │  │ :8080    │  │ 🔮           │
    │ :3502 (檢索專用)│  │          │  │ 網路搜尋 │  │              │
    └────────┬────────┘  └──────────┘  └──────────┘  └──────┬───────┘
             │                                              │
             └──────────────────────────────────────────────┘
                                   │
             ┌─────────────────────┼─────────────────────┐
             │                     │                     │
             ▼                     ▼                     ▼
    ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐
    │  PostgreSQL 17  │   │     Qdrant      │   │     Redis 🔮    │
    │                 │   │  (Vector DB)    │   │                 │
    │  - 主資料庫     │   │  - 向量儲存     │   │  - Session      │
    │  - 會員隔離 🔮  │   │  - 知識庫索引   │   │  - 任務佇列     │
    └─────────────────┘   └─────────────────┘   └─────────────────┘
```

### 3.2 技術棧詳細列表

| 層級 | 技術選擇 | 版本 | 狀態 | 說明 |
|------|----------|------|------|------|
| **Monorepo** | pnpm workspaces + Turborepo | pnpm 9.15+ | ✅ | TypeScript 專案管理 |
| **API Gateway** | Kong / Nginx | - | 🔮 | 流量管理、限流、路由 |
| **認證服務** | Keycloak | - | 🔮 | OAuth2、SSO、MFA |
| **後端 API** | NestJS + TypeScript | NestJS 11, TS 5.7+ | ✅ | RESTful API + WebSocket |
| **GraphQL** | Apollo Server | - | 🔮 | 彈性查詢介面 |
| **ORM (Node)** | Prisma | Latest | ✅ | NestJS 端資料庫存取 |
| **前端 (Admin)** | React + Vite + Ant Design | React 19, Vite 6, Ant Design v6 | ✅ | 管理後台 |
| **前端 (Chatbot)** | React + Vite + TailwindCSS | React 19, Vite 6, Tailwind v4 | ✅ | 聊天介面 |
| **前端 (Cleaner)** | React + Vite + Ant Design + TanStack React Query | React 19, Vite 6, Ant Design v6 | ✅ | 清洗審核管理介面 |
| **Python 管理** | uv workspace | Python 3.12+ | ✅ | Python 套件管理 |
| **RAG 服務** | FastAPI + Qdrant | - | ✅ | 向量檢索與文件索引（僅檢索，不含 LLM 生成） |
| **LLM 編排** | NestJS + @google/generative-ai + openai | - | ✅ | NestJS 直接呼叫 Gemini/OpenAI 進行回答生成 |
| **網路搜尋** | SearXNG + Cheerio | Docker | ✅ | RAG 不足時的網路搜尋補充 |
| **ORM (Python)** | SQLAlchemy + Alembic | SQLAlchemy 2.0 | ✅ | Python 端資料庫存取 |
| **PII 偵測** | Microsoft Presidio + spaCy | Presidio 2.2+ | ✅ | 個資偵測引擎 |
| **分析服務** | FastAPI + Pandas | - | 🔮 | 報表與分析 |
| **通知服務** | NestJS + FCM | - | 🔮 | 推播與 Email |
| **快取/佇列** | Redis 7 | - | 🔮 | Session、任務佇列 |
| **關聯式資料庫** | PostgreSQL | 17 | ✅ | 主資料儲存 |
| **向量資料庫** | Qdrant | Latest | ✅ | 文件向量檢索 |
| **物件儲存** | MinIO | - | 🔮 | 檔案儲存 |
| **監控** | Prometheus + Grafana | - | 🔮 | 系統監控 |
| **日誌** | ELK Stack | - | 🔮 | 集中日誌管理 |

### 3.3 服務通訊矩陣

| 來源服務 | 目標服務 | 協定 | 狀態 | 說明 |
|----------|----------|------|------|------|
| Frontend (Admin/Chatbot/Cleaner) | NestJS API | HTTP/WS | ✅ | 主要 API 通訊 |
| NestJS API | RAG + Cleaning Service | HTTP | ✅ | RAG 檢索與清洗任務（NestJS 呼叫 /retrieve） |
| NestJS API | Gemini/OpenAI API | HTTPS | ✅ | LLM 回答生成（NestJS 直接呼叫） |
| NestJS API | SearXNG | HTTP | ✅ | 網路搜尋補充（Docker 內部 8080） |
| NestJS API | PostgreSQL | TCP | ✅ | 資料存取 |
| RAG Service | Qdrant | HTTP/gRPC | ✅ | 向量檢索 |
| RAG Service | PostgreSQL | TCP | ✅ | 資料存取 |
| API Gateway | NestJS API | HTTP | 🔮 | 主要 API 路由 |
| API Gateway | Auth Service | HTTP | 🔮 | 認證請求 |
| NestJS API | Analytics Service | HTTP | 🔮 | 報表查詢 |
| NestJS API | Redis | TCP | 🔮 | Session、快取 |
| Notification Service | FCM | HTTP | 🔮 | 推播通知 |
| Notification Service | SMTP | TCP | 🔮 | Email 發送 |

### 3.4 連線埠分配

| 服務 | Port | 協定 | 狀態 | 說明 |
|------|------|------|------|------|
| NestJS API | 3051 | HTTP/WS | ✅ | 核心後端 API |
| Admin Dashboard | 5501 | HTTP | ✅ | 管理後台 |
| Chatbot UI | 5502 | HTTP | ✅ | 使用者聊天介面 |
| Cleaner UI | 5503 | HTTP | ✅ | 清洗審核管理介面 |
| RAG + Cleaning Service (FastAPI) | 3502 | HTTP | ✅ | RAG 與清洗服務（已合併） |
| PostgreSQL | 5432 | TCP | ✅ | 關聯式資料庫 |
| Qdrant REST | 6333 | HTTP | ✅ | 向量資料庫 REST |
| Qdrant gRPC | 6334 | gRPC | ✅ | 向量資料庫 gRPC |
| SearXNG | 8080 (Docker) | HTTP | ✅ | 網路搜尋引擎（僅 Docker 內部） |
| Redis | 6379 | TCP | 🔮 | 快取與佇列 |
| Keycloak | 8080 | HTTP | 🔮 | 認證服務 |

---

## 4. 使用者角色與權限

### 4.1 角色定義

| 角色 | 代碼 | 狀態 | 說明 |
|------|------|------|------|
| 新手（Free） | `basic_user` | ✅ | 僅使用 beginner 回應層級；每日訊息額度由後台設定 |
| 一般（Plus） | `user` | ✅ | IT/MIS 使用者；使用 beginner / standard 回應層級 |
| 資安 ISO 顧問 | `consultant` | ✅ | 透過專業化 AI 對話進行深入資安諮詢 |
| 資料清洗人員 | `data_cleaner` | ✅ | 清洗資料審核，僅存取 Cleaner App |
| 資料審核人員 | `data_reviewer` | ✅ | 清洗資料審批，僅存取 Cleaner App（Maker-Checker 審批者） |
| 系統管理員 | `admin` | ✅ | 系統設定與維護，可存取 Admin Dashboard + Cleaner App + Chatbot |
> **RFC-001 規範**：`it_user` 已移除並併入 `user`。本文件其他歷史範例若仍出現 `it_user`，僅供版本追溯，不得作為新資料或授權依據。

### 4.1.1 現行角色與應用程式存取矩陣

| 應用程式 | basic_user | user | consultant | data_cleaner | data_reviewer | admin |
|----------|:----------:|:----:|:----------:|:------------:|:-------------:|:-----:|
| Chatbot UI | ✓ | ✓ | ✓ | ✗ | ✗ | ✓ |
| Admin Dashboard | ✗ | ✗ | ✗ | ✗ | ✗ | ✓ |
| Cleaner App | ✗ | ✗ | ✗ | ✓ | ✓ | ✓ |

管理帳號、資料清理與 Maker-Checker 權限維持原規則；此次只調整客戶角色、回應層級、提示詞與免費額度。

### 4.3 舊版角色與應用程式存取矩陣（僅供歷史追溯）

| 應用程式 | `basic_user` | `user` | `it_user` | `consultant` | `data_cleaner` | `data_reviewer` | `admin` | `platform_admin` |
|----------|:------------:|:------:|:---------:|:------------:|:--------------:|:---------------:|:-------:|:-----------------:|
| Chatbot UI (5174) | ✓ | ✓ | ✓ | ✓ | ✗ | ✗ | ✓ | ✓ |
| Admin Dashboard (5173) | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✓ | ✓ |
| Cleaner App (5175) | ✗ | ✗ | ✗ | ✗ | ✓ | ✓ | ✓ | ✓ |

### 4.2 舊版詳細權限矩陣（僅供歷史追溯）

| 功能 | basic_user | user | it_user | consultant | data_cleaner | data_reviewer | admin | platform_admin |
|------|:----------:|:----:|:-------:|:----------:|:------------:|:-------------:|:-----:|:--------------:|
| **Chatbot** |
| Chatbot 對話 | ✓ | ✓ | ✓ | ✓ | ✗ | ✗ | ✓ | ✓ |
| 查看自己歷史對話 | ✓ | ✓ | ✓ | ✓ | ✗ | ✗ | ✓ | ✓ |
| 查看所有歷史對話 | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✓ | ✓ |
| 查看自己對話使用紀錄 | ✗ | ✗ | ✗ | ✓ | ✗ | ✗ | ✓ | ✓ |
| **回應模式** |
| 選擇回應模式 | ✗ | ✓ | ✓ | ✓ | ✗ | ✗ | ✓ | ✓ |
| **去識別化功能（僅 Admin）** |
| 上傳檔案 | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✓ | ✓ |
| 執行去識別化任務 | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✓ | ✓ |
| 下載去識別化結果 | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✓ | ✓ |
| **規則管理（僅 Admin）** |
| 查看規則 | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✓ | ✓ |
| 建立規則 | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✓ | ✓ |
| 修改規則 | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✓ | ✓ |
| 刪除規則 | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✓ | ✓ |
| 匯入/匯出規則 | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✓ | ✓ |
| **清洗審核管理（Cleaner App）** |
| 查看清洗任務審核資訊 | ✗ | ✗ | ✗ | ✗ | ✓ | ✓ | ✓ | ✓ |
| 檢視清洗後檔案內容 | ✗ | ✗ | ✗ | ✗ | ✓ | ✓ | ✓ | ✓ |
| 編輯清洗後內容 | ✗ | ✗ | ✗ | ✗ | ✓ | ✗ | ✓ | ✓ |
| 標記標籤 | ✗ | ✗ | ✗ | ✗ | ✓ | ✓ | ✓ | ✓ |
| 批准/駁回任務 | ✗ | ✗ | ✗ | ✗ | ✗ | ✓ | ✓ | ✓ |
| 送入 RAG 知識庫 | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✓ | ✓ |
| 查看資料分析儀表板 | ✗ | ✗ | ✗ | ✗ | ✓ | ✓ | ✓ | ✓ |
| 瀏覽來源資料 | ✗ | ✗ | ✗ | ✗ | ✓ | ✓ | ✓ | ✓ |
| 瀏覽知識庫文件 | ✗ | ✗ | ✗ | ✗ | ✓ | ✓ | ✓ | ✓ |
| 按來源刪除知識庫文件 | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✓ |
| **稽核日誌** |
| 查看自己日誌 | ✗ | ✗ | ✗ | ✓ | ✓ | ✓ | ✓ | ✓ |
| 查看所有日誌 | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✓ | ✓ |
| **使用者管理** 🔄 |
| 查看使用者列表 | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✓ | ✓ |
| 建立使用者 | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✓ | ✓ |
| 停用/啟用帳號 | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✓ | ✓ |
| 指派角色 | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✓ | ✓ |
| **系統設定** 🔄 |
| 提示詞管理 | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✓ | ✓ |
| 系統參數設定 | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✓ | ✓ |
| **會員管理** 🔮 |
| 查看會員列表 | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✓ |
| 建立/管理會員 | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✓ |
| 設定會員配額 | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✓ |

---

## 5. 功能需求 (FR)

### 5.1 功能需求總覽

| FR 編號 | 功能名稱 | 狀態 | 優先級 | 階段 |
|---------|----------|------|--------|------|
| FR-01 | 使用者認證與授權 | 🔄 | P1 | Phase 2 |
| FR-02 | RAG 查詢服務 | ✅ | P1 | Phase 1 |
| FR-03 | 提示詞管理 | 🔄 | P1 | Phase 2 |
| FR-04 | 資料去識別化功能 | ✅ | P1 | Phase 1 |
| FR-05 | 檔案上傳與下載 | ✅ | P1 | Phase 1 |
| FR-06 | 即時通知 | ✅ | P2 | Phase 1 |
| FR-07 | 稽核日誌 | ✅ | P2 | Phase 1 |
| FR-08 | 去識別化規則管理 | ✅ | P1 | Phase 1 |
| FR-09 | 任務管理 | ✅ | P1 | Phase 1 |
| FR-10 | 進階 RBAC 權限管理 | 🔮 | P1 | Phase 3 |
| FR-11 | 多會員架構 | 🔮 | P1 | Phase 3 |
| FR-12 | 多知識庫管理 | 🔮 | P2 | Phase 3 |
| FR-13 | 智慧分析儀表板 | 🔮 | P2 | Phase 3 |
| FR-14 | 第三方整合 | 🔮 | P3 | Phase 3 |
| FR-15 | 運維監控與告警 | 🔮 | P2 | Phase 3 |
| FR-16 | 三層式回應機制 | 🔄 | P1 | Phase 2 |
| FR-17 | 在地化法規知識庫 | ✅/🔄 | P1 | Phase 1-2 |
| FR-18 | 清洗審核管理 (Cleaner App) | ✅ | P1 | Phase 1 |
| FR-19 | 資料分析儀表板 (Analytics Dashboard) | ✅ | P2 | Phase 1 |
| FR-20 | 來源資料瀏覽 | ✅ | P1 | Phase 1 |
| FR-21 | 知識庫文件瀏覽 | ✅ | P1 | Phase 1 |

---

### 5.2 FR-01：使用者認證與授權 ✅

#### 5.2.1 功能描述

提供使用者身分驗證與權限控制機制。

#### 5.2.2 使用者故事

| US 編號 | 使用者故事 | 驗收標準 |
|---------|------------|----------|
| US-01-01 | 身為使用者，我希望能夠使用帳號密碼登入系統 | 1. 支援 Email + 密碼登入<br>2. 登入失敗顯示錯誤訊息<br>3. 密碼錯誤超過 5 次鎖定 15 分鐘<br>4. 每次登入建立獨立工作階段，最多 10 個有效工作階段<br>5. Access Token 60 分鐘、Refresh Token 7 天且單次輪替<br>6. Token 以 `tokenUse` 區分用途，Refresh Token 不可作為 Access Token |
| US-01-02 | 身為系統管理員，我希望能夠管理使用者帳號 | 1. 可建立新使用者<br>2. 可停用/啟用帳號<br>3. 可指派角色 |
| US-01-03 | 身為使用者，我希望能夠安全登出系統 | 1. 一般登出只撤銷目前工作階段<br>2. 可撤銷帳號全部工作階段<br>3. Refresh 無效時在原頁重新登入，不硬重載<br>4. 暫時性連線錯誤保留頁面與輸入<br>5. 重新登入只能使用原帳號，稍後處理後不因背景 401 自動重開 |

#### 5.2.3 API 規格

**POST /api/auth/login - 使用者登入**

```yaml
Request:
  Content-Type: application/json
  Body:
    email: string (required)
    password: string (required)
    clientType: admin | cleaner | chatbot | api (optional, default: api)

Response 200:
  {
    "success": true,
    "data": {
      "accessToken": "eyJhbGciOiJIUzI1NiIs...",
      "refreshToken": "eyJhbGciOiJIUzI1NiIs...",
      "sessionId": "uuid",
      "clientType": "chatbot",
      "user": {
        "id": "uuid",
        "email": "user@example.com",
        "name": "使用者名稱",
        "role": "user"
      }
    }
  }

Response 401:
  {
    "success": false,
    "error": {
      "code": "INVALID_CREDENTIALS",
      "message": "帳號或密碼錯誤"
    }
  }

Response 423:
  {
    "success": false,
    "error": {
      "code": "ACCOUNT_LOCKED",
      "message": "帳號已鎖定，請於 15 分鐘後再試"
    }
  }
```

**POST /api/auth/logout - 使用者登出**

```yaml
Request:
  Headers:
    Authorization: Bearer <accessToken>

Response 200:
  {
    "success": true,
    "message": "登出成功"
  }
```

此端點只撤銷 Bearer Token 內 `sid` 對應的目前工作階段。`POST /api/auth/logout-all` 會撤銷該帳號全部工作階段；`POST /api/auth/sso/exchange` 以目前 Admin Access Token 建立獨立的 Cleaner 工作階段，不傳遞 Admin Refresh Token。

**POST /api/auth/refresh - 更新 Token**

```yaml
Request:
  Content-Type: application/json
  Body:
    refreshToken: string (required)

Response 200:
  {
    "success": true,
    "data": {
      "accessToken": "eyJhbGciOiJIUzI1NiIs...",
      "refreshToken": "eyJhbGciOiJIUzI1NiIs...",
      "sessionId": "uuid",
      "clientType": "chatbot"
    }
  }
```

#### 5.2.4 資料模型

```sql
CREATE TABLE users (
    id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email          VARCHAR(255) NOT NULL UNIQUE,
    name           VARCHAR(100),
    password       VARCHAR(255) NOT NULL,
    role           VARCHAR(50) DEFAULT 'user',
    is_active      BOOLEAN DEFAULT TRUE,
    locked_until   TIMESTAMP,
    login_attempts INTEGER DEFAULT 0,
    last_login     TIMESTAMP,
    created_at     TIMESTAMP DEFAULT NOW(),
    updated_at     TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_role ON users(role);

CREATE TABLE auth_sessions (
    id                 TEXT PRIMARY KEY,
    user_id            UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    client_type        TEXT NOT NULL CHECK (client_type IN ('admin', 'cleaner', 'chatbot', 'api')),
    refresh_token_hash TEXT NOT NULL,
    current_jti        TEXT NOT NULL UNIQUE,
    expires_at         TIMESTAMP NOT NULL,
    revoked_at         TIMESTAMP,
    last_used_at       TIMESTAMP NOT NULL DEFAULT NOW(),
    created_at         TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at         TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX auth_sessions_user_id_revoked_at_idx ON auth_sessions(user_id, revoked_at);
CREATE INDEX auth_sessions_expires_at_idx ON auth_sessions(expires_at);
```

---

### 5.3 FR-02：RAG 查詢服務 ✅

#### 5.3.1 功能描述

提供基於 RAG 技術的智慧問答服務。

#### 5.3.2 使用者故事

| US 編號 | 使用者故事 | 驗收標準 |
|---------|------------|----------|
| US-02-01 | 身為使用者，我希望能夠用自然語言詢問資安問題 | 1. 支援中英文提問<br>2. 回應時間 < 10 秒<br>3. 回應包含引用來源 |
| US-02-02 | 身為使用者，我希望看到回答的來源出處 | 1. 顯示引用文件名稱<br>2. 顯示相關段落 |
| US-02-03 | 身為使用者，我希望查看歷史對話記錄 | 1. 對話以時間序列展示<br>2. 可搜尋歷史對話 |
| US-02-04 | 身為一般方案使用者，我希望選擇「一般」取得實務技術指引 | 1. 可選擇新手／一般回應層級<br>2. 回應包含操作步驟<br>3. 用語適合技術人員 |
| US-02-05 | 身為使用者，我希望預設以適合我角色的回應層級回答 | 1. 系統依角色自動選擇預設層級<br>2. 僅顯示角色可用層級 |
| US-02-06 | 身為使用者，我希望簡短縮寫或延續語句仍能在資安脈絡中得到合理處理 | 1. 非資安／不明確初判仍先檢索知識庫<br>2. 有達門檻證據時依知識庫回答<br>3. 縮寫／機構片段無知識庫證據時可執行限定資安網搜並以低信心推測語句回答<br>4. 無證據片段可保留一輪，下一則資安限定詞合併查詢<br>5. 建議問題必須屬於資安範圍且可回覆 |
| US-02-07 | 身為使用者，我希望在不滿意最新回答時重新產出 | 1. 只在最後提示詞對應、已持久化且分類為資安／混合的最新完成回覆顯示 icon<br>2. 保留原問題與原回覆，在底部新增相同問題與新回覆<br>3. 使用目前選擇的回應層級；每次新操作計入一次額度，同一操作的斷線重試不得重複計費<br>4. 串流中、失敗、未持久化、分類未知、非資安固定回覆與非最新回覆不提供重新產出 |

#### 5.3.3 檢索策略

| 參數 | 預設值 | 說明 |
|------|--------|------|
| Top-K | 5 | 檢索前 K 個最相關片段 |
| 相似度閾值 | 0.7 | 低於此分數的結果被過濾 |
| 混合檢索權重 | 向量 0.7 / 關鍵字 0.3 | Hybrid Search 權重 |
| 最大上下文長度 | 4000 tokens | 避免超出 LLM 上下文限制 |

#### 5.3.4 API 規格

**POST /api/chat - 發送對話訊息**

```yaml
Request:
  Headers:
    Authorization: Bearer <accessToken>
  Content-Type: application/json
  Body:
    message: string (required)
    conversationId: string (optional)
    messageAttemptId: UUID (optional for legacy clients; Chatbot always sends and reuses it for the same retry)
    regenerateFromMessageId: UUID (optional; requires conversationId and must be its latest assistant message)
    regenerationAttemptId: UUID (required when regenerateFromMessageId is provided; reused by retries)
    options:
      stream: boolean (default: true)
      temperature: number (default: 0.7)

Response 200 (非串流):
  {
    "success": true,
    "data": {
      "conversationId": "conv-uuid",
      "messageId": "msg-uuid",
      "role": "assistant",
      "content": "根據資安法規...",
      "sources": [
        {
          "documentId": "doc-uuid",
          "filename": "個資法條文.pdf",
          "chunk": "第六條規定...",
          "score": 0.92
        }
      ],
      "confidenceLevel": "high",
      "createdAt": "2026-02-05T10:30:00.000Z"
    }
  }

Response 200 (串流 - SSE):
  Content-Type: text/event-stream

  event: message
  data: {"content": "根據", "done": false}

  event: message
  data: {"content": "資安法規", "done": false}

  event: sources
  data: {"sources": [...]}

  event: done
  data: {"messageId": "msg-uuid", "conversationId": "conv-uuid", "confidenceLevel": "high"}
```

> **信心度欄位**：回應中包含 `confidenceLevel: 'high' | 'medium' | 'low'`，非串流模式亦於 `data` 中回傳。

#### 議題範圍分類

每次聊天在 Query 改寫後執行低輸出量分類。分類結果只作為 `initialTopicScope` 初判，不得直接把 `non_cybersecurity` 或 `unclear` 當成停止條件；所有初判都先向 RAG `/retrieve` 查詢：

| 初判／資料狀態 | 後端處理 | 最終 `topicScope`／前端行為 |
|---------------|----------|-----------------------------|
| `cybersecurity` | 執行既有 RAG、必要的網路補充與 LLM 回答 | `cybersecurity`；不顯示提醒 |
| `mixed` | 抽取資安問句後用於 RAG、網路搜尋、主回答與 `{query}` | `mixed`；顯示混合議題提醒 |
| `non_cybersecurity`／`unclear` 且有達門檻知識庫證據 | 依知識庫證據生成回答；top 1 fallback 不算有效證據 | `cybersecurity`；不顯示非資安阻擋 |
| `non_cybersecurity`／`unclear`、無知識庫證據，但本輪屬縮寫／機構片段，或上一輪待銜接片段後接明確短資安限定詞 | 在查詢補上資安、隱私、法遵與資訊風險詞後執行限定網搜 | 有安全且非空來源時為 `cybersecurity`、`low` 信心；API 固定在回答前加上「若您指的是…」推測性前綴 |
| `unclear`／候選片段且所有來源皆無證據 | 要求補充情境，候選片段保留一輪 | `unclear`；顯示補充提示 |
| 明確一般內容且無證據 | 回傳固定服務範圍引導，不回答一般內容 | `non_cybersecurity`；顯示非資安提醒 |
| RAG 服務不可用 | 明確告知知識庫暫時無法確認，禁止改用網搜推測 | `unclear`；顯示暫時無法確認提示 |

知識庫證據門檻沿用 `RAG_THRESHOLDS.hybrid_filter_min`／`cosine_filter_min`；一般檢索為避免空回答而保留的 top 1 fallback，不得將非資安／不明確初判恢復為資安回答。RAG 回應的每筆來源、分數、metadata 與 temporal context 都必須通過結構及有限數值驗證，否則視為 `rag_unavailable`。網搜來源 URL 在搜尋結果列入引用與實際抓取前都必須通過 HTTP(S)、DNS 與公開 IP 檢查；每次重新導向都重新驗證，HTTP agent 固定使用已驗證 IP，並正確支援 Node `lookup({ all: true })` 契約。

知識庫、網頁、使用者問題與顯示名稱不得展開到 system message。system message 只包含固定政策與資料位置標記；執行期內容以獨立 user data message 的 JSON 傳入，並標示為不可信資料。正式 Chat 與後台「測試提示詞」共用 `renderTrustedSystemPrompt()`／`buildUntrustedRuntimeData()`，後台分別顯示 `rendered` 與 `runtimeData`，兩者必須等於模型正式收到的 system 與 user data message。

`topicScope` 代表最終處理結果，寫入 `messages.metadata`，並出現在非串流 `answer.topicScope` 與 SSE `done.topicScope`。內部 metadata 另存 `initialTopicScope`、`scopeResolution`、必要時的 `scopedQuestion`、`pendingTopicBridge`、`topicBridgeSourceMessageId`、`topicBridgeConsumedAt`、`messageAttemptTopicBridgeFragment` 與 `messageAttemptTopicBridgeSourceMessageId`，用來追蹤判定來源和單輪銜接；全部沿用既有 JSON metadata，不需資料庫 migration，舊訊息沒有欄位時維持原畫面。

單輪銜接僅適用同一對話最新助理訊息保存的短縮寫／機構片段。下一則訊息必須完全符合短資安限定詞集合，例如「資安服務」「資訊安全措施」「隱私要求」；有主詞、動詞或完整敘述的「公司需要做資安風險評估」不得合併。完整新問句、含問號／句號的語句與重新產出也不合併。`MessageRepository` 在 PostgreSQL transaction 內鎖定 conversation row 並寫入 `topicBridgeConsumedAt`，確保併發請求只能消耗一次；帶 `messageAttemptId` 時，消耗必須發生在建立本輪 user claim 前，並將取得的 bridge 保存於 claim，讓 failed／逾時重試維持同一語意。不符合合併條件的下一則訊息仍會使片段失效。`test:chat-topic-bridge-integration` 必須以真實 PostgreSQL 驗證獨立 consume 的併發單次消耗，以及一般訊息 claim 的消耗、保存與重試契約。

分類器同時接收 `originalQuestion` 與 `rewrittenQuestion`：原始問題保留完整需求，改寫查詢補足多輪脈絡。原文寫入對話紀錄，使用者訊息 metadata 保存最終 `historyTopicScope` 與必要的 `scopedQuestion`。後續載入模型歷史時以 `scopedQuestion` 取代原文，並排除 `non_cybersecurity`、`unclear` 與 `rag_unavailable` 的固定回覆輪次；UI 仍顯示原始文字。

建議問題由 LLM 生成後逐題再經議題分類驗證，只保留資安範圍題目；生成或驗證失敗、逾時、數量不足時，以固定的資安風險盤點與控制有效性問題補足。按下建議問題按鈕會完整送出該文字，仍經相同 RAG 優先解析流程，因此提示本身必須可回覆，不得成為固定「非資安議題」回應。模型分類與解析行為由 `docs/02-testing/topic-classifier-eval.md` 的版本化案例及決定性單元測試驗證。

#### 一般訊息 attempt 冪等與復原

Chatbot 每次一般訊息送出都產生 `messageAttemptId`，同一次失敗重試沿用相同 UUID；該欄不得與 `regenerateFromMessageId`／`regenerationAttemptId` 併用。API 在 PostgreSQL transaction 內鎖定使用者與 conversation 資料列，依「使用者 + attempt」尋找既有 claim：首次請求原子建立必要的 conversation、消耗並保存 topic bridge、建立待完成 user 訊息與保留額度；相同 attempt 不得建立第二組對話或再次扣額。這項流程以既有 JSON metadata 保存 `messageAttemptId`、`messageAttemptStatus`、`messageAttemptClaimedAt`、`messageAttemptLeaseId` 與 bridge 識別，不新增資料表或 migration。

相同 attempt 的 5 分鐘處理租約有效時回 `409`；`failed` 或逾時 claim 可換發 lease 並沿用原 user 訊息。assistant 落盤前在 conversation row lock 內核對 lease，成功後將 user claim 標為 `completed`；舊 worker 不得寫入或釋放新 lease。已完成 attempt 直接回放相同 assistant、來源與 message ID。SSE 缺少 `done` 時，前端以 metadata 的 attempt ID 配對 user／assistant，不以相同問題文字猜測；狀態查詢失敗時採 fail-closed。

#### 最新回覆重新產出

Chatbot 只在最後一則使用者提示詞所對應、具有後端 UUID，且 `topicScope` 明確為 `cybersecurity` 或 `mixed` 的最新完成助理回覆顯示 `RefreshCw` icon；既有讚／不讚維持原位，icon 不顯示文字，滑鼠停留時以 `title` 顯示「重新產出」，並提供 `aria-label`。串流中、失敗、未持久化、分類未知、`non_cybersecurity`、`unclear`、沒有對應提示詞、唯讀狀態與非最新回覆不顯示；新手額度為 0 時顯示停用狀態及原因。

點擊後以目前畫面選擇的 `mode`、相同 `question`、原 `conversationId`、`regenerateFromMessageId` 與前端產生的 `regenerationAttemptId` 呼叫既有發送端點。前端在對話底部追加相同使用者問題與新助理串流，不刪除或覆蓋原問答；同步 in-flight 鎖確保同一頁面同時只有一條串流。新回答有獨立的來源、信心度、建議問題與回饋狀態，完成後重新產出 icon 移至新回答；只有最新失敗氣泡可重試，且重試沿用相同 attempt ID。串流進行中收到重試操作時，必須先拒絕，不可先刪除 UI 訊息。

API 以 PostgreSQL conversation row lock 串行化訊息寫入，並在同一資料庫交易內確認 `regenerateFromMessageId` 是目前最新 `assistant`、來源 `topicScope` 為 `cybersecurity`／`mixed`，且其前一則 user 訊息等於請求 `question`，再原子保留一次額度並建立待完成 user 訊息；任一步驟失敗時額度與 claim 一併回滾，任一條件不符即回 `400`。所有訊息建立時間在該 conversation 內至少比前一則晚 1ms，避免相同時間戳造成最新訊息判定不穩定。不同 attempt 的並發請求只有第一個可成立；相同 attempt 的 5 分鐘 lease 仍有效時回 `409`，禁止同時啟動第二次 LLM 生成。

待完成 user 訊息與完成 assistant 訊息都在既有 JSON metadata 保存 `regeneratedFromMessageId` 與 `regenerationAttemptId`；待完成訊息另保存 `regenerationStatus`、`regenerationClaimedAt` 與 `regenerationLeaseId`。前置處理補寫主題 metadata 時使用 lease compare-and-set；SSE 中斷或伺服器錯誤也只可用當前 lease 把 claim 標為 `failed`，避免逾時接手後舊 worker 把舊 lease 寫回或釋放新 worker。程序未正常釋放時，相同 attempt 可在 lease 逾時後接手；每次接手都換發 lease ID，assistant 寫入前須在 conversation lock 內驗證 lease，舊 worker 不得寫入重複回答。claim／assistant 均依 conversation、來源與 attempt 精確定位，不依賴仍為最後一則訊息，因此其他分頁插入訊息後仍可安全釋放、完成與回放。這些重試不重複扣額度；新的重新產出操作才計為新用量。不新增資料表或 migration。

SSE 在模型尚未完成時斷線，可中止該次生成並釋放目前 claim；模型完整生成後必須先將 assistant 回答與基礎 metadata 落盤，再生成及更新建議問題。客戶端缺少 `done` 時，重試前先依 attempt ID 讀取目前對話：已有對應完成回答即復原，不送出第二次請求；無法讀取時採 fail-closed 並保留重試狀態，不得把未確認的請求重送。已完成 claim 不得標為失敗。

#### 信心度判定邏輯

| 等級 | 標籤 | 條件 |
|------|------|------|
| high | 知識庫來源 | `best_score >= 0.7` 且未觸發 web search |
| medium | 混合來源 | `best_score 0.3~0.7` 或觸發 web search |
| low | 網路補充 | `best_score < 0.3` 或僅依賴 web search |

**POST /api/chat/messages/{messageId}/feedback — 回覆品質回饋**

```yaml
Request:
  Headers:
    Authorization: Bearer <accessToken>
  Path:
    messageId: string (required)
  Body:
    feedback: 'positive' | 'negative' (required)

Response 200:
  {
    "success": true,
    "data": {
      "messageId": "msg-uuid",
      "feedback": "positive"
    }
  }
```

**GET /api/chat/conversations - 取得對話列表**

```yaml
Request:
  Headers:
    Authorization: Bearer <accessToken>
  Query:
    limit: number (default: 20)
    offset: number (default: 0)

Response 200:
  {
    "success": true,
    "data": {
      "conversations": [
        {
          "id": "conv-uuid",
          "title": "關於個資法的問題",
          "lastMessage": "個資法第六條...",
          "messageCount": 5,
          "createdAt": "2026-02-05T09:00:00.000Z",
          "updatedAt": "2026-02-05T10:30:00.000Z"
        }
      ],
      "total": 15
    }
  }
```

管理後台使用 `GET /api/chat/admin/conversations` 取得跨使用者對話列表。其 `messageCount` 定義為該對話所有 `user` 與 `assistant` 訊息總和；列表中的 `messages` 只保留最新一則訊息作為摘要，兩者不可互相替代。正常完成一回合問答通常增加 2，生成失敗或串流中斷時可能只增加 1。

管理者使用 `GET /api/chat/admin/conversations/{conversationId}` 取得完整對話詳情，回應必須包含建立該對話的使用者摘要（`id`、`email`、`name`）與完整訊息。一般使用者詳情端點不額外回傳使用者摘要。

**GET /api/chat/conversations/{conversationId} - 取得對話詳情**

```yaml
Request:
  Headers:
    Authorization: Bearer <accessToken>
  Path:
    conversationId: string (required)

Response 200:
  {
    "success": true,
    "data": {
      "id": "conv-uuid",
      "title": "關於個資法的問題",
      "messages": [
        {
          "id": "msg-uuid-1",
          "role": "user",
          "content": "什麼是個資法？",
          "createdAt": "2026-02-05T09:00:00.000Z"
        },
        {
          "id": "msg-uuid-2",
          "role": "assistant",
          "content": "個資法全名為「個人資料保護法」...",
          "sources": [...],
          "createdAt": "2026-02-05T09:00:05.000Z"
        }
      ],
      "createdAt": "2026-02-05T09:00:00.000Z",
      "updatedAt": "2026-02-05T10:30:00.000Z"
    }
  }
```

#### 5.3.5 資料模型

```sql
CREATE TABLE conversations (
    id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id    UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    title      VARCHAR(255),
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_conversations_user_id ON conversations(user_id);
CREATE INDEX idx_conversations_updated_at ON conversations(updated_at);

CREATE TABLE messages (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    conversation_id UUID NOT NULL REFERENCES conversations(id) ON DELETE CASCADE,
    role            VARCHAR(20) NOT NULL,
    content         TEXT NOT NULL,
    sources         JSONB,
    metadata        JSONB,
    created_at      TIMESTAMP DEFAULT NOW()
);

CREATE INDEX messages_conversation_id_created_at_idx
    ON messages(conversation_id, created_at);
```

---

### 5.4 FR-03：提示詞管理 🔄

#### 5.4.1 功能描述

提供 beginner / standard / expert 三個回應層級的提示詞管理。正式聊天只依 mode 查找提示詞，不再使用 role × mode 組合。

#### 5.4.2 使用者故事

| US 編號 | 使用者故事 | 驗收標準 |
|---------|------------|----------|
| US-03-01 | 身為系統管理員，我希望管理三個回應層級提示詞 | 1. 管理介面固定三筆<br>2. 可編輯與測試<br>3. 不顯示新增、刪除操作 |
| US-03-02 | 身為顧問方案使用者，我希望獲得更專業的回應 | 1. expert 使用專業提示詞<br>2. 回應包含更多技術細節 |
| US-03-03 | 身為系統管理員，我希望能夠預覽提示詞效果 | 1. 提供測試對話功能<br>2. 可比較不同提示詞效果 |
| US-03-04 | 身為系統管理員，我希望回應層級直接對應提示詞 | 1. mode 唯一決定提示詞<br>2. 每個 mode 只有一筆啟用版本 |

#### 5.4.3 提示詞變數

| 變數名稱 | 說明 | 範例值 |
|----------|------|--------|
| `{user_name}` | 使用者名稱 | 王小明 |
| `{user_role}` | 使用者角色 | 資安顧問 |
| `{current_date}` | 當前日期 | 2026-02-05 |
| `{context}` | RAG 檢索結果 | (動態注入) |
| `{query}` | 使用者提問 | (動態注入) |
| `{response_mode}` | 回應模式 | beginner / standard / expert |

#### 5.4.4 API 規格

**GET /api/prompts - 取得提示詞範本列表**

```yaml
Request:
  Headers:
    Authorization: Bearer <accessToken>
  Query:
    role: string (optional)

Response 200:
  {
    "success": true,
    "data": {
      "prompts": [
        {
          "id": "prompt-uuid",
          "name": "一般使用者",
          "role": "user",
          "content": "你是 ODA Cyber Konsult...",
          "isActive": true,
          "createdAt": "2026-01-15T10:00:00.000Z",
          "updatedAt": "2026-02-01T14:30:00.000Z"
        }
      ],
      "total": 3
    }
  }
```

**POST /api/prompts - 建立提示詞範本**

```yaml
Request:
  Headers:
    Authorization: Bearer <accessToken>
  Content-Type: application/json
  Body:
    name: string (required)
    role: string (required)
    content: string (required)
    isActive: boolean (default: false)

Response 201:
  {
    "success": true,
    "data": {
      "id": "prompt-uuid",
      "name": "新提示詞範本",
      "role": "user",
      "content": "...",
      "isActive": false,
      "createdAt": "2026-02-05T10:00:00.000Z",
      "updatedAt": "2026-02-05T10:00:00.000Z"
    }
  }
```

**POST /api/prompts/{promptId}/test - 測試提示詞**

```yaml
Request:
  Headers:
    Authorization: Bearer <accessToken>
  Content-Type: application/json
  Body:
    variables:
      user_name: string (optional)
      query: string (optional)
      context: string (optional)

Response 200:
  {
    "success": true,
    "data": {
      "original": "原始提示詞範本...",
      "rendered": "模型正式收到的靜態 system message...",
      "runtimeData": "承載 query、context 與 user_name 的獨立 user JSON data message...",
      "variables": { "query": "如何防範釣魚攻擊？" }
    }
  }
```

`rendered` 不得包含執行期的知識庫、網頁、問題或顯示名稱；這些值只出現在 `runtimeData`。此輸出與正式 Chat 呼叫 LLM 的兩則訊息完全相同。

#### 5.4.5 資料模型

```sql
CREATE TABLE prompt_templates (
    id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name       VARCHAR(100) NOT NULL,
    role       VARCHAR(50), -- legacy only；執行期不參與查找
    mode       VARCHAR(50) DEFAULT 'standard',
    content    TEXT NOT NULL,
    variables  JSONB,
    is_active  BOOLEAN DEFAULT FALSE,
    created_by UUID REFERENCES users(id),
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_prompt_templates_mode_active ON prompt_templates(mode, is_active);
CREATE UNIQUE INDEX prompt_templates_one_active_per_mode
    ON prompt_templates(mode) WHERE is_active = TRUE;
```

#### 5.4.6 回應層級提示詞

| mode | 介面名稱 | 用途 |
|------|----------|------|
| beginner | 新手 | 白話說明與下一步建議 |
| standard | 一般 | IT/MIS 實務操作與注意事項 |
| expert | 顧問 | 法規、標準、風險與專業建議 |

---

### 5.5 FR-04：資料去識別化功能 ✅

#### 5.5.1 功能描述

提供文件 PII 偵測與去識別化處理功能。本功能僅供系統管理員使用，用於處理即將匯入 RAG 知識庫的訓練資料。

#### 5.5.2 使用者故事

| US 編號 | 使用者故事 | 驗收標準 |
|---------|------------|----------|
| US-04-01 | 身為系統管理員，我希望能夠自動偵測文件中的敏感資料 | 1. 支援 20 種實體類型<br>2. 偵測準確率 > 95%<br>3. 顯示偵測信心分數 |
| US-04-02 | 身為系統管理員，我希望能夠選擇不同的去識別化策略 | 1. 支援 6 種去識別化策略<br>2. 可自訂策略參數<br>3. 可預覽去識別化效果 |
| US-04-03 | 身為系統管理員，我希望系統自動套用內建的去識別化規則 | 1. 系統內建 6 種固定規則（`get_default_rules()`）<br>2. 清洗任務自動套用固定規則<br>3. 規則變更需透過程式碼更新 |
| US-04-04 | 身為系統管理員，我希望能夠批次處理多個檔案 | 1. 支援多檔上傳<br>2. 並行處理<br>3. 即時進度回報 |

#### 5.5.3 清洗管線流程

```
Upload → Parse → Detect PII → Anonymize → Output

1. Upload    - 接收檔案 (PDF/DOCX/XLSX/CSV/TXT)
2. Parse     - 依類型提取純文字
3. Detect    - Presidio + spaCy 偵測 20 種實體
4. Anonymize - 依規則執行 6 種去識別化策略
5. Output    - 產出清洗檔案與 JSON 報告
```

#### 5.5.4 支援的實體類型（20 種）

| 分類 | 實體類型 | 說明 | 範例 |
|------|----------|------|------|
| **標準 PII** | PERSON | 人名 | 王小明、John Smith |
| | EMAIL_ADDRESS | 電子郵件 | user@example.com |
| | PHONE_NUMBER | 電話號碼 | +886-2-1234-5678 |
| | LOCATION | 地址/地點 | 台北市信義區 |
| | DATE_TIME | 日期時間 | 2025-01-15、民國114年 |
| | CREDIT_CARD | 信用卡號 | 4111-1111-1111-1111 |
| | IBAN_CODE | 國際銀行帳號 | DE89370400440532013000 |
| | IP_ADDRESS | IP 位址 | 192.168.1.1 |
| | URL | 網址 | https://example.com |
| | ORGANIZATION | 組織名稱 | 台灣積體電路製造股份有限公司 |
| **台灣專屬** | TW_ID | 身分證字號 | A123456789 |
| | TW_PHONE | 台灣手機 | 0912-345-678 |
| | TW_UNIFIED_BUSINESS_NO | 統一編號 | 12345678 |
| | TW_ADDRESS | 台灣地址 | 台北市信義區信義路五段7號 |
| **企業機密** | API_KEY | API 金鑰 | sk-abc123def456... |
| | ACCESS_TOKEN | 存取權杖 | Bearer eyJhbGciOiJ... |
| | PRIVATE_KEY | 私密金鑰 | -----BEGIN RSA PRIVATE KEY----- |
| | AWS_ACCESS_KEY | AWS 金鑰 | AKIAIOSFODNN7EXAMPLE |
| | AZURE_KEY | Azure 金鑰 | DefaultEndpointsProtocol=... |
| | GCP_KEY | GCP 金鑰 | AIzaSyA1234567890... |

#### 5.5.5 支援的去識別化策略（6 種）

| 策略 | 說明 | 輸入範例 | 輸出範例 |
|------|------|----------|----------|
| `mask` | 完全遮罩 | 王小明 | \*\*\* |
| `partial_mask` | 部分遮罩 | 0912345678 | 0912\*\*\*678 |
| `pseudonymize` | 假名替換 | 王小明 | 陳大華（一致性對映） |
| `generalize` | 泛化處理 | 台北市信義區 | 台北市 |
| `keep_labeled` | 保留標籤 | 王小明 | [PERSON: 王小明] |
| `encrypt` | 加密替換 | A123456789 | enc:a3f8b2c1...（可逆） |

#### 5.5.6 策略參數

```typescript
// mask 策略參數
interface MaskParams {
  maskChar?: string;  // 遮罩字元，預設 "*"
}

// partial_mask 策略參數
interface PartialMaskParams {
  keep_first?: number;   // 前方保留字數
  keep_last?: number;    // 後方保留字數
  maskChar?: string;      // 遮罩字元
}

// pseudonymize 策略參數
interface PseudonymizeParams {
  prefix?: string;           // 假名前綴
  maintainRelations?: boolean;  // 是否維持關聯性
}

// generalize 策略參數
interface GeneralizeParams {
  precision?: 'year' | 'decade' | 'city' | 'country';
}

// keep_labeled 策略參數
interface KeepLabeledParams {
  format?: 'bracket' | 'xml';
  includeValue?: boolean;
}

// encrypt 策略參數
interface EncryptParams {
  algorithm?: string;  // 預設 "aes-256-gcm"
}
```

#### 5.5.7 API 規格

**POST /api/v1/clean - 啟動清洗任務**

```yaml
Request:
  Content-Type: application/json
  Body:
    file_ids: string[] (required)
    task_name: string (required, trimmed length 1-100)
    profile_id: string (optional)
    rules: EntityRule[] (optional)
    session_id: string (optional)

Response 200:
  {
    "success": true,
    "data": {
      "task_id": "task-uuid",
      "status": "pending",
      "message": "清洗任務已建立，共 3 個檔案排入處理佇列",
      "file_count": 3,
      "rule_id": "profile-uuid"
    }
  }
```

**POST /api/v1/clean/preview - 預覽清洗結果**

```yaml
Request:
  Content-Type: application/json
  Body:
    file_id: string (required)
    rules: EntityRule[] (optional)
    sample_size: number (default: 1000)

Response 200:
  {
    "success": true,
    "data": {
      "original": "客戶王小明（身分證 A123456789）...",
      "anonymized": "客戶陳大華（身分證 **********）...",
      "entities": [
        {
          "entity_type": "PERSON",
          "text": "王小明",
          "start": 2,
          "end": 5,
          "score": 0.95
        }
      ],
      "stats": {
        "total_entities": 3,
        "by_type": {
          "PERSON": 1,
          "TW_ID": 1,
          "DATE_TIME": 1
        }
      }
    }
  }
```

**GET /api/v1/clean/{task_id}/result - 取得清洗結果**

```yaml
Request:
  Path:
    task_id: string (required)

Response 200:
  {
    "success": true,
    "data": {
      "task_id": "task-uuid",
      "status": "completed",
      "total_files": 2,
      "completed_files": 2,
      "total_entities_found": 47,
      "files": [
        {
          "file_id": "file-uuid-1",
          "filename": "Q1_財務報告.pdf",
          "status": "completed",
          "entities_found": 32,
          "entity_breakdown": {
            "PERSON": 12,
            "TW_ID": 5,
            "EMAIL_ADDRESS": 8,
            "PHONE_NUMBER": 7
          },
          "processing_time_seconds": 2.5,
          "error": null
        }
      ],
      "created_at": "2026-02-05T10:30:00.000Z",
      "completed_at": "2026-02-05T10:32:15.000Z"
    }
  }
```

#### 5.5.8 資料模型

```sql
CREATE TABLE files (
    id            VARCHAR(36) PRIMARY KEY,
    original_name VARCHAR(255) NOT NULL,
    file_type     VARCHAR(50) NOT NULL,
    size_bytes    BIGINT NOT NULL,
    storage_path  VARCHAR(500),
    user_id       UUID REFERENCES users(id) ON DELETE SET NULL,
    uploaded_at   TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_files_user_id ON files(user_id);

-- 注意：rule_profiles 與 anonymization_rules 表已於 migration 005 移除。
-- 去識別化規則改為固定模式，由 data-pipeline 的 get_default_rules() 統一定義。
-- 6 種固定策略：mask、partial_mask、pseudonymize、generalize、keep_labeled、encrypt

CREATE TABLE tasks (
    id                      VARCHAR(36) PRIMARY KEY,
    status                  VARCHAR(50) DEFAULT 'pending',
    progress                INTEGER DEFAULT 0,
    message                 TEXT,
    files_total             INTEGER DEFAULT 0,
    files_processed         INTEGER DEFAULT 0,
    total_entities_found    INTEGER DEFAULT 0,
    processing_time_seconds FLOAT,
    approval_status         VARCHAR(50) DEFAULT 'pending',  -- pending/review_requested/approved/rejected/ingested
    submitted_by            UUID REFERENCES users(id),
    submitted_at            TIMESTAMP,
    approved_by             UUID REFERENCES users(id),
    approved_at             TIMESTAMP,
    session_id              VARCHAR(36),
    user_id                 UUID REFERENCES users(id) ON DELETE SET NULL,
    created_at              TIMESTAMP DEFAULT NOW(),
    started_at              TIMESTAMP,
    completed_at            TIMESTAMP,
    error_message           TEXT
);

CREATE INDEX idx_tasks_status ON tasks(status);
CREATE INDEX idx_tasks_rule_id ON tasks(rule_id);
CREATE INDEX idx_tasks_user_id ON tasks(user_id);
CREATE INDEX idx_tasks_created_at ON tasks(created_at);

CREATE TABLE task_files (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    task_id                 VARCHAR(36) NOT NULL REFERENCES tasks(id) ON DELETE CASCADE,
    file_id                 VARCHAR(36) NOT NULL REFERENCES files(id) ON DELETE RESTRICT,
    status                  VARCHAR(50) DEFAULT 'pending',
    entities_found          INTEGER DEFAULT 0,
    output_path             VARCHAR(500),
    processing_time_seconds FLOAT,
    error_message           TEXT,
    created_at              TIMESTAMP DEFAULT NOW(),
    completed_at            TIMESTAMP
);

CREATE INDEX idx_task_files_task_id ON task_files(task_id);
CREATE INDEX idx_task_files_file_id ON task_files(file_id);
```

---

### 5.6 FR-05：檔案上傳與下載 ✅

#### 5.6.1 功能描述

提供檔案上傳與下載功能。

Admin 在送出前檢查支援格式、每個檔案 50 MB 上限及整批原始檔案合計 50 MB 上限；NestJS 接收 51 MB multipart request，以保留約 1 MB 封裝空間。上傳失敗時必須保留已選檔案，並在上傳區顯示持續可見的中文原因。Refresh 端點回 400／401 時在原頁顯示重新登入視窗；Refresh 成功但上傳重試仍回 401 時視為暫時性驗證異常，保留登入資料、已選檔案與頁面。ZIP 完全未匯入時停留在上傳步驟；部分匯入成功時進入分類步驟，並顯示「部分檔案未匯入」及中文原因。知識庫儀表板的來源、切塊、標籤與入庫活動標題及狀態訊息使用中文，不顯示 `Unauthorized`、`Top`、`Chunks`、`RAG` 或未知內部 action code。

#### 5.6.2 支援的檔案格式

| 格式 | 副檔名 | MIME Type | 大小限制 |
|------|--------|-----------|----------|
| PDF | .pdf | application/pdf | 50MB |
| Word | .docx, .doc | application/vnd.openxmlformats... | 50MB |
| Excel | .xlsx, .xls | application/vnd.openxmlformats... | 50MB |
| CSV | .csv | text/csv | 50MB |
| 純文字 | .txt | text/plain | 50MB |
| Markdown | .md | text/markdown | 50MB |
| JSON | .json | application/json | 50MB |
| HTML | .html, .htm | text/html | 50MB |
| ZIP | .zip | application/zip | 上傳檔本身 50MB；解壓後總量依環境設定，預設 200MB |

#### 5.6.3 API 規格

**POST /api/v1/upload - 上傳檔案**

```yaml
Request:
  Content-Type: multipart/form-data
  Body:
    files: File[] (required)

Response 200:
  {
    "success": true,
    "data": [
      {
        "id": "file-uuid",
        "filename": "a1b2c3d4_report.pdf",
        "original_name": "Q1_財務報告.pdf",
        "size": 1048576,
        "type": "application/pdf",
        "uploaded_at": "2026-02-05T10:30:00.000Z"
      }
    ],
    "message": "成功上傳 1 個檔案"
  }
```

**POST /api/v1/upload — ZIP 上傳（同一端點）**

ZIP 檔案透過既有 `POST /api/v1/upload` 端點上傳，伺服器依 MIME Type 自動識別並解壓處理。

```yaml
Request:
  Content-Type: multipart/form-data
  Body:
    files: File (required, application/zip)

ZIP 安全限制:
  - 壓縮比上限: ≤ 20:1（防止 Zip Bomb）
  - 單一 ZIP 內檔案數: ≤ 100
  - 解壓後總大小: 依 `max_zip_total_size_mb` 設定，預設 ≤ 200MB
  - 路徑穿越防護: 禁止包含 `../` 或絕對路徑的條目
  - 僅接受支援的內部檔案格式（PDF, DOCX, XLSX, CSV, TXT, MD, JSON, HTML）

Response 200:
  {
    "success": true,
    "data": [
      {
        "id": "file-uuid-1",
        "filename": "a1b2c3d4_report.pdf",
        "original_name": "report.pdf",
        "size": 1048576,
        "type": "application/pdf",
        "source_zip": "upload_batch.zip",
        "uploaded_at": "2026-03-11T10:30:00.000Z"
      }
    ],
    "message": "成功從 ZIP 解壓並上傳 5 個檔案"
  }

Response 400 (ZIP 安全違規):
  {
    "success": false,
    "error": {
      "code": "ZIP_SECURITY_VIOLATION",
      "message": "ZIP 壓縮比超過上限 (20:1)"
    }
  }

Response 400 (路徑穿越):
  {
    "success": false,
    "error": {
      "code": "ZIP_PATH_TRAVERSAL",
      "message": "ZIP 包含不合法的路徑條目"
    }
  }
```

> ZIP 上傳自動解壓後，各檔案獨立建立 `files` 記錄，並以 `source_zip` 欄位關聯原始 ZIP 檔名。不支援或無法處理的內部項目會略過並記錄於回應的 `errors` 欄位。`files` 為空時 Admin 不得前進；`files` 非空且 `errors` 有值時，Admin 顯示部分匯入警告後前進。

**GET /api/v1/upload/{file_id} - 取得檔案資訊**

```yaml
Response 200:
  {
    "success": true,
    "data": {
      "id": "file-uuid",
      "filename": "a1b2c3d4_report.pdf",
      "original_name": "Q1_財務報告.pdf",
      "size": 1048576,
      "type": "application/pdf",
      "uploaded_at": "2026-02-05T10:30:00.000Z"
    }
  }
```

**GET /api/v1/download/{task_id} - 下載清洗結果（ZIP）**

```yaml
Authorization: Bearer token (admin | data_cleaner)
Response 200:
  Content-Type: application/zip
  Content-Disposition: attachment; filename="task_{task_id}.zip"
```

Admin 不以一般 `<a href>` 導航下載；前端使用目前登入者的 Bearer Token 取得 Blob，再觸發檔案下載。「下載清洗結果」與「前往審核」是兩個獨立動作，前者不代表 Maker-Checker 審核完成。

**GET /api/v1/download/{task_id}/{file_id} - 下載單一檔案**

```yaml
Authorization: Bearer token (admin | data_cleaner)
Response 200:
  Content-Type: (依原始檔案類型)
  Content-Disposition: attachment; filename="cleaned_{filename}"
```

**GET /api/v1/download/{task_id}/report - 下載清洗報告**

```yaml
Authorization: Bearer token (admin | data_cleaner)
Response 200:
  Content-Type: application/json
  Content-Disposition: attachment; filename="report_{task_id}.json"
  Body:
    {
      "task_id": "task-uuid",
      "generated_at": "2026-02-05T12:00:00.000Z",
      "summary": {
        "total_files": 2,
        "total_entities": 47,
        "processing_time_ms": 13500
      },
      "files": [...]
    }
```

---

### 5.7 FR-06：即時通知 ✅

#### 5.7.1 功能描述

透過 WebSocket 提供即時通知功能。

#### 5.7.2 WebSocket 端點

| 端點 | 說明 |
|------|------|
| `/ws/tasks/{task_id}` | 訂閱特定任務 |
| `/ws/all` | 訂閱所有任務 |

#### 5.7.3 訊息格式

```typescript
interface TaskUpdateMessage {
  type: 'task_update';
  task_id: string;
  status?: 'pending' | 'processing' | 'completed' | 'failed' | 'cancelled';
  progress?: number;
  message?: string;
  current_file?: string;
  entities_found?: number;
  error?: string;
}

interface ConnectedMessage {
  type: 'connected';
  task_id?: string;
  message: string;
}

interface PingPongMessage {
  type: 'ping' | 'pong';
}
```

---

### 5.8 FR-07：稽核日誌 ✅

#### 5.8.1 功能描述

記錄系統中所有重要操作的稽核日誌。

#### 5.8.2 記錄的操作類型

| 操作類型 | 說明 | 記錄內容 |
|----------|------|----------|
| `user.login` | 使用者登入 | IP、User Agent |
| `user.logout` | 使用者登出 | - |
| `file.upload` | 檔案上傳 | 檔案名稱、大小、類型 |
| `file.delete` | 檔案刪除 | 檔案 ID |
| `task.create` | 建立清洗任務 | 檔案數量、規則 ID |
| `task.cancel` | 取消任務 | 任務 ID |
| `task.complete` | 任務完成 | 處理結果摘要 |
| `rule.create` | 建立規則 | 規則內容 |
| `rule.update` | 更新規則 | 變更內容 |
| `rule.delete` | 刪除規則 | 規則 ID |
| `download.file` | 下載檔案 | 任務 ID、檔案 ID |
| `download.report` | 下載報告 | 任務 ID |

#### 5.8.3 API 規格

**GET /api/audit-logs - 查詢稽核日誌**

```yaml
Request:
  Headers:
    Authorization: Bearer <accessToken>
  Query:
    start_date: string (optional)
    end_date: string (optional)
    action: string (optional)
    user_id: string (optional)
    resource_type: string (optional)
    limit: number (default: 50)
    offset: number (default: 0)

Response 200:
  {
    "success": true,
    "data": {
      "logs": [
        {
          "id": "log-uuid",
          "timestamp": "2026-02-05T10:30:00.000Z",
          "action": "task.create",
          "resource_type": "task",
          "resource_id": "task-uuid",
          "user_id": "user-uuid",
          "user_email": "consultant@example.com",
          "ip_address": "192.168.1.100",
          "details": {
            "file_count": 3,
            "rule_id": "rule-uuid"
          },
          "status": "success"
        }
      ],
      "total": 150
    }
  }
```

#### 5.8.4 資料模型

```sql
CREATE TABLE audit_logs (
    id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    timestamp     TIMESTAMP DEFAULT NOW(),
    action        VARCHAR(100) NOT NULL,
    resource_type VARCHAR(50),
    resource_id   VARCHAR(36),
    user_id       UUID REFERENCES users(id) ON DELETE SET NULL,
    session_id    VARCHAR(36),
    ip_address    VARCHAR(45),
    user_agent    TEXT,
    details       JSONB,
    status        VARCHAR(50) DEFAULT 'success',
    error_message TEXT
);

CREATE INDEX idx_audit_logs_timestamp ON audit_logs(timestamp);
CREATE INDEX idx_audit_logs_action ON audit_logs(action);
CREATE INDEX idx_audit_logs_user_id ON audit_logs(user_id);
CREATE INDEX idx_audit_logs_resource_type ON audit_logs(resource_type);
```

---

### 5.9 FR-08：去識別化規則管理 — 已簡化 ✅

> **⚠️ 已簡化**：原規劃為自訂規則 CRUD API，已簡化為固定規則模式。系統內建 6 種去識別化規則（由 `get_default_rules()` 統一定義），不再支援自訂規則 CRUD。原 API 端點（`/api/v1/rules/*`）及相關資料表（`rule_profiles`、`anonymization_rules`）已於 migration 005 移除。

#### 5.9.1 功能描述

提供去識別化規則設定檔的完整管理功能。

#### 5.9.2 API 規格

**GET /api/v1/rules - 列出規則設定檔**

```yaml
Response 200:
  {
    "success": true,
    "data": {
      "profiles": [
        {
          "id": "profile-uuid",
          "name": "標準台灣 PII 規則",
          "description": "適用於台灣地區的標準個資去識別化規則",
          "rules": [
            {
              "entityType": "PERSON",
              "strategy": "pseudonymize",
              "params": { "prefix": "User_" },
              "enabled": true
            }
          ],
          "isDefault": true,
          "usageCount": 156,
          "createdAt": "2026-01-15T10:00:00.000Z",
          "updatedAt": "2026-02-01T14:30:00.000Z"
        }
      ],
      "total": 5,
      "page": 1,
      "limit": 20
    }
  }
```

**POST /api/v1/rules - 建立規則設定檔**

```yaml
Request:
  Body:
    name: string (required)
    description: string (optional)
    rules: EntityRule[] (required)
    isDefault: boolean (default: false)

Response 201:
  {
    "success": true,
    "data": {
      "id": "new-profile-uuid",
      "name": "金融業嚴格規則",
      ...
    },
    "message": "規則設定檔建立成功"
  }
```

**GET /api/v1/rules/{profileId}/export - 匯出規則**

```yaml
Response 200:
  Content-Type: application/json
  Content-Disposition: attachment; filename="rule_profile_{name}.json"
  Body:
    {
      "version": "1.0",
      "exportedAt": "2026-02-05T12:00:00.000Z",
      "profile": {
        "name": "標準台灣 PII 規則",
        "description": "...",
        "rules": [...]
      }
    }
```

**POST /api/v1/rules/import - 匯入規則**

```yaml
Request:
  Body:
    version: string (required)
    profile: RuleProfileData (required)
    conflictStrategy: "skip" | "rename" | "overwrite" (default: "rename")

Response 200:
  {
    "success": true,
    "data": {
      "id": "imported-profile-uuid",
      "name": "標準台灣 PII 規則 (imported)",
      "rulesCount": 8
    },
    "message": "規則設定檔匯入成功"
  }
```

---

### 5.10 FR-09：任務管理 ✅

#### 5.10.1 功能描述

提供清洗任務的完整生命週期、命名、快速改名與一致的台灣時間顯示。`tasks.created_at` 等既有無時區欄位視為 UTC；FastAPI 輸出含 offset 的 ISO 8601 字串，Admin 使用 `Asia/Taipei` 格式化，不直接對字串硬加 8 小時。

#### 5.10.2 任務狀態機

```
    ┌─────────┐     啟動處理      ┌────────────┐
    │ pending │──────────────────▶│ processing │
    └────┬────┘                   └─────┬──────┘
         │                              │
         │ 使用者取消                    ├── 處理成功 ──▶ ┌───────────┐
         │                              │               │ completed │
         │                              │               └───────────┘
         ▼                              │
    ┌───────────┐                       ├── 處理失敗 ──▶ ┌────────┐
    │ cancelled │◀── 使用者取消 ────────┘               │ failed │
    └───────────┘                                       └────────┘
```

#### 5.10.3 API 規格

**GET /api/v1/tasks - 列出任務**

```yaml
Request:
  Query:
    status: string (optional)
    startDate: string (optional)
    endDate: string (optional)
    page: number (default: 1)
    limit: number (default: 20)
    sortBy: string (default: "createdAt")
    sortOrder: "asc" | "desc" (default: "desc")

Response 200:
  {
    "success": true,
    "data": {
      "tasks": [
        {
          "taskId": "task-uuid",
          "taskName": "114 年度資訊資產盤點表清洗",
          "status": "completed",
          "progress": 100,
          "filesTotal": 3,
          "filesProcessed": 3,
          "totalEntitiesFound": 47,
          "processingTimeSeconds": 12.5,
          "profileId": "profile-uuid",
          "profileName": "標準台灣 PII 規則",
          "createdAt": "2026-02-05T10:30:00.000Z",
          "startedAt": "2026-02-05T10:30:01.000Z",
          "completedAt": "2026-02-05T10:30:13.000Z"
        }
      ],
      "total": 150,
      "page": 1,
      "limit": 20
    }
  }
```

**PATCH /api/v1/tasks/{taskId}/name - 修改任務名稱（Admin）**

```yaml
Request:
  Body:
    task_name: string (required, trimmed length 1-100)
Response 200:
  task_id: UUID
  task_name: string
  created_at: ISO 8601 UTC offset
Errors:
  403: 非 Admin
  404: 任務不存在
  422: 名稱空白或超過 100 字
```

**DELETE /api/v1/tasks/{taskId} - 取消任務**

```yaml
Response 200:
  {
    "success": true,
    "data": {
      "taskId": "task-uuid",
      "status": "cancelled",
      "message": "任務已取消"
    }
  }

Response 400:
  {
    "success": false,
    "error": {
      "code": "TASK_NOT_CANCELLABLE",
      "message": "已完成或已取消的任務無法再次取消"
    }
  }
```

---

### 5.11 FR-10：進階 RBAC 權限管理 🔮

#### 5.11.1 功能描述

提供細粒度的權限控制機制。

#### 5.11.2 權限模型設計

```yaml
permissions:
  - resource: "file"
    actions: ["create", "read", "update", "delete"]
    scopes: ["own", "department", "tenant", "all"]

  - resource: "task"
    actions: ["create", "read", "cancel", "delete"]
    scopes: ["own", "department", "tenant", "all"]

  - resource: "rule"
    actions: ["create", "read", "update", "delete", "export", "import"]
    scopes: ["own", "tenant", "all"]

  - resource: "user"
    actions: ["create", "read", "update", "delete", "assign_role"]
    scopes: ["department", "tenant", "all"]

roles:
  - name: "user"
    permissions:
      - { resource: "file", action: "read", scope: "own" }
      - { resource: "task", action: "read", scope: "own" }

  - name: "consultant"
    inherits: ["user"]
    permissions:
      - { resource: "file", action: "*", scope: "own" }
      - { resource: "task", action: "*", scope: "own" }
      - { resource: "rule", action: "*", scope: "own" }

  - name: "admin"
    inherits: ["consultant"]
    permissions:
      - { resource: "*", action: "*", scope: "tenant" }

```

#### 5.11.3 資料模型

```sql
CREATE TABLE roles (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name         VARCHAR(100) NOT NULL UNIQUE,
    display_name VARCHAR(100) NOT NULL,
    description  TEXT,
    is_system    BOOLEAN DEFAULT FALSE,
    tenant_id    UUID REFERENCES tenants(id),
    created_at   TIMESTAMP DEFAULT NOW(),
    updated_at   TIMESTAMP DEFAULT NOW()
);

CREATE TABLE permissions (
    id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    resource   VARCHAR(100) NOT NULL,
    action     VARCHAR(100) NOT NULL,
    scope      VARCHAR(50) DEFAULT 'own',
    conditions JSONB,
    UNIQUE(resource, action, scope)
);

CREATE TABLE role_permissions (
    role_id       UUID REFERENCES roles(id) ON DELETE CASCADE,
    permission_id UUID REFERENCES permissions(id) ON DELETE CASCADE,
    PRIMARY KEY (role_id, permission_id)
);

CREATE TABLE role_inheritance (
    role_id        UUID REFERENCES roles(id) ON DELETE CASCADE,
    parent_role_id UUID REFERENCES roles(id) ON DELETE CASCADE,
    PRIMARY KEY (role_id, parent_role_id)
);

CREATE TABLE user_roles (
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    role_id UUID REFERENCES roles(id) ON DELETE CASCADE,
    PRIMARY KEY (user_id, role_id)
);
```

---

### 5.12 FR-11：多會員架構 🔮

#### 5.12.1 功能描述

支援多組織在同一平台上獨立運營。

#### 5.12.2 資源配額設計

```yaml
tenant_quota:
  storage:
    max_total_size_gb: 100
    max_file_size_mb: 50
    max_files_count: 10000

  tasks:
    max_concurrent_tasks: 5
    max_daily_tasks: 100
    max_files_per_task: 50

  users:
    max_users: 50
    max_admins: 5

  knowledge_base:
    max_documents: 5000
    max_collections: 10

  api:
    requests_per_minute: 100
    requests_per_day: 10000
```

#### 5.12.3 資料模型

```sql
CREATE TABLE tenants (
    id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name       VARCHAR(255) NOT NULL,
    slug       VARCHAR(100) NOT NULL UNIQUE,
    status     VARCHAR(50) DEFAULT 'active',
    quota      JSONB NOT NULL DEFAULT '{}',
    settings   JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- 修改 users 表，加入 tenant_id
ALTER TABLE users ADD COLUMN tenant_id UUID REFERENCES tenants(id);
CREATE INDEX idx_users_tenant_id ON users(tenant_id);

-- 啟用 Row Level Security
ALTER TABLE files ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation_files ON files
    USING (tenant_id = current_setting('app.tenant_id')::UUID);
```

---

### 5.13 FR-12：多知識庫管理 🔮

#### 5.13.1 功能描述

支援建立多個獨立的知識庫集合。

#### 5.13.2 資料模型

```sql
CREATE TABLE knowledge_bases (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id   UUID REFERENCES tenants(id),
    name        VARCHAR(255) NOT NULL,
    description TEXT,
    settings    JSONB NOT NULL DEFAULT '{}',
    status      VARCHAR(50) DEFAULT 'active',
    created_by  UUID REFERENCES users(id),
    created_at  TIMESTAMP DEFAULT NOW(),
    updated_at  TIMESTAMP DEFAULT NOW()
);

CREATE TABLE kb_documents (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    kb_id       UUID REFERENCES knowledge_bases(id) ON DELETE CASCADE,
    file_id     VARCHAR(36) REFERENCES files(id),
    title       VARCHAR(255),
    status      VARCHAR(50) DEFAULT 'pending',
    chunk_count INTEGER DEFAULT 0,
    metadata    JSONB,
    version     INTEGER DEFAULT 1,
    created_at  TIMESTAMP DEFAULT NOW(),
    updated_at  TIMESTAMP DEFAULT NOW()
);

CREATE TABLE kb_document_versions (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id UUID REFERENCES kb_documents(id) ON DELETE CASCADE,
    version     INTEGER NOT NULL,
    file_id     VARCHAR(36) REFERENCES files(id),
    changes     TEXT,
    created_by  UUID REFERENCES users(id),
    created_at  TIMESTAMP DEFAULT NOW()
);
```

---

### 5.14 FR-13：智慧分析儀表板 🔮

#### 5.14.1 功能描述

提供視覺化的數據分析儀表板。

#### 5.14.2 API 規格

**GET /api/v1/analytics/overview - 取得概覽數據**

```yaml
Request:
  Query:
    period: "day" | "week" | "month" (default: "week")

Response 200:
  {
    "success": true,
    "data": {
      "period": {
        "start": "2026-01-29",
        "end": "2026-02-05"
      },
      "summary": {
        "activeUsers": 45,
        "tasksCompleted": 234,
        "filesProcessed": 892,
        "entitiesDetected": 15678
      },
      "trends": {
        "tasks": [
          { "date": "2026-01-29", "count": 28 },
          ...
        ]
      },
      "topEntityTypes": [
        { "type": "PERSON", "count": 5234, "percentage": 33.4 },
        ...
      ]
    }
  }
```

**GET /api/v1/analytics/compliance - 取得合規分析**

```yaml
Response 200:
  {
    "success": true,
    "data": {
      "complianceScore": 87.5,
      "lastUpdated": "2026-02-05T10:00:00.000Z",
      "breakdown": {
        "dataProtection": 92,
        "accessControl": 85,
        "auditTrail": 90,
        "encryption": 83
      },
      "risks": [
        {
          "id": "risk-001",
          "severity": "high",
          "category": "dataProtection",
          "description": "發現 15 個檔案包含未處理的信用卡資訊",
          "recommendation": "執行清洗任務處理敏感資料"
        }
      ]
    }
  }
```

---

### 5.15 FR-14：第三方整合 🔮

#### 5.15.1 功能描述

提供 Webhook 和 API Gateway 整合能力。

#### 5.15.2 Webhook 事件

```yaml
webhook_events:
  - event: "task.created"
    payload: { taskId, fileCount, createdAt }

  - event: "task.completed"
    payload: { taskId, status, entitiesFound, processingTime }

  - event: "task.failed"
    payload: { taskId, error }

  - event: "file.uploaded"
    payload: { fileId, filename, size }

  - event: "compliance.risk_detected"
    payload: { riskId, severity, description }
```

---

### 5.16 FR-15：運維監控與告警 🔮

#### 5.16.1 功能描述

提供系統運維監控能力。

#### 5.16.2 監控指標

```yaml
system_metrics:
  - name: "cpu_usage_percent"
    type: "gauge"
    alert_threshold: 80

  - name: "memory_usage_percent"
    type: "gauge"
    alert_threshold: 85

  - name: "http_request_duration_seconds"
    type: "histogram"
    buckets: [0.1, 0.5, 1, 2, 5]

  - name: "task_queue_length"
    type: "gauge"
    alert_threshold: 100

alert_rules:
  - name: "HighCPUUsage"
    condition: "cpu_usage_percent > 80 for 5m"
    severity: "warning"
    channels: ["email", "slack"]

  - name: "HighErrorRate"
    condition: "rate(errors_total[5m]) > 0.05"
    severity: "critical"
    channels: ["email", "slack", "pagerduty"]
```

---

### 5.17 FR-16：三層式回應機制 🔄

#### 5.17.1 功能描述

提供三種回應層級（beginner/standard/expert），各層級有不同的 system prompt、回應風格與引用深度。

#### 5.17.2 回應層級定義

| 回應層級 | 代碼 | 目標使用者 | 回應風格 | 引用深度 |
|------|------|-----------|---------|---------|
| 新手 | beginner | 非技術人員 | 白話文、故事化、類比說明 | 僅結論，不引用法條編號 |
| 一般 | standard | IT/MIS 工程師 | 實務技術建議、操作步驟 | 引用相關標準章節 |
| 顧問 | expert | 資安顧問 | 專業分析、風險評估 | 完整引用 ISO 條款/法規依據 |

#### 5.17.3 角色預設回應層級對應

| 角色 | 預設回應層級 | 可切換範圍 |
|------|---------|-----------|
| basic_user | beginner | beginner only |
| user | standard | beginner, standard |
| consultant | expert | beginner, standard, expert |
| data_cleaner | standard | standard only |
| data_reviewer | standard | standard only |
| admin | standard | beginner, standard, expert |

#### 5.17.3.1 新手每日訊息額度

- 僅 `basic_user` 受限，預設 20 次，可由 Admin 設定 1～1000。
- 日界採 `Asia/Taipei` 每日 00:00；並行請求以資料庫原子條件更新計數。
- 額度用完回 `429`，錯誤碼 `CHAT_DAILY_QUOTA_EXCEEDED`，並回傳 `limit`、`used`、`remaining`、`resetAt`。
- `GET /api/chat/modes` 回應包含目前使用者的 `quota` 狀態。

#### 5.17.4 使用者故事

| US 編號 | 使用者故事 | 驗收標準 |
|---------|------------|----------|
| US-16-01 | 身為一般方案使用者，我希望系統預設以一般層級回應 | 1. 預設一般層級<br>2. 可切換新手／一般<br>3. 無法切換顧問 |
| US-16-02 | 身為顧問方案使用者，我希望切換回應層級取得不同深度的回答 | 1. 可切換新手／一般／顧問<br>2. 顧問回答含 ISO 條款<br>3. 切換即時生效 |
| US-16-03 | 身為系統管理員，我希望管理各回應層級的提示詞 | 1. 固定顯示三筆啟用提示詞<br>2. 可編輯與預覽各層級差異 |

#### 5.17.5 API 規格

**POST /api/chat — 擴展 `responseMode` 欄位**

```yaml
Request:
  Headers:
    Authorization: Bearer <accessToken>
  Content-Type: application/json
  Body:
    message: string (required)
    conversationId: string (optional)
    responseMode: string (optional, enum: beginner|standard|expert)
    options:
      stream: boolean (default: true)
      temperature: number (default: 0.7)
```

> 若未提供 `responseMode`，系統依使用者角色自動選擇預設回應層級；內部欄位名稱維持相容。

**GET /api/chat/modes — 取得可用回應層級**

```yaml
Request:
  Headers:
    Authorization: Bearer <accessToken>

Response 200:
  {
    "success": true,
    "data": {
      "currentMode": "standard",
      "availableModes": ["beginner", "standard"],
      "modes": [
        {
          "code": "beginner",
          "name": "新手",
          "description": "以白話文、故事化方式回應，適合非技術背景使用者"
        },
        {
          "code": "standard",
          "name": "一般",
          "description": "提供實務技術建議與操作步驟，適合 IT/MIS 工程師"
        }
      ]
    }
  }
```

#### 5.17.6 資料模型

新增 ResponseMode enum（beginner, standard, expert）。

prompt_templates 表已在 FR-03（§5.4.5）擴展 `mode` 欄位。

conversations 表擴展：

```sql
ALTER TABLE conversations ADD COLUMN response_mode VARCHAR(50) DEFAULT 'standard';
```

---

### 5.18 FR-17：在地化法規知識庫 ✅/🔄

#### 5.18.1 功能描述

建構以台灣資安法規與國際標準為核心的 RAG 知識庫，確保系統回答具備在地化法規依據。

#### 5.18.2 知識庫範圍

| 分類 | 內容 | 階段 |
|------|------|------|
| 台灣法規 | 資通安全管理法、個人資料保護法、電子簽章法 | Phase 1 |
| 國際標準 | ISO 27001、ISO 27701、ISO 42001 | Phase 1-2 |
| 產業指引 | 金管會資安規範、衛福部個資指引 | Phase 2 |

#### 5.18.3 使用者故事

| US 編號 | 使用者故事 | 驗收標準 |
|---------|------------|----------|
| US-17-01 | 身為使用者，我希望查詢台灣資安法規相關問題時能得到基於法規原文的回答 | 1. 回答引用法規條文<br>2. 顯示引用來源文件<br>3. 法規內容為最新版本 |

---

### 5.19 FR-18：清洗審核管理 (Cleaner App) ✅

#### 5.19.1 功能描述

提供獨立的清洗審核管理前端應用 (apps/cleaner, Port 5503)，支援完整的審核工作流：查看清洗結果 → 標籤分類 → 內容編輯 → 批准/駁回 → 送入 RAG 知識庫。本功能供 `data_cleaner` 與 `admin` 角色使用。

#### 5.19.2 技術架構

| 層級 | 技術選擇 | 說明 |
|------|----------|------|
| **框架** | React 19 + Vite 6 | 前端建構工具 |
| **UI 元件** | Ant Design v6 | 企業級 UI 組件庫 |
| **資料獲取** | TanStack React Query | 伺服器狀態管理與快取 |
| **路由** | React Router v7 | SPA 路由管理 |
| **HTTP 客戶端** | Axios | API 請求封裝 |
| **Dev Port** | 5503 | 開發伺服器連接埠 |
| **API 代理** | Vite dev server → NestJS (3051) | 開發環境 API 代理 |

#### 5.19.3 使用者故事

| US 編號 | 使用者故事 | 驗收標準 |
|---------|------------|----------|
| US-18-01 | 身為資料清洗人員，我希望查看待審核的清洗任務 | 1. 列出所有已完成清洗的任務<br>2. 顯示任務基本資訊與審核狀態<br>3. 可依狀態篩選 |
| US-18-02 | 身為資料清洗人員，我希望逐檔檢視清洗後的內容 | 1. 顯示清洗前後內容對比<br>2. 標示偵測到的 PII 實體位置<br>3. 顯示實體類型與去識別化策略 |
| US-18-03 | 身為資料清洗人員，我希望手動修正清洗結果中的錯誤 | 1. 提供內容編輯功能<br>2. 儲存編輯後內容<br>3. 記錄修改歷程 |
| US-18-04 | 身為資料清洗人員，我希望為檔案標記分類標籤 | 1. 可新增/移除標籤<br>2. 支援自訂標籤名稱<br>3. 標籤用於後續知識庫分類 |
| US-18-05 | 身為資料清洗人員，我希望批准或駁回清洗任務 | 1. 可批准已審核完成的任務<br>2. 可駁回品質不合格的任務<br>3. 記錄審核意見 |
| US-18-06 | 身為系統管理員，我希望將批准的資料送入 RAG 知識庫 | 1. 僅 data_reviewer / admin 可執行送入操作<br>2. 顯示送入進度<br>3. 記錄送入結果 |
| US-18-07 | 身為資料清洗人員，我希望瀏覽所有上傳的來源資料 | 1. 列出所有檔案與審核狀態<br>2. 支援檔名、類型與審核狀態篩選<br>3. 篩選後的總數與分頁一致且穩定 |
| US-18-08 | 身為資料清洗人員，我希望瀏覽知識庫中的向量化文件 | 1. 列出 Qdrant 中的文件<br>2. 依來源/標籤篩選<br>3. 顯示內容摘要 |
| US-18-09 | 身為系統管理員，我希望按來源刪除知識庫文件 | 1. 選擇來源<br>2. 確認後刪除<br>3. 顯示刪除結果 |

#### 5.19.4 審核工作流狀態機

```
                    清洗完成
                       │
                       ▼
               ┌──────────────┐
               │   pending    │ (待審核)
               └──────┬───────┘
                      │ 送審 (submit_for_review)
                      ▼
               ┌──────────────┐
               │  review_requested  │ (已送審：等待審核)
               └──────┬───────┘
                      │
            ┌─────────┼─────────┐
            │                   │
            ▼                   ▼
    ┌──────────────┐    ┌──────────────┐
    │   approved   │    │   rejected   │
    │  (已批准)     │    │  (已駁回)     │
    └──────┬───────┘    └──────┬───────┘
           │                   │ 重新編輯後再次送審
           │                   │ (submit_for_review 重設
           │                   │  file review_status → pending
           │                   │  且不清除 review_note)
           │                   ▼
           │           ┌──────────────┐
           │           │ review_requested │ (重送)
           │           └──────────────┘
           │ 送入 RAG (data_reviewer / admin)
           ▼
    ┌──────────────┐
    │   ingested   │
    │ (已匯入知識庫) │
    └──────────────┘

註：rejected 狀態保留作為審核歷史軌跡，不會自動回到 pending。
   重送由 submit_for_review 處理：rejected → review_requested。
   送入失敗會補償本次 Qdrant/BM25 資料並 rollback DB 版本異動，
   任務維持 approved 且 ingested_at 為 NULL，可重新送入。
   並發保護：approve / reject / ingest / update_file_review_status
   均使用 SELECT ... FOR UPDATE 鎖定 task 列（PR-2 起）。
```

#### 5.19.5 Review API 規格（7 支 API）

**GET /api/v1/review/{task_id} - 取得任務審核資訊**

```yaml
Request:
  Headers:
    Authorization: Bearer <accessToken>
  Path:
    task_id: string (required)

Response 200:
  {
    "success": true,
    "data": {
      "task_id": "task-uuid",
      "status": "completed",
      "approval_status": "pending",
      "files_total": 3,
      "files_reviewed": 1,
      "created_at": "2026-02-10T10:00:00.000Z",
      "completed_at": "2026-02-10T10:05:00.000Z",
      "rule_profile": {
        "id": "profile-uuid",
        "name": "標準台灣 PII 規則"
      },
      "files": [
        {
          "file_id": "file-uuid-1",
          "original_name": "Q1_財務報告.pdf",
          "entities_found": 32,
          "review_status": "pending",
          "tags": [],
          "reviewed_by": null,
          "reviewed_at": null
        }
      ]
    }
  }
```

**GET /api/v1/review/{task_id}/files/{file_id}/content - 取得檔案內容**

```yaml
Request:
  Headers:
    Authorization: Bearer <accessToken>
  Path:
    task_id: string (required)
    file_id: string (required)

Response 200:
  {
    "success": true,
    "data": {
      "file_id": "file-uuid",
      "original_name": "Q1_財務報告.pdf",
      "original_content": "客戶王小明（身分證 A123456789）...",
      "cleaned_content": "客戶陳大華（身分證 **********）...",
      "edited_content": null,
      "entities": [
        {
          "entity_type": "PERSON",
          "original_text": "王小明",
          "anonymized_text": "陳大華",
          "start": 2,
          "end": 5,
          "score": 0.95,
          "strategy": "pseudonymize"
        }
      ],
      "tags": ["財務", "Q1"],
      "review_status": "pending",
      "review_note": null
    }
  }
```

**PUT /api/v1/review/{task_id}/files/{file_id}/content - 更新編輯內容**

```yaml
Request:
  Headers:
    Authorization: Bearer <accessToken>
  Content-Type: application/json
  Path:
    task_id: string (required)
    file_id: string (required)
  Body:
    edited_content: string (required)

Response 200:
  {
    "success": true,
    "data": {
      "file_id": "file-uuid",
      "edited_content": "更新後的內容...",
      "updated_at": "2026-02-10T11:00:00.000Z"
    },
    "message": "檔案內容已更新"
  }
```

**PUT /api/v1/review/{task_id}/files/{file_id}/tags - 更新標籤**

```yaml
Request:
  Headers:
    Authorization: Bearer <accessToken>
  Content-Type: application/json
  Path:
    task_id: string (required)
    file_id: string (required)
  Body:
    tags: string[] (required)

Response 200:
  {
    "success": true,
    "data": {
      "file_id": "file-uuid",
      "tags": ["法規類", "個資法", "Q1"],
      "updated_at": "2026-02-10T11:05:00.000Z"
    },
    "message": "標籤已更新"
  }
```

**PUT /api/v1/review/{task_id}/files/{file_id}/status - 審核狀態**

```yaml
Request:
  Headers:
    Authorization: Bearer <accessToken>
  Content-Type: application/json
  Path:
    task_id: string (required)
    file_id: string (required)
  Body:
    review_status: string (required, enum: "approved" | "rejected")
    review_note: string (optional)

Response 200:
  {
    "success": true,
    "data": {
      "file_id": "file-uuid",
      "review_status": "approved",
      "reviewed_by": "user-uuid",
      "reviewed_at": "2026-02-10T11:10:00.000Z",
      "review_note": "內容確認無誤"
    },
    "message": "檔案審核狀態已更新"
  }
```

**POST /api/v1/review/{task_id}/approve - 批准任務**

```yaml
Request:
  Headers:
    Authorization: Bearer <accessToken>
  Content-Type: application/json
  Path:
    task_id: string (required)
  Body:
    note: string (optional)

Response 200:
  {
    "success": true,
    "data": {
      "task_id": "task-uuid",
      "approval_status": "approved",
      "approved_by": "user-uuid",
      "approved_at": "2026-02-10T11:30:00.000Z"
    },
    "message": "任務已批准"
  }

Response 400:
  {
    "success": false,
    "error": {
      "code": "FILES_NOT_REVIEWED",
      "message": "尚有 2 個檔案未完成審核"
    }
  }
```

**POST /api/v1/review/{task_id}/ingest - 送入 RAG 知識庫**

> 僅 `data_reviewer` 或 `admin` 角色可執行此操作。

```yaml
Request:
  Headers:
    Authorization: Bearer <accessToken>
  Content-Type: application/json
  Path:
    task_id: string (required)
  Body:
    tags: string[] (optional, 額外全域標籤)
    collection: string (optional, Qdrant collection 名稱)

Response 200:
  {
    "success": true,
    "data": {
      "task_id": "task-uuid",
      "ingested_at": "2026-02-10T12:00:00.000Z",
      "ingest_result": {
        "files_ingested": 3,
        "chunks_created": 45,
        "vectors_stored": 45,
        "errors": []
      }
    },
    "message": "已成功送入 RAG 知識庫"
  }

Response 403:
  {
    "success": false,
    "error": {
      "code": "FORBIDDEN",
      "message": "僅系統管理員可執行送入 RAG 知識庫操作"
    }
  }

Response 400:
  {
    "success": false,
    "error": {
      "code": "TASK_NOT_APPROVED",
      "message": "任務尚未批准，無法送入 RAG 知識庫"
    }
  }

Response 503:
  {
    "success": false,
    "error": {
      "message": "送入 RAG 失敗，任務仍維持已批准，可重新嘗試",
      "errors": ["file-uuid: qdrant unavailable"],
      "compensation_errors": []
    }
  }
```

送入流程採任務級補償：任一檔案或外部儲存失敗時，成功檔案也不會讓任務進入 `ingested`；Qdrant 使用來源 filter 刪除全部符合向量，並同步清除 BM25。法規版本以 `effective_from` 為主，依序備援 `effective_date`、`version_date` 與目前日期；完整 current／deprecated 歷程會重建為不重疊區間，僅較新的生效日可取代現行版。自動入庫與人工取代同步關閉 PostgreSQL／Qdrant 有效期間，失敗時反向還原；vector、hierarchical 與 hybrid BM25-only 結果都套用法規生命週期條件。

#### 5.19.6 資料模型擴展

**task_files 表新增欄位：**

```sql
ALTER TABLE task_files ADD COLUMN tags JSONB DEFAULT '[]';
ALTER TABLE task_files ADD COLUMN edited_content TEXT;
ALTER TABLE task_files ADD COLUMN review_status VARCHAR(50) DEFAULT 'pending';
ALTER TABLE task_files ADD COLUMN reviewed_by UUID REFERENCES users(id) ON DELETE SET NULL;
ALTER TABLE task_files ADD COLUMN reviewed_at TIMESTAMP;
ALTER TABLE task_files ADD COLUMN review_note TEXT;

CREATE INDEX idx_task_files_review_status ON task_files(review_status);
```

**tasks 表新增欄位：**

```sql
ALTER TABLE tasks ADD COLUMN approval_status VARCHAR(50) DEFAULT 'pending';
ALTER TABLE tasks ADD COLUMN submitted_by UUID REFERENCES users(id) ON DELETE SET NULL;
ALTER TABLE tasks ADD COLUMN submitted_at TIMESTAMP;
ALTER TABLE tasks ADD COLUMN approved_by UUID REFERENCES users(id) ON DELETE SET NULL;
ALTER TABLE tasks ADD COLUMN approved_at TIMESTAMP;
ALTER TABLE tasks ADD COLUMN ingested_at TIMESTAMP;
ALTER TABLE tasks ADD COLUMN ingest_result JSONB;

CREATE INDEX idx_tasks_approval_status ON tasks(approval_status);
```

**新增欄位說明：**

| 表 | 欄位 | 類型 | 說明 |
|----|------|------|------|
| task_files | tags | JSONB | 檔案分類標籤陣列，如 `["法規類", "Q1"]` |
| task_files | edited_content | TEXT | 審核人員手動編輯後的內容 |
| task_files | review_status | VARCHAR(50) | 檔案審核狀態：pending / approved / rejected |
| task_files | reviewed_by | UUID | 審核人員 ID (FK → users) |
| task_files | reviewed_at | TIMESTAMP | 審核時間 |
| task_files | review_note | TEXT | 審核備註 |
| tasks | approval_status | VARCHAR(50) | 任務批准狀態：pending / review_requested / approved / rejected / ingested |
| tasks | submitted_by | UUID | 送審提交者 ID (FK → users)，Maker-Checker 中的 Maker |
| tasks | submitted_at | TIMESTAMP | 送審提交時間 |
| tasks | approved_by | UUID | 批准人員 ID (FK → users)，Maker-Checker 中的 Checker |
| tasks | approved_at | TIMESTAMP | 批准時間 |
| tasks | ingested_at | TIMESTAMP | 送入 RAG 知識庫時間 |
| tasks | ingest_result | JSONB | 送入結果，含 files_ingested / chunks_created / errors |

**Maker-Checker 職責分離規則：**

> 以下四個端點強制執行 Maker-Checker 規則：**送審者（submitted_by）不得為審批者（approved_by）**。
> 操作者身份由 NestJS JWT `@CurrentUser` 注入，FastAPI 端透過 `X-User-Id` header 驗證。

| 端點 | 動作 | Maker-Checker 驗證 |
|------|------|-------------------|
| `POST /api/v1/review/{task_id}/approve` | 批准任務 | `approved_by != submitted_by` |
| `POST /api/v1/review/{task_id}/reject` | 駁回任務 | `rejected_by != submitted_by` |
| `PUT /api/v1/review/{task_id}/files/{file_id}/status` | 檔案審核 | `reviewed_by != submitted_by` |
| `POST /api/v1/review/{task_id}/ingest` | 送入 RAG | `ingested_by != submitted_by`（額外限制：僅 `admin` 角色） |

違反 Maker-Checker 規則時回應：

```yaml
Response 403:
  {
    "success": false,
    "error": {
      "code": "MAKER_CHECKER_VIOLATION",
      "message": "送審者不得為審批者（Maker-Checker 職責分離）"
    }
  }
```

---

### 5.20 FR-19：資料分析儀表板 (Analytics Dashboard) ✅

#### 5.20.1 功能描述

提供清洗統計、知識庫統計、時間軸統計等資料分析功能，位於 Cleaner App 中。供 `data_cleaner` 與 `admin` 角色使用。

#### 5.20.2 使用者故事

| US 編號 | 使用者故事 | 驗收標準 |
|---------|------------|----------|
| US-19-01 | 身為資料清洗人員，我希望查看清洗統計 | 1. 顯示任務完成率<br>2. PII 實體偵測分佈圖<br>3. 平均處理時間 |
| US-19-02 | 身為系統管理員，我希望查看知識庫統計 | 1. 已匯入文件數<br>2. 向量數量<br>3. 知識庫覆蓋度 |
| US-19-03 | 身為資料清洗人員，我希望查看時間軸統計 | 1. 每日/週/月處理量趨勢<br>2. 可自訂時間範圍<br>3. 圖表視覺化 |

#### 5.20.3 Analytics API 規格（3 支 API）

**GET /api/v1/analytics/cleaning - 清洗統計**

```yaml
Request:
  Headers:
    Authorization: Bearer <accessToken>
  Query:
    period: "day" | "week" | "month" (default: "month")
    start_date: string (optional, ISO 8601)
    end_date: string (optional, ISO 8601)

Response 200:
  {
    "success": true,
    "data": {
      "period": {
        "start": "2026-01-01",
        "end": "2026-02-10"
      },
      "summary": {
        "total_tasks": 120,
        "completed_tasks": 105,
        "failed_tasks": 5,
        "cancelled_tasks": 10,
        "completion_rate": 87.5,
        "total_files_processed": 450,
        "total_entities_found": 12500,
        "avg_processing_time_seconds": 8.2
      },
      "entity_distribution": [
        { "entity_type": "PERSON", "count": 4200, "percentage": 33.6 },
        { "entity_type": "EMAIL_ADDRESS", "count": 2100, "percentage": 16.8 },
        { "entity_type": "TW_ID", "count": 1800, "percentage": 14.4 },
        { "entity_type": "PHONE_NUMBER", "count": 1500, "percentage": 12.0 }
      ],
      "strategy_distribution": [
        { "strategy": "mask", "count": 5000, "percentage": 40.0 },
        { "strategy": "pseudonymize", "count": 3500, "percentage": 28.0 },
        { "strategy": "partial_mask", "count": 2000, "percentage": 16.0 }
      ],
      "review_stats": {
        "pending_review": 8,
        "approved": 95,
        "rejected": 12,
        "ingested": 80
      }
    }
  }
```

**GET /api/v1/analytics/knowledge-base - 知識庫統計**

```yaml
Request:
  Headers:
    Authorization: Bearer <accessToken>

Response 200:
  {
    "success": true,
    "data": {
      "documents": {
        "total": 250,
        "by_source": {
          "upload": 180,
          "gdrive": 70
        },
        "by_type": {
          "pdf": 120,
          "docx": 80,
          "txt": 30,
          "xlsx": 20
        }
      },
      "vectors": {
        "total_chunks": 5200,
        "total_vectors": 5200,
        "collection_size_mb": 128.5
      },
      "tags": [
        { "tag": "法規類", "count": 85 },
        { "tag": "技術指引", "count": 60 },
        { "tag": "政策文件", "count": 45 }
      ],
      "last_ingested_at": "2026-02-10T12:00:00.000Z"
    }
  }
```

**GET /api/v1/analytics/timeline - 時間軸統計**

```yaml
Request:
  Headers:
    Authorization: Bearer <accessToken>
  Query:
    granularity: "day" | "week" | "month" (default: "day")
    start_date: string (required, ISO 8601)
    end_date: string (required, ISO 8601)
    metrics: string (optional, comma-separated: "tasks,files,entities,ingestions")

Response 200:
  {
    "success": true,
    "data": {
      "granularity": "day",
      "period": {
        "start": "2026-02-01",
        "end": "2026-02-10"
      },
      "timeline": [
        {
          "date": "2026-02-01",
          "tasks_created": 5,
          "tasks_completed": 4,
          "files_processed": 18,
          "entities_found": 520,
          "files_ingested": 12
        },
        {
          "date": "2026-02-02",
          "tasks_created": 8,
          "tasks_completed": 7,
          "files_processed": 25,
          "entities_found": 780,
          "files_ingested": 20
        }
      ],
      "totals": {
        "tasks_created": 45,
        "tasks_completed": 40,
        "files_processed": 150,
        "entities_found": 4500,
        "files_ingested": 100
      }
    }
  }
```

---

### 5.21 FR-20：來源資料瀏覽 ✅

#### 5.21.1 使用者故事與驗收規則

| US 編號 | 使用者故事 | 驗收標準 |
|---------|------------|----------|
| US-20-01 | 身為資料清洗人員，我希望查看所有來源檔案、最新任務名稱及審核狀態 | 支援關鍵字、類型、審核狀態篩選；最新任務名稱、狀態與操作連結取自同一筆確定性最新任務；條件在計數與分頁前套用；預設排除已邏輯刪除資料，`pipeline_status=deleted` 則只回傳已刪除資料 |
| US-20-02 | 身為系統管理員，我希望移除不應繼續審核的待審核來源資料 | 管理員限定；先取得完整影響，填寫原因並輸入「刪除」；交易內鎖定並重驗所有關聯、Qdrant 切塊及刪除資格 |

#### 5.21.2 審核狀態與分頁契約

| 邊界 | 必要行為 |
|------|----------|
| 最新任務 | 同一來源有多筆任務時，以 `tasks.created_at DESC, tasks.id DESC` 選出唯一最新任務。 |
| 任務名稱 | `GET /api/v1/files` 由同一最新任務回傳 `latest_task_id` 與 `latest_task_name`；Cleaner 顯示名稱，空值／空白字串以「任務 {ID 前 8 碼}」備援，無任務顯示 `-`。長名稱省略並由 tooltip 顯示全文。 |
| 狀態層級 | 來源狀態依最新任務的 `status`／`approval_status` 判定；逐檔 `task_files.review_status` 不得覆寫任務工作流狀態。 |
| 未送審 | 最新任務 `status=completed` 且 `approval_status=pending` 時回傳 `pending_submission`，Cleaner 顯示「未送審」。 |
| 篩選順序 | 有效／已刪除資料集、檔案類型、檔名與審核狀態都在 SQL 的 `COUNT`、排序與 `LIMIT/OFFSET` 前套用。 |
| 穩定分頁 | 以 `files.uploaded_at DESC, files.id DESC` 排序；相同條件下每頁 `total` 相同，換頁不得遺漏或重複資料。 |
| 無效狀態 | 不在支援列舉內的 `pipeline_status` 回傳 HTTP 422，不執行資料查詢。 |

#### 5.21.3 刪除資格與資料保留

| 邊界 | 必要行為 |
|------|----------|
| 最新狀態 | 最新關聯必須為「未送審」或「待審核」；沒有任務關聯、處理中、已退回、已批准或已入庫均禁止刪除。逐檔已標記「未通過」時須先由任務流程處理。 |
| 全關聯檢查 | 不只檢查最新任務；任一歷史關聯已批准、已入庫或仍在處理即禁止。 |
| 知識庫檢查 | 以來源路徑中的精確 File UUID 統計 Qdrant 切塊；有切塊或服務不可用時採失敗關閉。 |
| 邏輯刪除 | 寫入 `files.deleted_at`、`deleted_by`、`deletion_reason`；不移除原始檔、清洗產物、TaskFile 或任務歷史。 |
| 任務一致性 | 排除已刪除來源後，重新計算每一筆關聯任務的檔案數、完成數及實體數；有效檔案歸零時將任務設為 `cancelled`、進度 0、審核狀態 `pending`。 |
| 歷史檢視 | 任務詳情保留已刪除來源並標示「來源已刪除」，但審核、送審、批准及入庫路徑皆不得處理該資料。 |
| 來源清單 | 已刪除資料回傳 `pipeline_status=deleted`；有關聯任務時只提供最新任務歷史入口，無關聯任務時操作欄顯示 `-`。未處理有效資料的操作欄同樣顯示 `-`，tooltip 說明尚未建立任務。 |
| 操作紀錄 | 以實際 JWT 管理員身分記錄 `source_file_soft_deleted`、原因、來源檔名、受影響任務及保留產物數。 |

#### 5.21.4 時間顯示契約

FastAPI 與 NestJS 持續使用 UTC 傳輸時間；Cleaner 共用格式函式將帶 `Z`／offset 的字串依其時區解析，舊資料若沒有 offset 則視為 UTC。來源資料、任務列表、任務詳情、檔案審核、首頁最近活動與知識庫匯入日期都明確以 `Asia/Taipei` 顯示，不依賴瀏覽器或作業系統時區。

#### 5.21.5 併發與錯誤契約

刪除端點於同一資料庫交易鎖定 File、Task 及 TaskFile 後重新執行資格判斷，不信任預檢結果。狀態衝突回 HTTP 409；Qdrant 無法驗證回 HTTP 503；理由或確認文字不符回 HTTP 400；非管理員由 NestJS RBAC 回 HTTP 403。此功能沿用既有 `files` 邏輯刪除欄位，不新增 migration。

---

### 5.22 FR-24：法規／知識分類治理 ✅

#### 5.22.1 契約與處理規則

跨 TypeScript、Python 的類型集合以 `contracts/enums.yml` 為唯一來源。完整集合包含 `general` 與相容用 `other`；新上傳可選集合排除 `other`；法規治理集合固定為 `law`、`regulation`、`iso_standard`、`nist`、`cis`。

| 邊界 | 必要行為 |
|---|---|
| Admin | 一批選一種類型；逐檔 PATCH 全部成功後才 POST clean。 |
| NestJS | DTO 以契約集合驗證 `regulation_type`，維持 JWT、RBAC 與代理責任。 |
| FastAPI metadata／clean | PATCH 與建立任務鎖定同一 File row；任務建立後分類不可變。載入所有 file records 後先檢查分類；缺少時不得建立 Task／TaskFile 或提交 queue。 |
| Review／Ingest | 直接依分類判斷法規治理，不再以 tags 推測。 |
| Regulations／Analytics | 版本、刪除、關聯、影響分析與過期提醒只查法規治理集合；類型分佈顯示全部非空分類。 |

#### 5.22.2 錯誤契約

```json
{
  "detail": {
    "code": "REGULATION_TYPE_REQUIRED",
    "message": "Every file must have a regulation/knowledge type before cleaning",
    "file_ids": ["<uuid>"]
  }
}
```

任務建立後再變更分類時回傳 HTTP 409：

```json
{
  "detail": {
    "code": "REGULATION_TYPE_LOCKED",
    "message": "Regulation/knowledge type cannot change after cleaning has started",
    "file_id": "<uuid>"
  }
}
```

#### 5.22.3 資料遷移

Alembic `011` 建立 migration-owned ledger，記錄原本 `regulation_type IS NULL` 的 file IDs，再回填 `general`。資料欄位維持 nullable，因為檔案上傳與分類是兩個 API 步驟；完整性由建立清洗任務的服務邊界保證。

---

## 6. 非功能需求 (NFR)

### 6.1 NFR-01：效能需求

| 需求編號 | 需求描述 | 目標值 | 階段 |
|----------|----------|--------|------|
| NFR-01-01 | RAG 查詢回應時間 | P95 < 10 秒 | Phase 1 |
| NFR-01-02 | RAG 查詢回應時間 | P95 < 7 秒 | Phase 2-3 |
| NFR-01-03 | 檔案上傳處理時間 | < 5 秒 (50MB) | Phase 1 |
| NFR-01-04 | 檔案上傳處理時間 | < 3 秒 (50MB) | Phase 2 |
| NFR-01-05 | 清洗任務吞吐量 | > 10 檔案/分鐘 | Phase 1 |
| NFR-01-06 | 清洗任務吞吐量 | > 50 檔案/分鐘 | Phase 3 |
| NFR-01-07 | WebSocket 延遲 | < 500ms | Phase 1 |
| NFR-01-08 | API 並發處理 | > 100 RPS | Phase 1 |

> RAG 查詢回應時間含 API 閘道路由與外部 AI 平台（Gemini/OpenAI）推論延遲。
| NFR-01-09 | API 並發處理 | > 1000 RPS | Phase 3 |
| NFR-01-10 | 資料庫查詢時間 | P95 < 100ms | Phase 2 |

### 6.2 NFR-02：安全性需求

| 需求編號 | 需求描述 | 實作方式 | 階段 |
|----------|----------|----------|------|
| NFR-02-01 | 傳輸層加密 | HTTPS/TLS 1.3 | Phase 1 |
| NFR-02-02 | 身分驗證 | JWT + Refresh Token | Phase 2 |
| NFR-02-03 | 密碼儲存 | bcrypt (cost factor 12) | Phase 2 |
| NFR-02-04 | API 認證 | Bearer Token | Phase 2 |
| NFR-02-05 | 輸入驗證 | 請求參數驗證、XSS 防護 | Phase 1 |
| NFR-02-06 | 檔案掃描 | 上傳檔案惡意程式掃描 | Phase 2 |
| NFR-02-07 | 稽核追蹤 | 所有操作記錄 | Phase 1 |
| NFR-02-08 | OAuth2/OIDC | Keycloak | Phase 3 |
| NFR-02-09 | MFA 支援 | TOTP/WebAuthn | Phase 3 |
| NFR-02-10 | 資料加密 | AES-256 靜態加密 | Phase 2 |
| NFR-02-11 | WAF 防護 | CloudFlare/AWS WAF | Phase 3 |

### 6.3 NFR-03：可用性需求

| 需求編號 | 需求描述 | 目標值 | 階段 |
|----------|----------|--------|------|
| NFR-03-01 | 系統可用性 | ≥ 99.5% | Phase 1-2 |
| NFR-03-02 | 系統可用性 | ≥ 99.9% | Phase 3 |
| NFR-03-03 | 計畫性維護時間 | < 4 小時/月 | Phase 1-2 |
| NFR-03-04 | 計畫性維護時間 | < 2 小時/月 | Phase 3 |
| NFR-03-05 | 非計畫性停機 | < 1 小時/月 | Phase 1-2 |
| NFR-03-06 | 資料備份頻率 | 每日增量、每週全量 | Phase 1 |
| NFR-03-07 | 資料恢復時間 | < 4 小時 | Phase 1-2 |
| NFR-03-08 | RTO | < 1 小時 | Phase 3 |
| NFR-03-09 | RPO | < 15 分鐘 | Phase 3 |

### 6.4 NFR-04：可擴展性需求

| 需求編號 | 需求描述 | 目標值 | 階段 |
|----------|----------|--------|------|
| NFR-04-01 | 同時線上使用者 | ≥ 100 | Phase 1-2 |
| NFR-04-02 | 同時線上使用者 | ≥ 10,000 | Phase 3 |
| NFR-04-03 | 知識庫文件數 | ≥ 10,000 | Phase 1-2 |
| NFR-04-04 | 知識庫文件數 | ≥ 100,000 | Phase 3 |
| NFR-04-05 | 清洗任務佇列 | ≥ 1,000 待處理 | Phase 1 |
| NFR-04-06 | 會員數量 | ≥ 100 | Phase 3 |
| NFR-04-07 | 儲存空間成長 | 支援線性擴展 | Phase 1 |
| NFR-04-08 | 水平擴展 | 無狀態服務設計 | Phase 2 |

### 6.5 NFR-05：相容性需求

| 需求編號 | 需求描述 | 支援範圍 |
|----------|----------|----------|
| NFR-05-01 | 瀏覽器支援 | Chrome 90+, Firefox 90+, Safari 14+, Edge 90+ |
| NFR-05-02 | 行動裝置 | 響應式設計，支援 iOS/Android |
| NFR-05-03 | API 版本 | 支援 v1 版本，向下相容 |

### 6.6 NFR-06：AI 品質需求

| 需求編號 | 需求描述 | 目標值 | 階段 |
|----------|----------|--------|------|
| NFR-06-01 | AI 回答使用者滿意度 | ≥ 85% | Phase 2 |
| NFR-06-02 | AI 回答準確率（含引用正確性） | ≥ 95% | Phase 2 |
| NFR-06-03 | 模式切換回應差異度 | 三種模式回應風格明顯不同 | Phase 2 |
| NFR-06-04 | 法規知識庫覆蓋率 | ≥ 90%（台灣資安法規） | Phase 1 |

---

## 7. 資料模型

### 7.1 完整 ER 圖

```
┌───────────────┐                    ┌───────────────┐
│   tenants 🔮  │◄───────────────────│    users      │
├───────────────┤    1:N             ├───────────────┤
│ id (PK)       │                    │ id (PK)       │
│ name          │                    │ tenant_id(FK) │
│ slug          │                    │ email         │
│ status        │                    │ name          │
│ quota         │                    │ password      │
│ settings      │                    │ role          │
└───────────────┘                    │ is_active     │
       │                             └───────┬───────┘
       │                                     │
       │                                     │ N:M
       │                             ┌───────┴───────┐
       │                             │  user_roles 🔮│
       │                             └───────┬───────┘
       │                                     │
       │                             ┌───────┴───────┐
       │                             │   roles 🔮    │
       │                             └───────────────┘
       │
       ├──────────────────────┬──────────────────────┐
       │                      │                      │
       ▼                      ▼                      ▼
┌───────────────┐    ┌───────────────┐    ┌───────────────────┐
│knowledge_bases│    │    files      │    │cleaning_audit_logs│
│      🔮       │    ├───────────────┤    ├───────────────────┤
└───────┬───────┘    │ id (PK)       │    │ id (PK)           │
        │            │ original_name │    │ task_id (FK)      │
        │            │ file_type     │    │ action            │
        │            │ size_bytes    │    │ user_id           │
        │            │ auto_tags     │    │ details           │
        │            └───────┬───────┘    └───────────────────┘
        │                    │
        ▼                    │
┌───────────────┐            │
│ kb_documents  │            │
│      🔮       │            │
└───────────────┘            │
                             │
              ┌──────────────┴──────────────┐
              │                             │
              ▼                             ▼
      ┌───────────────┐            ┌──────────────────┐
      │    tasks      │◄───────────│   task_files     │
      ├───────────────┤            ├──────────────────┤
      │ approval_     │            │ tags (JSONB)     │
      │   status      │            │ edited_content   │
      │ submitted_by  │            │ review_status    │
      │ submitted_at  │            │ reviewed_by      │
      │ approved_by   │            │ reviewed_at      │
      │ approved_at   │            │ review_note      │
      │ ingested_at   │            │                  │
      │ ingest_result │            │                  │
      └───────────────┘            └──────────────────┘

┌───────────────┐    ┌───────────────┐    ┌───────────────┐
│  audit_logs   │    │conversations  │    │   messages    │
└───────────────┘    └───────────────┘    └───────────────┘

┌───────────────────┐
│ prompt_templates  │ 🔄
└───────────────────┘

┌───────────────┐    ┌───────────────┐
│  webhooks 🔮  │    │   alerts 🔮   │
└───────────────┘    └───────────────┘
```

### 7.2 資料字典

#### 任務狀態 (TaskStatus)

| 值 | 說明 |
|----|------|
| pending | 等待處理 |
| processing | 處理中 |
| completed | 已完成 |
| failed | 處理失敗 |
| cancelled | 已取消 |

#### 使用者角色 (UserRole)

| 值 | 說明 | 階段 |
|----|------|------|
| basic_user | 新手方案（僅 beginner） | ✅ |
| user | 一般方案（beginner、standard） | ✅ |
| consultant | 顧問方案（beginner、standard、expert） | ✅ |
| data_cleaner | 資料清洗人員（僅 Cleaner App） | ✅ |
| data_reviewer | 資料審核人員（Maker-Checker 審批者） | ✅ |
| admin | 系統管理員 | ✅ |

#### 回應層級 (ResponseMode)

| 值 | 說明 | 階段 |
|----|------|------|
| beginner | 新手 | ✅ |
| standard | 一般 | ✅ |
| expert | 顧問 | ✅ |

#### 實體類型 (EntityType)

| 值 | 說明 | 分類 |
|----|------|------|
| PERSON | 人名 | 標準 PII |
| EMAIL_ADDRESS | 電子郵件 | 標準 PII |
| PHONE_NUMBER | 電話號碼 | 標準 PII |
| LOCATION | 地址/地點 | 標準 PII |
| DATE_TIME | 日期時間 | 標準 PII |
| CREDIT_CARD | 信用卡號 | 標準 PII |
| IBAN_CODE | 國際銀行帳號 | 標準 PII |
| IP_ADDRESS | IP 位址 | 標準 PII |
| URL | 網址 | 標準 PII |
| ORGANIZATION | 組織名稱 | 標準 PII |
| TW_ID | 台灣身分證 | 台灣專屬 |
| TW_PHONE | 台灣手機 | 台灣專屬 |
| TW_UNIFIED_BUSINESS_NO | 統一編號 | 台灣專屬 |
| TW_ADDRESS | 台灣地址 | 台灣專屬 |
| API_KEY | API 金鑰 | 企業機密 |
| ACCESS_TOKEN | 存取權杖 | 企業機密 |
| PRIVATE_KEY | 私密金鑰 | 企業機密 |
| AWS_ACCESS_KEY | AWS 金鑰 | 企業機密 |
| AZURE_KEY | Azure 金鑰 | 企業機密 |
| GCP_KEY | GCP 金鑰 | 企業機密 |

#### 審核狀態 (ReviewStatus)

| 值 | 說明 | 階段 |
|----|------|------|
| pending | 待審核 | ✅ |
| approved | 已通過 | ✅ |
| rejected | 已駁回 | ✅ |

#### 任務批准狀態 (ApprovalStatus)

| 值 | 說明 | 階段 |
|----|------|------|
| pending | 待批准 | ✅ |
| review_requested | 已送審（等待審核） | ✅ |
| approved | 已批准 | ✅ |
| rejected | 已駁回 | ✅ |
| ingested | 已匯入知識庫 | ✅ |

#### 去識別化策略 (StrategyType)

| 值 | 說明 |
|----|------|
| mask | 完全遮罩 |
| partial_mask | 部分遮罩 |
| pseudonymize | 假名替換 |
| generalize | 泛化處理 |
| keep_labeled | 保留標籤 |
| encrypt | 加密替換 |

---

## 8. API 規格總覽

### 8.1 端點清單

| 分類 | 方法 | 路徑 | 說明 | 狀態 |
|------|------|------|------|------|
| **認證** | POST | /api/auth/login | 使用者登入 | 🔄 |
| | POST | /api/auth/logout | 使用者登出 | 🔄 |
| | POST | /api/auth/refresh | 更新 Token | 🔄 |
| **對話** | POST | /api/chat | 發送對話訊息 | ✅ |
| | GET | /api/chat/modes | 取得可用回應層級與每日額度 | ✅ |
| | GET/PUT | /api/chat/usage-config | 取得／更新新手每日訊息額度（Admin） | ✅ |
| | GET | /api/chat/conversations | 取得對話列表 | ✅ |
| | GET | /api/chat/conversations/{id} | 取得對話詳情 | ✅ |
| | POST | /api/chat/messages/{messageId}/feedback | 回覆品質回饋 | ✅ |
| **提示詞** | GET | /api/prompts | 取得提示詞列表 | ✅ |
| | POST | /api/prompts | 建立提示詞 | ✅ |
| | PUT | /api/prompts/{id} | 更新提示詞 | ✅ |
| | DELETE | /api/prompts/{id} | 刪除停用提示詞 | ✅ |
| | POST | /api/prompts/{id}/test | 測試提示詞 | ✅ |
| **上傳** | POST | /api/v1/upload | 上傳檔案 | ✅ |
| | GET | /api/v1/upload/{id} | 取得檔案資訊 | ✅ |
| | GET | /api/v1/files | 列出所有上傳檔案（支援 pipeline_status 篩選） | ✅ |
| | GET | /api/v1/files/{id}/deletion-impact | 取得待審核來源刪除影響與阻擋原因（Admin） | ✅ |
| | DELETE | /api/v1/files/{id} | 交易式邏輯刪除待審核來源資料（Admin） | ✅ |
| **清洗** | POST | /api/v1/clean | 啟動清洗任務 | ✅ |
| | POST | /api/v1/clean/preview | 預覽清洗結果 | ✅ |
| | GET | /api/v1/clean/{id}/result | 取得清洗結果 | ✅ |
| **任務** | GET | /api/v1/tasks | 列出所有任務 | ✅ |
| | GET | /api/v1/tasks/{id} | 取得任務狀態 | ✅ |
| | DELETE | /api/v1/tasks/{id} | 取消任務 | ✅ |
| ~~**規則**~~ | | | 已移除 — 改為固定規則模式（`get_default_rules()`），migration 005 已刪除相關資料表 | ❌ removed |
| **下載** | GET | /api/v1/download/{task_id} | 下載所有結果 | ✅ |
| | GET | /api/v1/download/{task_id}/{file_id} | 下載單一檔案 | ✅ |
| | GET | /api/v1/download/{task_id}/report | 下載清洗報告 | ✅ |
| **稽核** | GET | /api/audit-logs | 查詢稽核日誌 | ✅ |
| **審核** | GET | /api/v1/review/{task_id} | 取得任務審核資訊 | ✅ |
| | GET | /api/v1/review/{task_id}/files/{file_id}/content | 取得檔案內容 | ✅ |
| | PUT | /api/v1/review/{task_id}/files/{file_id}/content | 更新編輯內容 | ✅ |
| | PUT | /api/v1/review/{task_id}/files/{file_id}/tags | 更新標籤 | ✅ |
| | PUT | /api/v1/review/{task_id}/files/{file_id}/status | 審核狀態 | ✅ |
| | POST | /api/v1/review/{task_id}/approve | 批准任務 | ✅ |
| | POST | /api/v1/review/{task_id}/ingest | 送入 RAG 知識庫 | ✅ |
| **分析** | GET | /api/v1/analytics/cleaning | 清洗統計 | ✅ |
| | GET | /api/v1/analytics/knowledge-base | 知識庫統計 | ✅ |
| | GET | /api/v1/analytics/timeline | 時間軸統計 | ✅ |
| | GET | /api/v1/analytics/pipeline | 資料管線統計（漏斗各階段計數） | ✅ |
| | GET | /api/v1/analytics/recent-activity | 最近操作記錄（Timeline） | ✅ |
| **知識庫瀏覽** | GET | /api/v1/knowledge-base/documents | 列出知識庫文件 | ✅ |
| | GET | /api/v1/knowledge-base/documents/grouped | 按文件聚合瀏覽（以文件為核心） | ✅ |
| | GET | /api/v1/knowledge-base/documents/by-source | 按來源查看 chunks | ✅ |
| | GET | /api/v1/knowledge-base/sources | 列出所有來源 | ✅ |
| | GET | /api/v1/knowledge-base/search | 全文搜尋知識庫 | ✅ |
| | DELETE | /api/v1/knowledge-base/documents/by-source | 按來源刪除 | ✅ |
| **角色** 🔮 | GET | /api/v1/roles | 列出角色 | 🔮 |
| | POST | /api/v1/roles | 建立自訂角色 | 🔮 |
| **會員** 🔮 | POST | /api/v1/tenants | 建立會員 | 🔮 |
| | GET | /api/v1/tenants/{id}/usage | 取得資源使用量 | 🔮 |
| **知識庫** 🔮 | POST | /api/v1/knowledge-bases | 建立知識庫 | 🔮 |
| | POST | /api/v1/knowledge-bases/{id}/documents | 上傳文件 | 🔮 |
| **分析（進階）** 🔮 | GET | /api/v1/analytics/overview | 取得概覽數據 | 🔮 |
| | GET | /api/v1/analytics/compliance | 取得合規分析 | 🔮 |
| **Webhook** 🔮 | POST | /api/v1/webhooks | 建立 Webhook | 🔮 |
| **監控** 🔮 | GET | /api/v1/monitoring/metrics | 取得監控指標 | 🔮 |
| | POST | /api/v1/monitoring/alerts | 建立告警規則 | 🔮 |
| **WebSocket** | WS | /ws/tasks/{task_id} | 訂閱特定任務 | ✅ |
| | WS | /ws/all | 訂閱所有任務 | ✅ |

### 8.2 通用回應格式

```json
// 成功回應
{
  "success": true,
  "data": { ... },
  "message": "操作成功",
  "timestamp": "2026-02-05T10:30:00.000Z"
}

// 錯誤回應
{
  "success": false,
  "error": {
    "code": "ERROR_CODE",
    "message": "錯誤訊息描述"
  },
  "timestamp": "2026-02-05T10:30:00.000Z"
}
```

### 8.3 錯誤代碼定義

| 錯誤代碼 | HTTP 狀態碼 | 說明 |
|----------|------------|------|
| VALIDATION_ERROR | 400 | 請求參數驗證失敗 |
| INVALID_CREDENTIALS | 401 | 帳號或密碼錯誤 |
| UNAUTHORIZED | 401 | 未授權存取 |
| TOKEN_EXPIRED | 401 | Token 已過期 |
| ACCOUNT_LOCKED | 423 | 帳號已鎖定 |
| FORBIDDEN | 403 | 權限不足 |
| NOT_FOUND | 404 | 資源不存在 |
| INVALID_FILE_TYPE | 400 | 不支援的檔案格式 |
| FILE_TOO_LARGE | 413 | 檔案大小超過限制 |
| TASK_NOT_CANCELLABLE | 400 | 任務無法取消 |
| RULE_IN_USE | 400 | 規則使用中無法刪除 |
| FILES_NOT_REVIEWED | 400 | 尚有檔案未完成審核 |
| TASK_NOT_APPROVED | 400 | 任務尚未批准 |
| TASK_ALREADY_INGESTED | 400 | 任務已匯入知識庫 |
| INTERNAL_ERROR | 500 | 伺服器內部錯誤 |

---

## 9. 介面設計規格

### 9.1 前台 (Chatbot UI)

```
┌─────────────────────────────────────────────────────────────┐
│  Header: Logo | 對話標題 | 回應層級（新手/一般/顧問）| 使用者選單 │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────┐   │
│  │                                                      │   │
│  │  對話歷史區域                                         │   │
│  │  - 使用者訊息 (靠右)                                  │   │
│  │  - 助手回應 (靠左，含引用來源)                         │   │
│  │                                                      │   │
│  └─────────────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────┐ ┌─────────┐   │
│  │  輸入訊息...                             │ │  送出   │   │
│  └─────────────────────────────────────────┘ └─────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### 9.2 後台 (Admin Dashboard)

```
┌─────────────────────────────────────────────────────────────┐
│  Sidebar │ Header: 歡迎訊息 | 通知 | 使用者選單               │
│          ├─────────────────────────────────────────────────┤
│  - Home  │                                                  │
│  - Rules │  主要內容區域                                     │
│  - Tasks │                                                  │
│  ─────── │  - Home: 檔案上傳、規則選擇、任務監控              │
│  KB 🔮   │  - Rules: 規則 CRUD、匯入/匯出                    │
│  分析 🔮 │  - Tasks: 任務列表、進度追蹤、下載                 │
│  ─────── │  - KB 🔮: 知識庫管理                              │
│  Users🔄 │  - 分析 🔮: 儀表板                                │
│  Roles🔮 │  - Users 🔄: 使用者管理                           │
│  Webhook │  - Roles 🔮: 角色管理                             │
│  Audit   │  - Audit: 稽核日誌查詢                            │
│  ─────── │                                                  │
│  設定 🔄 │                                                  │
│  監控 🔮 │                                                  │
└──────────┴──────────────────────────────────────────────────┘
```

### 9.3 清洗管理介面 (Cleaner App)

```
┌─────────────────────────────────────────────────────────────────┐
│  Sidebar │ Header: 歡迎訊息 | 通知 | 使用者選單                    │
│          ├─────────────────────────────────────────────────────┤
│          │                                                      │
│  總覽    │  ■ 總覽 (Dashboard)                                  │
│  ─────── │    - 資料管線漏斗（5 階段）:                          │
│  來源資料│      上傳 → 清洗 → 審核 → 批准 → 入庫                 │
│  任務列表│    - 雙欄統計: 清洗效能（完成率/偵測數/處理時間）       │
│  知識庫  │                + 知識庫概況（文件數/向量數/覆蓋度）     │
│  ─────── │    - 最近活動 Timeline（操作類型 + 時間戳）            │
│          │                                                      │
│          │  ■ 來源資料 (Files)                                   │
│          │    - 檔案列表 + 檔名/類型/審核狀態篩選                 │
│          │    - 審核狀態欄:                                      │
│          │      未處理/清洗中/清洗失敗/未送審/待審核/已批准/已入庫/已退回 │
│          │    - Admin 可預檢並邏輯刪除未送審或待審核資料；其餘角色不顯示操作 │
│          │                                                      │
│          │  ■ 知識庫 (Knowledge Base) — 三 Tab                   │
│          │    Tab 1: 文件管理 — 以「文件」為核心聚合瀏覽          │
│          │    Tab 2: 語意搜尋 — RAG 檢索品質測試                  │
│          │    Tab 3: 來源分析 — 來源/標籤分佈統計                 │
│          │                                                      │
│          │  ■ 檔案審核 (Review Detail)                           │
│          │    - 左側檔案導航面板 + 60vh 內容區                   │
│          │    - 閱讀/編輯 Toggle + PII 摘要壓縮                  │
│          │    - 審核操作: 標籤/批准/退回/送入 RAG                 │
│          │                                                      │
└──────────┴──────────────────────────────────────────────────────┘
```

---

## 10. 安全需求

### 10.1 身分驗證機制

| 項目 | 規格 |
|------|------|
| 驗證方式 | JWT (JSON Web Token) |
| Access Token 效期 | 60 分鐘 |
| Refresh Token 效期 | 7 天 |
| 工作階段上限 | 每帳號 10 個有效工作階段；超出時撤銷最久未使用者 |
| Refresh 輪替 | `sid` 維持不變、`jti` 單次使用，以條件更新原子輪替 |
| Token 用途 | 新發 Token 具有 `tokenUse=access` 或 `tokenUse=refresh`；JWT Strategy 拒絕 Refresh Token 存取受保護端點 |
| 前端失效判定 | 僅 Refresh 400／401 或缺少 Refresh Token 要求同頁重新登入；網路／429／5xx 保留狀態 |
| 跨分頁協調 | 優先使用 Web Locks；fallback 採 Bakery-style 多鍵競爭者互斥，租約每 2 秒續期、5 秒到期，選出競爭者後須再次檢查舊版單鍵租約；執行前寫入 lease 並讀回確認 owner，失敗時移除 contender 後重新競爭；登入、重新驗證、SSO、密碼變更、登出與 Refresh 共用同一鎖；Refresh 帶認證協定版本 2，API 以 426 阻擋舊頁面輪替；HTTP 回應及 JSON 解析後均重驗 Access／Refresh／使用者快照，帳號主體不同時不清除或覆寫 Token |
| 同頁重新登入 | 唯讀顯示原帳號，只接受相同使用者 ID；不同使用者 ID 時拒絕切換並撤銷該次新工作階段；稍後處理後保留手動重新登入入口 |
| Token 演算法 | HS256 |
| 密碼雜湊 | bcrypt (cost factor 12) |
| 登入失敗鎖定 | 5 次失敗後鎖定 15 分鐘 |

### 10.2 資料加密標準

| 項目 | 標準 |
|------|------|
| 傳輸加密 | TLS 1.3 |
| 儲存加密 | AES-256 (敏感資料) |
| 密碼儲存 | bcrypt |
| API 金鑰 | 單向雜湊 + 安全比對 |

### 10.3 個資保護措施

| 措施 | 說明 |
|------|------|
| 資料最小化 | 僅收集必要資料 |
| 存取控制 | RBAC 權限管理 |
| 稽核追蹤 | 所有操作記錄 |
| 資料保留 | 依法規保留期限 |
| 匿名化處理 | 支援多種去識別化策略 |

---

## 11. 驗收標準

### 11.1 功能驗收測試案例

| 測試案例 ID | 測試項目 | 測試步驟 | 預期結果 |
|------------|----------|----------|----------|
| TC-01-001 | 正常登入 | 輸入正確帳密並點擊登入 | 登入成功，跳轉至首頁 |
| TC-01-002 | 登入失敗 | 輸入錯誤密碼並點擊登入 | 顯示錯誤訊息，保留在登入頁 |
| TC-02-001 | RAG 查詢 | 輸入資安問題並送出 | 回應時間 < 10 秒，顯示引用來源 |
| TC-02-002 | 重新產出最新回覆 | 完成一輪資安問答後點擊最新回覆的重新產出 icon；另測修改問題、雙 attempt 並發、processing 重送、failed 重試與落盤後斷線 | 原問答保留；底部追加相同問題與新回覆；舊回覆不再顯示 icon；修改問題與並發 loser 被拒絕；processing 重送回 409；failed／已完成的同 attempt 重試不重複扣額度 |
| TC-03-001 | 檔案上傳 | 拖放 PDF 檔案 | 顯示上傳進度，成功後列表顯示 |
| TC-04-001 | 清洗任務 | 選擇檔案與規則，開始清洗 | 任務建立成功，顯示即時進度 |
| TC-05-001 | 清洗審核 | 查看清洗任務，檢視檔案內容 | 顯示清洗前後對比，標示 PII 實體 |
| TC-05-002 | 內容編輯 | 手動修正清洗結果 | 儲存成功，顯示更新後內容 |
| TC-05-003 | 標籤管理 | 為檔案新增分類標籤 | 標籤儲存成功 |
| TC-05-004 | 任務批准 | 審核完成後批准任務 | 任務狀態更新為 approved |
| TC-05-005 | 送入 RAG | Admin 將批准任務送入知識庫 | 文件向量化成功，顯示結果 |
| TC-05-006 | Maker-Checker 送審 | data_cleaner 登入 → 完成清洗任務 → 送審 → 確認 submitted_by 記錄 | 任務狀態變更為 review_requested，submitted_by = cleaner ID |
| TC-05-007 | Maker-Checker 退回 | data_reviewer 登入 → 開啟已送審任務 → 退回（附理由）→ 確認狀態 | 任務狀態變更為 `rejected`（保留審核軌跡），檔案解除凍結，記錄 rejection_reason 與 review_note；cleaner 可重新編輯並再送審（rejected → review_requested） |
| TC-06-001 | 清洗統計 | 查看 Analytics 清洗統計頁面 | 顯示任務完成率、PII 分佈 |
| TC-06-002 | 時間軸統計 | 查看時間軸趨勢圖表 | 圖表正確顯示每日處理量 |
| TC-20-001 | 刪除與查閱待審核來源資料 | 以 Admin 開啟待審核資料刪除預檢，確認影響、填寫原因並輸入「刪除」；另測非 Admin、已批准／已入庫／處理中、Qdrant 有切塊與 Qdrant 不可用，再切換「已刪除」篩選 | 只有完全符合資格的資料完成邏輯刪除；預設列表排除且「已刪除」篩選可查閱；有任務時只可查看歷史；原檔、清洗產物與歷史仍存在；所有阻擋情境不產生狀態變更 |
| TC-20-002 | Cleaner 台北時間 | 以 UTC、無 offset UTC 與跨日時間載入來源、任務、審核及知識庫頁面 | 各頁固定顯示 `Asia/Taipei` 時間，結果不受執行測試或瀏覽器所在時區影響 |
| TC-20-003 | 來源審核狀態與分頁 | 在真實 PostgreSQL 交易內建立 25 筆同狀態來源，查詢第 1／2 頁；另驗證 SQL 狀態矩陣、逐檔未通過與同時間多任務 | 兩頁 `total=25`、資料不重複且只含指定狀態；逐檔結果不覆寫來源的「未送審」；同時間任務以 `id DESC` 決勝；交易 rollback 後無 fixture 殘留 |

### 11.2 效能驗收指標

| 測試項目 | 目標值 | 測試方法 |
|----------|--------|----------|
| RAG 查詢回應時間 | P95 < 10 秒 | 壓力測試 1000 次查詢 |
| 檔案上傳速度 | 10MB/秒 | 上傳 50MB 檔案 |
| 清洗處理速度 | > 10 檔案/分鐘 | 批次處理 100 個 1MB 檔案 |
| 並發使用者 | 支援 100 同時在線 | 並發連線測試 |

### 11.3 安全性驗收檢核表

| 檢核項目 | 通過標準 |
|----------|----------|
| SQL Injection | 所有輸入已參數化 |
| XSS | 所有輸出已轉義 |
| CSRF | Token 驗證已實作 |
| 認證繞過 | 所有 API 已驗證 |
| 密碼強度 | 最少 8 字元、含數字與字母 |
| Session 管理 | Token 過期機制正常 |
| 檔案上傳 | 類型與大小限制正常 |

---

## 12. 附錄

### 12.1 術語表

| 術語 | 全稱 | 說明 |
|------|------|------|
| RAG | Retrieval-Augmented Generation | 檢索增強生成 |
| PII | Personally Identifiable Information | 個人可識別資訊 |
| NER | Named Entity Recognition | 命名實體識別 |
| LLM | Large Language Model | 大型語言模型 |
| JWT | JSON Web Token | 用於身分驗證的開放標準 |
| RBAC | Role-Based Access Control | 基於角色的存取控制 |
| SSE | Server-Sent Events | 伺服器推送事件 |
| SLA | Service Level Agreement | 服務水準協議 |
| RTO | Recovery Time Objective | 恢復時間目標 |
| RPO | Recovery Point Objective | 恢復點目標 |
| Cleaner App | - | 獨立的清洗審核管理前端應用 (Port 5503) |
| TanStack React Query | - | React 伺服器狀態管理與快取函式庫 |

### 12.2 版本規劃藍圖

```
Phase 1 (已完成)      Phase 2 (規劃中)      Phase 3 (願景)
─────────────────    ─────────────────    ─────────────────
✅ RAG 查詢          🔄 使用者認證        🔮 多會員架構
✅ 資料去識別化      🔄 提示詞管理        🔮 進階 RBAC
✅ 規則管理          🔄 角色權限          🔮 多知識庫
✅ 任務管理          🔄 三層式回應        🔮 進階分析儀表板
✅ 即時通知                              🔮 第三方整合
✅ 稽核日誌                              🔮 運維監控
✅ 清洗審核管理
✅ 資料分析儀表板
✅ Cleaner App

─────────────────────────────────────────────────────────────
|         |         |         |         |         |
Q1 2026   Q2 2026   Q3 2026   Q4 2026   Q1 2027   Q2 2027
```

### 12.3 技術債務追蹤

| 項目 | 說明 | 優先級 | 計畫解決時間 |
|------|------|--------|--------------|
| 快取層 | 加入 Redis 快取提升效能 | P2 | Phase 2 |
| 非同步處理 | 清洗任務改為真正的非同步 | P1 | Phase 2 |
| API 版本控制 | 實作 API 版本機制 | P2 | Phase 2 |
| 測試覆蓋率 | 提升至 80% 以上 | P2 | Phase 2 |
| 文件自動化 | OpenAPI 文件自動生成 | P3 | Phase 2 |

---

## 文件版本歷程

| 版本 | 日期 | 變更說明 |
|------|------|----------|
| 2.8.0 | 2026-07-26 | 議題分類改為初判；新增全題 RAG 優先、嚴格證據恢復、縮寫限定網搜推測、單輪原子銜接、RAG 不可用分流、資安建議題驗證與網頁來源安全邊界 |
| 2.7.3 | 2026-07-18 | 來源列表增加 `latest_task_name` 契約、舊任務 ID 備援、RWD 欄位與最新任務名稱／狀態／連結一致性驗證 |
| 2.7.2 | 2026-07-17 | 補強來源資料狀態層級、未送審、先篩選後計數／穩定分頁、HTTP 422 與 PostgreSQL rollback 整合驗證 |
| 2.7.1 | 2026-07-16 | 補強已刪除專用篩選、任務歷史入口、操作欄語意與 Cleaner 共用台北時區顯示契約 |
| 2.7.0 | 2026-07-16 | 新增 US-20-02：管理員限定的待審核來源預檢與邏輯刪除、全關聯與 Qdrant 失敗關閉、交易鎖、任務統計重算及歷史保留 |
| 2.6.5 | 2026-07-14 | 重新產出新增 metadata lease CAS、後端主題限制與真實額度回滾驗證 |
| 2.6.4 | 2026-07-14 | 重新產出額度與 claim 改為同交易，失敗釋放加入 lease fencing |
| 2.6.3 | 2026-07-14 | 重新產出加入 5 分鐘 lease、lease fencing、跨分頁 attempt 定位與 completed claim 狀態 |
| 2.6.2 | 2026-07-14 | 重新產出加入 processing／failed claim 狀態、處理中 409 防重入與中斷後安全續跑 |
| 2.6.1 | 2026-07-14 | 強化重新產出：相同問題驗證、conversation row lock、attempt 冪等、落盤後回放、單調訊息時間與重試鎖 |
| 2.6.0 | 2026-07-14 | 新增 US-02-07 最新回覆重新產出、`regenerateFromMessageId` 驗證、訊息 metadata 關聯、額度與 UI 狀態契約 |
| 2.5.0 | 2026-07-13 | Admin、Cleaner、Chatbot 共用的 Access Token 效期由 30 分鐘延長為 60 分鐘；Refresh Token 維持 7 天 |
| 2.3.0 | 2026-07-13 | 認證工作階段穩定化：Prisma 0009 `auth_sessions`、30 分鐘 Access Token、Refresh 原子輪替、SSO 交換、目前／全部登出與三前端同頁復原 |
| 2.4.0 | 2026-07-13 | 補 Token 用途隔離、同帳號重新登入、跨分頁租約續期與不同帳號切換保護 |
| 2.2.2 | 2026-07-13 | 管理者對話詳情回傳使用者摘要，修正 Drawer 使用者欄位固定顯示「-」 |
| 2.2.1 | 2026-07-13 | 對話列表新增實際訊息總數；計數限定 user/assistant，並以 Prisma 0008 建立 messages(conversation_id, created_at) 複合索引 |
| 2.2.0 | 2026-07-12 | RFC-003：任務名稱必填、Admin 列表快速改名、Alembic 012 與 Asia/Taipei 時間契約 |
| 2.1.2 | 2026-07-12 | 上傳失敗改為持續顯示中文原因；補上格式、單檔／整批 50 MB、登入失效、ZIP 完全失敗／部分成功流程，並將知識庫來源與入庫活動訊息中文化 |
| 2.1.1 | 2026-07-12 | 補強 RFC-002 分類鎖定與競態規則；下載改為 Bearer fetch → Blob，並區分清洗產物與 Cleaner 審核 |
| 2.1.0 | 2026-07-12 | RFC-002：新增 FR-24 法規／知識分類契約、Clean API 防線、治理集合與 Alembic 011 |
| 1.8.0 | 2026-03-16 | ML-15 全面審查：PII 實體統一為 20 種（含 TW_ADDRESS）；API 路徑修正（Auth/Chat/Prompts 移除 /v1/）；partial_mask 參數名修正（keep_first/keep_last）；UserRole enum 補齊 7 角色；FR-08 Rules API 標記 deprecated；Ingest 權限修正（data_reviewer/admin）；審核狀態機 reviewing→review_requested；新增 Maker-Checker 驗收案例（TC-05-006/007）；新增回饋 API 規格；新增信心度判定邏輯；新增 it_user 專屬提示詞說明 |
| 1.7.0 | 2026-03-11 | 新增 basic_user 角色（beginner only）與 data_reviewer 角色（Maker-Checker 審批者）；上傳格式新增 MD、JSON、HTML、ZIP；新增 ZIP 上傳 API 規格（安全限制：壓縮比≤20:1、檔案數≤100、總大小≤200MB、禁止路徑穿越）；新增 Maker-Checker 職責分離欄位（submitted_by/submitted_at）與四端點驗證規則（approve、reject、file_status、ingest）；更新三層模式角色對應表（7 角色） |
| 1.6.0 | 2026-02-13 | 移除獨立 Cleaning Service :8001（已合併至 RAG Service :3502）；新增 5 支 API（analytics/pipeline、analytics/recent-activity、knowledge-base/documents/grouped、knowledge-base/documents/by-source、files pipeline_status 篩選）；重寫 Cleaner App 介面規格（管線漏斗、三 Tab 知識庫、檔案審核面板）；更新服務通訊矩陣與連線埠分配 |
| 1.5.0 | 2026-02-12 | 新增 FR-20 來源資料瀏覽（GET /api/v1/files）；新增 FR-21 知識庫文件瀏覽（4 支 API）；更新功能權限矩陣；新增 US-18-07~09 使用者故事；更新 API 端點清單；更新介面設計規格（4 個側邊欄項目） |
| 1.4.0 | 2026-02-11 | 新增 Cleaner App 獨立前端 (Port 5503)；新增 data_cleaner 角色與應用程式存取矩陣；新增 FR-18 清洗審核管理（含 7 支 Review API）；新增 FR-19 資料分析儀表板（含 3 支 Analytics API）；擴展 task_files/tasks 資料模型（審核欄位）；更新系統架構圖、連線埠分配、API 端點清單、ER 圖、資料字典、介面設計規格 |
| 1.3.0 | 2026-02-09 | 對齊計畫書：目標市場改為中小企業；新增三層式AI顧問模式(FR-16)；新增 IT User 角色；新增在地化法規知識庫(FR-17)；新增 AI 品質需求(NFR-06)；修復 header 版本號 |
| 1.2.0 | 2026-02-09 | 「租戶」→「會員」；移除 Tenant Admin 角色（功能併入 Platform Admin）；效能指標調整為 P95 < 10/7 秒 |
| 1.1.0 | 2026-02-09 | 修正角色定位：去識別化功能限 Admin；Consultant 改為對話導向；術語「脫敏」→「去識別化」 |
| 1.0.0 | 2026-02-05 | 初始版本（整合 SRS_CURRENT + SRS_VISION） |

---

> **文件結束**
>
> 本文件為 ODA Cyber Konsult 資安助手 RAG 系統的完整技術 SRS。
> 如需查看商業需求概覽，請參閱 `SRS_BUSINESS.md`。
