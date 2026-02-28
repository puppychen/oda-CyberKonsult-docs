# 測試策略文件

> **ODA Cyber Konsult - 資安助手 RAG 系統**
>
> 文件版本：1.0.0
> 建立日期：2026-03-01
> 文件類型：測試策略（Test Strategy）

---

## 1. 測試目標

確保系統在功能正確性、效能、安全性三個維度達到 [SRS_TECHNICAL.md](../specs/SRS_TECHNICAL.md) 第 11 章定義的驗收標準，同時建立可持續的測試回歸機制。

---

## 2. 測試金字塔

```
            ┌─────────┐
            │  E2E    │  ~5%
            │ (整合)   │
           ┌┴─────────┴┐
           │ Integration│  ~25%
           │ (模組整合)  │
          ┌┴────────────┴┐
          │  Unit Tests   │  ≥70%
          │  (單元測試)    │
          └───────────────┘
```

| 層級 | 比例目標 | 範圍 | 執行頻率 |
|------|---------|------|---------|
| **Unit** | ≥70% | 個別函式、Service、Repository、Utility | 每次 commit |
| **Integration** | ~25% | 模組間互動、API 端點、資料庫操作 | 每次 PR |
| **E2E** | ~5% | 完整使用者流程、跨服務整合 | 每日 / Release 前 |

---

## 3. 工具鏈

### 3.1 NestJS（TypeScript）

| 工具 | 用途 | 設定檔 |
|------|------|--------|
| **Jest** | 測試框架 | `apps/api/jest.config.ts` |
| **supertest** | HTTP 端點測試 | E2E 測試使用 |
| **@nestjs/testing** | NestJS 模組測試工具 | 整合測試使用 |
| **Prisma Mock** | 資料庫層 Mock | In-memory Map 實作 |

### 3.2 Python（FastAPI）

| 工具 | 用途 | 設定檔 |
|------|------|--------|
| **pytest** | 測試框架 | `python/*/pyproject.toml` |
| **httpx** | 非同步 HTTP 測試 | FastAPI TestClient |
| **pytest-asyncio** | 非同步測試支援 | rag-service 使用 |
| **unittest.mock** | Mock 工具 | Qdrant/LLM Mock |

### 3.3 前端（React）

| 工具 | 用途 | 狀態 |
|------|------|------|
| **Vitest** | 測試框架（規劃） | Phase 3 |
| **React Testing Library** | 元件測試（規劃） | Phase 3 |
| **Playwright** | E2E 瀏覽器測試（規劃） | Phase 3 |

---

## 4. 當前測試涵蓋率

### 4.1 NestJS API

| 指標 | 數值 | 說明 |
|------|------|------|
| Test Suites | 43 | 涵蓋所有模組 |
| Test Cases | 445 | 全數通過 |
| 涵蓋模組 | auth, users, chat, prompts, audit, health, cleaning, datasources, websocket, llm, websearch | 完整模組覆蓋 |

**執行指令**：
```bash
# 單元測試
pnpm --filter @oda-cyber/api test

# E2E 測試
pnpm --filter @oda-cyber/api test:e2e
```

### 4.2 Python — rag-service

| 指標 | 數值 | 說明 |
|------|------|------|
| Test Files | 21+ | 涵蓋核心服務 |
| Internal Auth | 8 tests | X-Internal-Token 中介軟體 |
| 涵蓋範圍 | RAG retrieve, cleaning pipeline, file loaders, PII analyzers, anonymizers, API routes | 核心管線覆蓋 |

**執行指令**：
```bash
cd python/rag-service && uv run pytest
```

### 4.3 Python — data-pipeline

| 指標 | 數值 | 說明 |
|------|------|------|
| Test Files | 13 | 涵蓋各解析器與策略 |
| 涵蓋範圍 | PDF/DOCX/XLSX loaders, Presidio analyzers, 6 anonymization strategies, TW recognizers | 清洗管線覆蓋 |

**執行指令**：
```bash
cd python/data-pipeline && uv run pytest
```

### 4.4 前端

| 應用 | 測試覆蓋 | 狀態 |
|------|---------|------|
| Admin Dashboard | 0% | Phase 3 規劃 |
| Chatbot UI | 0% | Phase 3 規劃 |
| Cleaner App | 0% | Phase 3 規劃 |

---

## 5. 目標涵蓋率（Phase 3 目標）

| 層級 | 當前 | 目標 | 差距 |
|------|------|------|------|
| NestJS Unit | ~80% 模組覆蓋 | ≥80% 行覆蓋率 | 新增行覆蓋率量測 |
| Python Unit | ~70% | ≥80% | 擴充邊界測試 |
| 前端 Unit | 0% | ≥50% | 從 Cleaner App 開始 |
| E2E | 功能性 | 關鍵路徑 100% | 新增自動化 E2E |

---

## 6. 測試類型與策略

### 6.1 功能測試

對應 SRS_TECHNICAL.md §11.1 驗收測試案例：

