---
audience: human-primary
purpose: reference
status: approved
owner: ODA Cyber Konsult
---

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
| [`PRD.md`](./PRD.md) | human-primary | User Story、驗收條件、角色、來源審核狀態／穩定分頁、邏輯刪除與台北時間體驗 | Active SSoT |
| [`SRS_BUSINESS.md`](./SRS_BUSINESS.md) | human-primary | 商業需求、使用情境、KPI、來源審核狀態與資料治理 | Active SSoT |
| [`SRS_TECHNICAL.md`](./SRS_TECHNICAL.md) | AI + dev | FR/NFR、資料模型、API、來源狀態／穩定分頁與時區契約 | Active SSoT |
| [`domain-glossary.md`](./domain-glossary.md) | human + AI | 通用語言、核心術語、避免同義詞漂移 | Active SSoT |
| [`context-map.md`](./context-map.md) | human + AI | Bounded Context、上下游關係、ACL 與 Shared Kernel | Active SSoT |
| [`architecture/ARCH.md`](./architecture/ARCH.md) | human + AI | ADR、系統級架構決策與後果 | Active SSoT |
| [`threat-model.md`](./threat-model.md) | human + AI | STRIDE 威脅模型、Token 用途隔離、Refresh 重播／競態與緩解措施 | Active SSoT |
| [`RTM.md`](./RTM.md) | AI + QA | US → FR → API → Module → Test → Security／來源狀態、分頁、刪除與時區驗證追溯 | Active SSoT |
| [`RFC-001.md`](./RFC-001.md) | human-primary | 客戶角色與回應層級簡化決策 | Approved |
| [`RFC-002.md`](./RFC-002.md) | human-primary | 法規／知識類型必填與治理邊界 | Approved |
| [`RFC-003.md`](./RFC-003.md) | human-primary | 任務命名、快速改名與台灣時間 | Approved |
| [`RFC-REGISTRY.md`](./RFC-REGISTRY.md) | human-primary | RFC 索引 | Active |
| [`admin-feature-mechanisms.md`](./admin-feature-mechanisms.md) | human-primary | 管理後台六項功能的實際運作機制、規則與使用邊界 | Current |

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
- 不在 Tier 1 複製 `contracts/*.yml` 的完整內容；只引用其用途與值來源。
- 自動驗證：`contracts/*.yml` 變更後跑 `pnpm lint:docs`（broken-link 檢查 + Tier 1 long-line copy 偵測；實作見 [`../../scripts/docs-lint.mjs`](../../scripts/docs-lint.mjs)）。
