---
audience: both
purpose: reference
status: approved
owner: ODA Cyber Konsult
---

# 架構文件索引

## TL;DR

本目錄保存系統層級架構的唯一真實來源（SSoT）與歷史設計參考。短期原型與使用者流程應放在 [`../../02-design/`](../../02-design/)，確認成為穩定架構後才移入本目錄。

## 給 Jake 看（決策 / 簽收）

| 檔名 | audience | purpose | status | 摘要 |
|------|----------|---------|--------|------|
| [`ARCH.md`](./ARCH.md) | human-primary | decision | approved | 架構決策紀錄與已接受的系統決策 |
| [`system-architecture.drawio`](./system-architecture.drawio) | human-primary | decision | approved | 主要系統架構圖 |
| [`system-operation-flows.drawio`](./system-operation-flows.drawio) | human-primary | decision | approved | 跨服務操作流程圖 |

## 給 AI 引用（細節記錄）

| 檔名 | audience | purpose | status | 摘要 |
|------|----------|---------|--------|------|
| [`rag-system-blueprint.md`](./rag-system-blueprint.md) | ai-primary | reference | approved | RAG 實作細節藍圖 |

## 雙用

| 檔名 | audience | purpose | status | 摘要 |
|------|----------|---------|--------|------|
| [`system-overview.md`](./system-overview.md) | both | reference | approved | 服務拓撲、Cleaner 已刪除檢視與台北時區機制 |
| [`data-cleaning-workflow.md`](./data-cleaning-workflow.md) | both | reference | approved | 資料清洗與 Maker-Checker 工作流程 |
| [`rag-pipeline.md`](./rag-pipeline.md) | both | reference | approved | RAG 載入、切塊、索引與檢索流程 |
| [`rag-enhancements.md`](./rag-enhancements.md) | both | reference | approved | Query 改寫、重排序、信心度與品質強化 |
