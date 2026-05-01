# 規格文件（Outer Loop / Tier 1 SSoT）

> ODA Cyber Konsult — 需求、領域模型、架構、安全、API 與追溯矩陣。

本目錄是 AI-Native SDLC 的 **Outer Loop** 與 Three-Tier SSoT 的 **Tier 1**。此層回答「系統做什麼、為什麼做、可如何驗收」，下游設計、程式、測試與運維文件都應回指此層。

## 文件鏈

```text
PRD → SRS_BUSINESS/SRS_TECHNICAL → domain-glossary/context-map
    → architecture/ARCH.md → threat-model.md → RTM.md
    → api/ + implementation/ + downstream testing/operations
```

## 核心文件

| 文件 | Audience | 說明 | 狀態 |
|------|----------|------|------|
| [`PRD.md`](./PRD.md) | human-primary | User Story、Acceptance Criteria、優先級、角色矩陣 | Active SSoT |
| [`SRS_BUSINESS.md`](./SRS_BUSINESS.md) | human-primary | 商業需求、使用情境、KPI、風險假設 | Active SSoT |
| [`SRS_TECHNICAL.md`](./SRS_TECHNICAL.md) | AI + dev | FR/NFR、資料模型、API 總覽、驗收標準 | Active SSoT |
| [`domain-glossary.md`](./domain-glossary.md) | human + AI | 通用語言、核心術語、避免同義詞漂移 | Active SSoT |
| [`context-map.md`](./context-map.md) | human + AI | Bounded Context、上下游關係、ACL 與 Shared Kernel | Active SSoT |
| [`architecture/ARCH.md`](./architecture/ARCH.md) | human + AI | ADR、系統級架構決策與後果 | Active SSoT |
| [`threat-model.md`](./threat-model.md) | human + AI | STRIDE 威脅模型、攻擊面、緩解措施 | Active SSoT |
| [`RTM.md`](./RTM.md) | AI + QA | US → FR → API → Module → Test → Security 追溯 | Active SSoT |

## 子目錄

| 子目錄 | 說明 |
|--------|------|
| [`api/`](./api/) | API 規格與端點索引 |
| [`architecture/`](./architecture/) | 架構圖、ADR、RAG/清洗子系統設計 |
| [`implementation/`](./implementation/) | 既有實作說明；不作為新需求 SSoT |
| [`guides/`](./guides/) | 操作導引與開發 cookbook |

## 追溯與變更規則

- 新增/修改 User Story 或 AC：先改 `PRD.md`，再同步 `SRS_TECHNICAL.md` 與 `RTM.md`。
- 新增跨上下文或跨服務規則：同步檢查 `domain-glossary.md`、`context-map.md`、`threat-model.md`。
- 新增高風險功能工作記憶：只在觸發條件成立時建立 `../04-features/{feature}/`，並在 `RTM.md` 對應 FR 區塊加 spec pack 指針。
- 不在 Tier 1 複製 `contracts/*.yml` 的完整內容；只引用其用途與值來源。
- 自動驗證：Tier 2A `contracts/*.yml` 與 Tier 2B `04-features/*` 變更後跑 `pnpm lint:spec-pack`（三件套完整性 / D0 紅線 / D5 gate；實作見 [`../../scripts/spec-pack-lint.mjs`](../../scripts/spec-pack-lint.mjs)）。
