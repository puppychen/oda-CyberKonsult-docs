---
audience: both
purpose: testing
status: approved
owner: ODA Cyber Konsult
---

# 議題分類模型評估

## TL;DR

議題分類器只提供初判，最終回答範圍由「初判 → RAG 證據 → 必要的限定網搜 → 提醒／回答」解析器決定。預設閘門以 mock 與固定邊界值驗證完整解析流程，不呼叫外部 LLM 或 SearXNG；實際模型評估仍須明確啟用。

## 為什麼需要

Mock 單元測試可驗證固定值解析、證據門檻、流程分支、原子銜接、API 契約與提示結構，但無法證明實際模型會正確理解所有自然語言。實際模型評估使用 [topic-classifier.eval-cases.ts](../../apps/api/src/modules/chat/services/topic-classifier.eval-cases.ts) 作為版本化案例來源，並依序執行 `rewriteForCybersecurityScope()`、`ContextBuilderService` 混合議題提示與最終回答。

## 執行方式

```bash
RUN_LLM_TOPIC_EVALS=1 pnpm --filter @oda-cyber/api test -- --runInBand topic-classifier.live-eval.spec.ts
```

執行時會讀取專案根目錄 `.env` 的 `LLM_PROVIDER` 與對應 API key，並產生少量外部 LLM 費用。未設定 `RUN_LLM_TOPIC_EVALS=1` 時，此測試套件會跳過，不會意外呼叫外部服務。

## 驗收範圍

### 分類器初判

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

### 最終範圍解析

| 情境 | 決定性預期 | 主要測試 |
|------|------------|----------|
| `SGS`／`TUV`／`TAF` 初判不明確，但 RAG 分數等於正式門檻 | 依知識庫回答，最終為 `cybersecurity` | `chat.service.spec.ts` |
| RAG 分數為正式門檻減 ε，只有 top 1 fallback | 不得視為恢復證據 | `chat.service.spec.ts` |
| 縮寫無 RAG 證據，但安全網搜有非空內容 | `web_inference`、低信心、API 固定加推測前綴並附網路來源 | `chat.service.spec.ts`、`chat.controller.spec.ts`、`context-builder.service.spec.ts` |
| 安全網頁正文與搜尋摘要皆空 | 不得進入 `web_inference`，改要求補充 | `chat.service.spec.ts` |
| `ODA` 無證據後接「資安服務」 | 同一對話合併成一次查詢；真實 PostgreSQL row lock 確保雙併發只消耗一次 | `chat.service.spec.ts`、`message.repository.spec.ts`、`test:chat-topic-bridge-integration` |
| `ODA` 後接無問號完整敘述 | 不合併，依新問題獨立解析 | `chat.service.spec.ts` |
| 候選片段後接完整新問題 | 不合併舊片段 | `chat.service.spec.ts` |
| RAG 服務逾時／錯誤 | `rag_unavailable`，不得呼叫 SearXNG | `rag-proxy.service.spec.ts`、`chat.service.spec.ts` |
| 建議題生成、驗證失敗或逾時 | 仍回傳可送出的固定資安問題 | `chat.service.spec.ts`、`llm.service.spec.ts`、Chatbot `MessageList.test.tsx` |
| 網頁 URL 指向私有、metadata、IPv6 非公開位址或重新導向至內網 | 不抓取、不列入引用 | `public-url-safety.service.spec.ts`、`web-fetcher.service.spec.ts` |
| 歷史、知識庫或網頁內容含操作指令 | 提示將內容標示為不可信資料，不得遵循 | `query-preprocessor.service.spec.ts`、`context-builder.service.spec.ts` |
| 後台測試提示詞與正式 Chat 使用相同模板、變數 | `rendered`／`runtimeData` 分別等於正式 system／user data message；執行期資料不得出現在 system | `prompts.service.spec.ts`、`prompt-renderer.util.spec.ts`、`context-builder.service.spec.ts` |
| 模型完成後於建議題等待期間 SSE 斷線 | 回答先落盤，再更新建議題；不釋放已完成的一般或 regeneration claim | `chat.controller.spec.ts` |
| SSE `done` 遺失但回答已落盤 | 依 attempt ID 復原伺服器回答，不以問題文字猜測、不重新送出；狀態查詢失敗時 fail-closed | Chatbot `useChat.test.ts` |
| 同一一般訊息 attempt 併發重試 | user row lock 後只允許一個 claim 建立訊息及保留額度，另一個回處理中衝突 | `message.repository.spec.ts`、`chat.service.spec.ts`、`test:chat-topic-bridge-integration` |
| RAG result／score／temporal metadata 異常，或正文與摘要皆空白 | 分流為 `rag_unavailable`，不得當作知識庫證據或進入推測流程 | `rag-proxy.service.spec.ts` |
| Node pinned DNS lookup 使用 `all=true` | 回傳 `LookupAddress[]`，避免 Node 22 autoSelectFamily 解析失敗 | `web-fetcher.service.spec.ts` |

預設決定性驗證使用專案測試指令，不設定 `RUN_LLM_TOPIC_EVALS`。外部 LLM 與 SearXNG smoke test 會產生費用或依賴外部環境，不屬於預設品質閘門，也不得在未取得明確同意時執行。

## 判讀原則

任何案例失敗都視為分類或範圍解析行為回歸，不可只修改預期值讓測試通過。應先檢查初判、RAG 可用性、證據門檻、限定網搜、單輪銜接、提示詞與模型版本；若業務邊界確實改變，需同步修改 PRD、SRS、RTM、Chat API 與案例。
