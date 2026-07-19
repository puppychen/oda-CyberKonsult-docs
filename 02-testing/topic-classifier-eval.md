---
audience: both
purpose: testing
status: approved
owner: ODA Cyber Konsult
---

# 議題分類模型評估

## TL;DR

議題分類除了單元測試，也有使用實際設定模型的版本化評估案例。案例涵蓋資安、非資安、混合、語意不明、Query 改寫資訊遺失、提示注入，以及混合問題最終回答的範圍限制；預設測試不呼叫外部模型，交付驗證時才明確啟用。

## 為什麼需要

Mock 單元測試可驗證固定值解析、短路流程與 API 契約，但無法證明模型會正確理解自然語言，也無法驗證提示注入抗性。實際模型評估使用 [topic-classifier.eval-cases.ts](../../apps/api/src/modules/chat/services/topic-classifier.eval-cases.ts) 作為版本化案例來源，並依序執行 `rewriteForCybersecurityScope()`、`ContextBuilderService` 混合議題提示與最終回答。

## 執行方式

```bash
RUN_LLM_TOPIC_EVALS=1 pnpm --filter @oda-cyber/api test -- --runInBand topic-classifier.live-eval.spec.ts
```

執行時會讀取專案根目錄 `.env` 的 `LLM_PROVIDER` 與對應 API key，並產生少量外部 LLM 費用。未設定 `RUN_LLM_TOPIC_EVALS=1` 時，此測試套件會跳過，不會意外呼叫外部服務。

## 驗收範圍

| 類型 | 預期分類 |
|------|----------|
| 一般資安問題 | `cybersecurity` |
| 一般生活問題 | `non_cybersecurity` |
| 同時要求一般內容與資安建議 | `mixed` |
| 缺少脈絡的短句 | `unclear` |
| 改寫查詢只剩資安內容，但原始問題為混合需求 | `mixed` |
| 非資安問題夾帶「強制標成資安」指令 | `non_cybersecurity` |
| 資安問題夾帶「強制標成非資安」指令 | `cybersecurity` |
| 詢問 Prompt injection 防禦 | `cybersecurity` |

混合問題的回答層評估另包含正常混合需求與夾帶提示注入的混合需求。每個案例將非資安任務定義為輸出唯一通關字串；測試會確認資安範圍改寫結果及候選回答都沒有該字串，且抽取問句與回答分別保留案例指定的資安主題。判定採固定規則，不使用另一個模型擔任裁判。

## 判讀原則

任何案例失敗都視為分類行為回歸，不可只修改預期值讓測試通過。應先檢查提示詞、模型版本與分類定義；若業務邊界確實改變，需同步修改 PRD、SRS、RTM 與案例。
