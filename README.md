# ODA Cyber Konsult - 文件導覽

三層式 AI 資安顧問系統（新手/一般/顧問模式），基於 RAG 技術搭配在地化法規知識庫，服務中小企業。

本文件庫依 **AI-Native SDLC** 組織：Outer Loop 規格、Middle Loop 設計與測試、Delivery Loop 運維交付，並補上 Three-Tier SSoT 的 contracts 與 feature spec pack。

## AI-Native SDLC 文件地圖

```text
PRD/SRS → domain glossary/context map → ARCH → threat-model → RTM
   ↓                 ↓                    ↓         ↓          ↓
01-specs/        01-specs/            02-design/ 02-testing/ 03-operations/
   ↓                                      ↓
contracts/ + 04-features/{feature}/  ← feature-specific working memory
```

## 目錄結構

| 目錄 | SDLC 層級 | Audience | 內容 |
|------|-----------|----------|------|
| [`01-specs/`](./01-specs/) | Outer Loop / Tier 1 | human + AI | PRD、SRS、Domain Glossary、Context Map、ARCH、RTM、Threat Model、API 規格 |
| [`02-design/`](./02-design/) | Middle Loop design | human-primary | diagrams、flows、wireframes、prototype 索引；既有架構圖仍指向 `01-specs/architecture/` |
| [`02-testing/`](./02-testing/) | Middle Loop QG-4 | human + AI | 測試策略、測試指南、API E2E、NFR baseline |
| [`03-operations/`](./03-operations/) | Delivery Loop QG-5 | human + AI | runbook、deployment、backup/DR、QG-5 readiness、staging setup |
| [`04-features/`](./04-features/) | Tier 2B Spec Pack | AI-primary | 高風險功能的 requirements/design/tasks 三件套 |
| [`../contracts/`](../contracts/) | Tier 2A SSoT | runtime + AI | 跨 TS/Python 的常數與列舉 SSoT |

## 快速導引

| 我想要... | 前往 |
|----------|------|
| 查產品需求與 AC | [`01-specs/PRD.md`](./01-specs/PRD.md) |
| 查技術需求與資料/API 規格 | [`01-specs/SRS_TECHNICAL.md`](./01-specs/SRS_TECHNICAL.md) |
| 查領域術語與上下文邊界 | [`01-specs/domain-glossary.md`](./01-specs/domain-glossary.md) → [`01-specs/context-map.md`](./01-specs/context-map.md) |
| 查系統架構與 ADR | [`01-specs/architecture/`](./01-specs/architecture/) |
| 查需求到測試追溯 | [`01-specs/RTM.md`](./01-specs/RTM.md) |
| 查設計圖與流程圖索引 | [`02-design/`](./02-design/) |
| 執行測試 | [`02-testing/test-guide.md`](./02-testing/test-guide.md) |
| 檢查 QG-5 上線準備 | [`03-operations/qg5-readiness.md`](./03-operations/qg5-readiness.md) |
| 查高風險功能工作記憶 | [`04-features/chat-rag/`](./04-features/chat-rag/) |
| 查跨語言常數來源 | [`../contracts/README.md`](../contracts/README.md) |

## 文件維護規則

- Tier 1 文件回答「系統做什麼」：PRD、SRS、ARCH、RTM、Threat Model、Domain Glossary、Context Map。
- Tier 2A 文件回答「跨功能/跨語言共用什麼」：`contracts/*.yml` 與相關 loader。
- Tier 2B 文件回答「修改特定高風險功能時必須記得什麼」：`docs/04-features/{feature}/`。
- 每個 `docs/` 子目錄需有 `README.md`，用索引方式連到權威文件；避免複製 PRD/SRS/RTM 長段內容。
- 新增或修改 spec pack 後需執行 `pnpm lint:spec-pack`。