| 測試案例 | 對應 FR | 說明 | 當前狀態 |
|----------|---------|------|---------|
| TC-01-001 | FR-01 | 正常登入 | ✅ Unit + E2E |
| TC-01-002 | FR-01 | 登入失敗 | ✅ Unit + E2E |
| TC-02-001 | FR-02 | RAG 查詢 | ✅ Unit（mock RAG） |
| TC-03-001 | FR-05 | 檔案上傳 | ✅ Unit |
| TC-04-001 | FR-04 | 清洗任務 | ✅ Unit + Python |
| TC-05-001 | FR-18 | 清洗審核 | ✅ Unit |
| TC-05-002 | FR-18 | 內容編輯 | ✅ Unit |
| TC-05-003 | FR-18 | 標籤管理 | ✅ Unit |
| TC-05-004 | FR-18 | 任務批准 | ✅ Unit |
| TC-05-005 | FR-18 | 送入 RAG | ✅ Unit |
| TC-06-001 | FR-19 | 清洗統計 | ✅ Python |
| TC-06-002 | FR-19 | 時間軸統計 | ✅ Python |

### 6.2 效能測試

對應 SRS_TECHNICAL.md §11.2 效能驗收指標：

| 測試項目 | 目標值 | 測試方法 | 工具 | 狀態 |
|----------|--------|----------|------|------|
| RAG 查詢回應時間 | P95 < 10s | 1000 次查詢壓力測試 | k6 / Artillery | 規劃中 |
| 檔案上傳速度 | 10MB/s | 50MB 檔案上傳 | supertest | 規劃中 |
| 清洗處理速度 | > 10 files/min | 100 個 1MB 檔案 | pytest | 規劃中 |
| 並發使用者 | 100 同時在線 | 並發連線測試 | k6 | 規劃中 |

### 6.3 安全性測試

對應 SRS_TECHNICAL.md §11.3 安全性驗收檢核表：

| 檢核項目 | 通過標準 | 測試方法 | 狀態 |
|----------|----------|----------|------|
| SQL Injection | 所有輸入已參數化 | Prisma + SQLAlchemy 自動參數化 | ✅ 通過 |
| XSS | 所有輸出已轉義 | React 自動轉義 + helmet headers | ✅ 通過 |
| 認證繞過 | 所有 API 已驗證 | JwtAuthGuard 全域啟用 | ✅ 通過 |
| 密碼強度 | 8+ 碼 + 複雜度 | IsStrongPassword validator | ✅ 通過 |
| Session 管理 | Token 過期機制 | JWT expiry + refresh token | ✅ 通過 |
| 檔案上傳 | 類型與大小限制 | 50MB limit + MIME 檢查 | ✅ 通過 |
| Rate Limiting | 60 req/min | ThrottlerModule | ✅ 通過 |
| 內部 API 認證 | X-Internal-Token | NestJS↔FastAPI 認證 | ✅ 通過 |

---

## 7. Mock 策略

### 7.1 NestJS 測試 Mock

| 依賴 | Mock 方式 | 說明 |
|------|----------|------|
| PrismaService | In-memory Map | 完整 CRUD + Transaction |
| RAG Service | HTTP Mock | 固定測試檢索結果 |
| LLM Provider | Mock Provider | 固定回應內容 |
| SearXNG | HTTP Mock | 固定搜尋結果 |

### 7.2 Python 測試 Mock

| 依賴 | Mock 方式 | 說明 |
|------|----------|------|
| Qdrant Client | AsyncMock | 模擬向量操作 |
| PostgreSQL | SQLite in-memory | 輕量資料庫 |
| Gemini/OpenAI | unittest.mock | 固定 embedding |

---

## 8. CI/CD 測試流程（規劃）

```
Git Push → Lint → Unit Tests → Integration Tests → Build → E2E Tests → Deploy
                                                                          │
                                                              ┌───────────┤
                                                              │           │
                                                           Staging    Production
                                                         (自動部署)   (手動審核)
```

| 階段 | 觸發條件 | 失敗處理 |
|------|---------|---------|
| Lint | 每次 push | 阻擋合併 |
| Unit Tests | 每次 push | 阻擋合併 |
| Integration | PR 建立/更新 | 阻擋合併 |
| E2E | Release branch | 阻擋 Release |

---

## 9. 測試資料管理

### 9.1 種子資料

定義於 `apps/api/prisma/seed.ts`：
- 4 個預設帳號（admin / cleaner / consultant / user）
- 6 個提示詞範本（3 角色 × 2 模式）
- 3 個規則設定檔（醫療/學術/合規）
- Demo 對話記錄

### 9.2 測試隔離原則

1. 每個測試案例獨立，不依賴執行順序
2. Mock 層確保不影響真實資料庫
3. E2E 測試使用獨立測試資料庫（規劃中）

---

## 10. 相關文件

| 文件 | 說明 | 路徑 |
|------|------|------|
| 測試指南 | 系統啟動與手動測試案例 | [testing/test-guide.md](./test-guide.md) |
| E2E 測試說明 | E2E 測試架構與 Mock 策略 | [testing/api-e2e-testing.md](./api-e2e-testing.md) |
| 驗收標準 | 功能/效能/安全驗收 | [specs/SRS_TECHNICAL.md §11](../specs/SRS_TECHNICAL.md) |
| PRD | User Story 與 AC | [specs/PRD.md](../specs/PRD.md) |

---

> **文件結束**
>
> 本文件為 ODA Cyber Konsult 的正式測試策略。
> 手動測試案例請參閱 `test-guide.md`，E2E 實作細節請參閱 `api-e2e-testing.md`。
