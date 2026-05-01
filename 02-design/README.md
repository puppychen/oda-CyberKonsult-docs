# 設計文件（Middle Loop Design）

本目錄是 AI-Native SDLC 在 Standard/Careful 節奏下的設計文件索引。既有系統級架構圖仍保留在 `../01-specs/architecture/` 作為 SSoT；本目錄負責讓 Middle Loop 的 diagram / flow / wireframe / prototype 有固定入口。

## 目前權威來源

現有架構文件與圖檔維持在 `../01-specs/architecture/`，避免破壞既有審計引用。新設計產物可放入本目錄的對應子目錄。

| 類型 | 入口 | 目前來源 | 狀態 |
|------|------|----------|------|
| 系統架構圖 | [`diagrams/`](./diagrams/) | [`../01-specs/architecture/system-architecture.drawio`](../01-specs/architecture/system-architecture.drawio) | Indexed |
| 跨服務流程 | [`flows/`](./flows/) | [`../01-specs/architecture/system-operation-flows.drawio`](../01-specs/architecture/system-operation-flows.drawio) | Indexed |
| 功能線框 | [`wireframes/`](./wireframes/) | 尚未建立獨立 wireframe | Placeholder |
| 原型備忘 | [`prototype/`](./prototype/) | 尚未建立獨立 prototype | Placeholder |

## 放置規則

- 穩定的系統架構 SSoT 放在 `../01-specs/architecture/`。
- 新增跨功能設計圖時放在 `diagrams/` 或 `flows/`，並在本 README 加索引。
- 不複製 PRD、SRS、RTM 或 ARCH 內容；只放指針與設計產物位置。
