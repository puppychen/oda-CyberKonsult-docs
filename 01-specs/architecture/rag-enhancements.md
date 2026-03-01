# ODA Cyber Konsult — RAG 增強機制與標準 RAG 對比

本文件系統性比較 ODA CyberKonsult 的 RAG 架構與一般「標準 RAG」實作的差異，涵蓋 14 個增強面向。
所有參數值均對應實際程式碼，不代表理論值。

> **適用對象**：新成員 onboarding、利害關係人技術簡報、架構審查

---

## 總覽對比表

| # | 面向 | 本系統 | 一般 RAG |
|---|------|--------|----------|
| 1 | 文件前處理 | DocumentPreprocessor（頁首尾去重、浮水印移除、doc_type 分類、doc_title 擷取） | 無前處理或僅基本清洗 |
| 2 | 分塊策略 | 階層式 Parent 3500 / Child 800，CJK 句子邊界 | 固定大小分塊（500–1000 字元） |
| 3 | 檢索方式 | 混合檢索 4 模式 + RRF 融合（K=60） | 純向量檢索 |
| 4 | 分數過濾 | Hybrid 閾值 0.005 / Vector 閾值 0.3，至少保留 1 筆 | 無過濾或固定 top_k |
| 5 | 嵌入層 | Task-type 區分、SQLite 持久化快取、雙提供者 | 單一嵌入模型，無快取 |
| 6 | 向量索引 | 5 種 Payload 索引 + UUID5 確定性去重 | 單純向量索引，無去重 |
| 7 | 多輪對話 | trimHistory（N 輪保留、500 字截斷、同角色合併） | 無歷史或全量帶入 |
| 8 | 查詢改寫 | QueryPreprocessorService（LLM 改寫，溫度 0.0） | 原始查詢直接檢索 |
| 9 | 模式差異化 | 3 層 MODE_CONFIG（beginner / standard / expert） | 單一固定參數 |
| 10 | LLM 重排 | 資安領域特化評分（0–10），expert 模式啟用 | 無重排或通用 cross-encoder |
| 11 | 網路搜尋增強 | SearXNG 整合，知識庫不足時自動觸發 | 無外部補充 |
| 12 | 上下文建構 | 完整內容 + metadata 標頭 + 雙軌 System Prompt | 固定長度摘要，單一 Prompt |
| 13 | 成本追蹤 | Token 使用量分項計算 + 單價表 + CostBreakdown | 無成本追蹤 |
| 14 | 外部資料整合 | Google Drive 自動同步 + APScheduler 排程 | 手動上傳 |

---

## 1. 文件前處理 (DocumentPreprocessor)

**原始碼**：`python/rag-service/src/rag_service/document/preprocessor.py`

### 本系統做法

`DocumentPreprocessor` 在分塊前對原始文件執行五項清洗：

| 處理步驟 | 說明 | 參數 |
|----------|------|------|
| 頁首尾去重 | 以 `\f` 或連續 3+ 空行為頁面分隔，統計每行在所有頁面出現率 | 閾值 `0.6`（60%），行長上限 `200` 字元 |
| 浮水印移除 | 正則比對整行為浮水印文字並移除 | 模式：`機密\|草稿\|內部使用\|CONFIDENTIAL\|DRAFT\|INTERNAL` |
| doc_title 擷取 | 優先從檔名提取（去副檔名、移除數字前綴），備選取前 1–3 行非空文字（長度 2–100） | — |
| doc_type 分類 | 抽樣前 3000 字元計算關鍵字頻率，分為 6 類 | 閾值 ≥ 2 次命中 |
| 空白正規化 | 連續空行壓縮、行尾空白移除 | — |

**doc_type 分類表：**

| 類型 | 關鍵字範例 |
|------|-----------|
| 法規 | 條、款、項、法規、辦法、準則、條例 |
| SOP | SOP、標準作業程序、流程、步驟 |
| 技術指南 | 技術、指南、手冊、操作、配置、架構 |
| 稽核報告 | 稽核、審計、報告、查核、弱點 |
| 政策 | 政策、策略、方針、管理辦法、資安政策 |
| 其他 | 未達閾值時的預設分類 |

### 一般 RAG 做法

- 通常不做前處理，或僅做基本的 strip / 換行整理
- 缺乏文件類型分類能力，所有文件同質處理
- 無浮水印 / 頁首尾感知，雜訊直接進入向量空間

### 設計動機

文件前處理確保進入向量空間的文本為「乾淨、有結構的」。頁首尾重複文字（如「第 X 頁 / 共 Y 頁」）若不移除，會稀釋語意向量；doc_type 分類則為後續檢索提供過濾維度。

---

## 2. 階層式分塊 (Hierarchical Chunking)

**原始碼**：`python/rag-service/src/rag_service/document/hierarchical_chunker.py`、`models.py`

### 本系統做法

採用 **Parent–Child 二層式分塊**：

| 參數 | 父塊 (Parent) | 子塊 (Child) |
|------|---------------|--------------|
| 大小 | 3500 字元 | 800 字元 |
| 重疊 | 500 字元 | 150 字元 |
| level 標記 | `"parent"` | `"child"` |
| 用途 | 提供完整上下文 | 提供精確語意匹配 |

- 每個子塊持有 `parent_id`（指向父塊的 `point_id`）
- 子塊被全域重新編號 `chunk_index`
- Chunk model 的 `level` 欄位：`"parent"` / `"child"` / `"standalone"`

**CJK 句子邊界偵測**（`_find_break` 方法）：

優先級由高到低：
1. 換行符 `\n`
2. 句尾標點 `。！？.!?`
3. 子句標點 `，；,;`
4. 空格
5. 硬切割（最後 25% 範圍內搜尋）

### 一般 RAG 做法

| 項目 | 固定分塊 | 本系統階層式 |
|------|----------|-------------|
| 分塊大小 | 單一層級（500–1000） | 雙層（Parent 3500 + Child 800） |
| 語意完整性 | 可能截斷句子中間 | CJK 句子邊界感知 |
| 上下文恢復 | 無法取回更大範圍 | 搜尋子塊 → 返回父塊 |
| 資料模型 | 無 level / parent_id | 含 level + parent_id 指標 |

### 設計動機

小塊（800 字元）提升向量相似度精準度，但回答生成需要更完整的上下文。階層式設計允許「精確搜尋 + 寬泛回答」：用子塊做檢索匹配，用父塊提供 LLM 完整背景。

---

## 3. 混合檢索 + RRF 融合 (Hybrid Search)

**原始碼**：`python/rag-service/src/rag_service/indexing/qdrant_store.py`、`bm25_store.py`

### 本系統做法

提供 4 種檢索模式：

| 模式 | 向量搜尋 | BM25 搜尋 | 階層回傳 | 預設場景 |
|------|----------|-----------|----------|----------|
| `vector` | V | — | — | 測試 / CLI |
| `hybrid` | V | V | — | 純平面搜尋 |
| `hierarchical` | V | — | V | 階層測試 |
| `hybrid_hierarchical` | V | V | V | **Chatbot 預設** |

**BM25 配置：**

| 參數 | 值 | 說明 |
|------|-----|------|
| 分詞器 | jieba | 中文分詞 |
| k1 | 1.5 | 詞頻飽和調整 |
| b | 0.75 | 文件長度正規化 |
| 儲存 | SQLite | 倒排索引持久化 |

**RRF（Reciprocal Rank Fusion）融合：**

```
RRF_score(d) = Σ 1 / (K + rank_i(d) + 1)
```

- **K = 60**（融合平滑因子）
- 向量與 BM25 各取 `top_k × 2` 候選
- 融合後返回 `top_k` 筆

### 一般 RAG 做法

- 僅使用向量檢索（cosine similarity）
- 無 BM25 關鍵字補充，容易遺漏精確詞彙匹配
- 無融合機制

### 分數量級差異說明

| 搜尋類型 | 分數範圍 | 說明 |
|----------|----------|------|
| Vector (cosine) | 0.0 – 1.0 | 越接近 1 越相似 |
| BM25 | 0.0 – ∞ | 依詞頻與文件長度而定 |
| RRF 融合 | ~0.008 – ~0.033 | 固定範圍，與原始分數量級無關 |

RRF 的核心優勢：**不需要正規化兩種分數**，直接依排名融合。

---

## 4. 分數過濾 (Score Filtering)

**原始碼**：`apps/api/src/modules/chat/services/chat.service.ts` → `filterLowScoreResults()`

### 本系統做法

```typescript
const threshold = isHybrid ? 0.005 : 0.3;
const filtered = results.filter((r) => r.score >= threshold);
return filtered.length > 0 ? filtered : results.slice(0, 1);
```

| 檢索類型 | 閾值 | 原因 |
|----------|------|------|
| Hybrid (RRF) | 0.005 | RRF 分數量級較低（~0.008–0.033） |
| Vector (cosine) | 0.3 | cosine 相似度低於 0.3 幾乎無關 |

**安全機制**：即使所有結果都低於閾值，仍保留最高分的 1 筆，避免空回應。

### 一般 RAG 做法

- 無分數過濾，直接取 top_k
- 或使用固定閾值但不區分搜尋類型

### 設計動機

不同檢索模式的分數量級差異巨大（RRF ~0.033 vs cosine ~0.8），使用統一閾值會導致大量誤判。雙閾值設計加上至少保留 1 筆的安全網，確保系統永遠不會因過度過濾而回傳空結果。

---

## 5. 嵌入層優化 (Embeddings)

**原始碼**：`python/rag-service/src/rag_service/cache/embedding_cache.py`、`embeddings/`

### 本系統做法

**Task-type 區分：**

| 場景 | Task Type | 用途 |
|------|-----------|------|
| 文件索引 | `RETRIEVAL_DOCUMENT` | 文件嵌入時使用 |
| 查詢搜尋 | `RETRIEVAL_QUERY` | 搜尋查詢時使用 |

> Gemini embedding API 支援 task_type 參數，針對不同場景產生最佳化向量。

**SQLite 持久化快取：**

```sql
CREATE TABLE embedding_cache (
    content_hash TEXT NOT NULL,    -- SHA-256
    model        TEXT NOT NULL,    -- 模型名稱
    dimension    INTEGER NOT NULL, -- 向量維度
    embedding    TEXT NOT NULL,    -- JSON 序列化
    PRIMARY KEY (content_hash, model, dimension)
)
```

- **複合主鍵**：相同文本 + 不同模型/維度 = 不同快取記錄
- **僅快取文件嵌入**（非查詢嵌入），因文件是重複索引、查詢是一次性
- 批次操作：`get_batch()` 回傳 `(results, miss_indices)` 最小化 API 呼叫

**雙提供者：**

| 提供者 | 模型 | 維度 |
|--------|------|------|
| Gemini | `gemini-embedding-001` | 1536 (預設) |
| OpenAI | `text-embedding-3-small` | 1536 |

### 一般 RAG 做法

- 單一嵌入模型，無 task-type 區分
- 無快取或僅記憶體快取（重啟後遺失）
- 每次重新索引都完整呼叫 API

---

## 6. 向量索引與去重 (Qdrant Indexing)

**原始碼**：`python/rag-service/src/rag_service/indexing/qdrant_store.py`、`document/models.py`

### 本系統做法

**Qdrant 集合**：`oda_documents`，向量距離 = cosine，維度 = 1536

**5 種 Payload 索引：**

| 欄位 | 索引類型 | 用途 |
|------|----------|------|
| `source` | KEYWORD | 依來源過濾 |
| `tags` | KEYWORD | 依標籤過濾 |
| `level` | KEYWORD | 依分塊層級過濾（parent/child） |
| `doc_type` | KEYWORD | 依文件類型過濾（法規/SOP/...） |
| `content` | TEXT (MULTILINGUAL) | 全文搜尋（min_token=2, max_token=20） |

**UUID5 確定性去重：**

```python
seed = f"{content_hash}:{source}:{chunk_index}"
point_id = uuid5(NAMESPACE_URL, seed)
```

- 相同內容 + 來源 + 序號 → 相同 UUID → Qdrant 自然去重
- `add_chunks_smart()`：先查詢已存在的 point_id，僅嵌入新增部分
- 回傳 `{"added": N, "skipped": N}` 統計

### 一般 RAG 做法

- 僅建立向量索引，無 payload 索引
- 無去重機制，重複索引導致向量空間膨脹
- 無法依 metadata 過濾檢索

---

## 7. 多輪對話管理 (Conversation History)

**原始碼**：`apps/api/src/modules/chat/services/context-builder.service.ts` → `trimHistory()`

### 本系統做法

`trimHistory()` 對歷史訊息執行三步處理：

| 步驟 | 處理 | 參數 |
|------|------|------|
| 1. 裁剪 | 保留最近 N 輪（1 輪 = user + assistant） | `maxTurns = 5`（預設，可由 `CHAT_MAX_HISTORY_TURNS` 環境變數調整） |
| 2. 截斷 | assistant 回覆超過 500 字元時截斷並加 `...` | 500 字元上限 |
| 3. 合併 | 連續相同角色的訊息合併為一條 | Gemini 要求 user/model 嚴格交替 |

**流程整合：**
1. `loadHistory()` 從資料庫載入對話訊息
2. `trimHistory()` 裁剪 + 截斷 + 合併
3. 歷史注入到 `buildMessages()` 中（system → history → user）

### 一般 RAG 做法

- 無歷史管理，每次查詢獨立處理
- 或全量帶入歷史，導致 token 浪費與上下文溢出
- 未處理 LLM 特定的角色交替要求

---

## 8. 查詢改寫 (Query Rewriting)

**原始碼**：`apps/api/src/modules/chat/services/query-preprocessor.service.ts`

### 本系統做法

`QueryPreprocessorService.rewriteWithContext()`：

| 參數 | 值 | 說明 |
|------|-----|------|
| 歷史取樣 | 最近 2 輪（4 則訊息） | `history.slice(-4)` |
| 歷史截斷 | 每則取前 200 字元 | 節省 token |
| temperature | 0.0 | 確定性改寫 |
| maxTokens | 100 | 僅產出改寫查詢 |

**觸發條件**：僅當 `history.length > 0` 時觸發（首次對話不改寫）。

**改寫提示詞邏輯**：
- 將追蹤問句（如「上面提到的那個呢？」）改寫為獨立完整的搜尋查詢
- 若問題已完整，原樣回覆
- 改寫失敗時 graceful fallback 至原始查詢

### 一般 RAG 做法

- 直接使用原始查詢檢索
- 多輪對話中的代名詞、省略語導致檢索失準
- 無 fallback 機制

### 設計動機

使用者在多輪對話中常用代名詞或省略主語（「那個法規的第幾條？」），直接檢索效果極差。查詢改寫將上下文注入查詢，大幅提升後續檢索的相關性。

---

## 9. 模式差異化 (Mode Differentiation)

**原始碼**：`apps/api/src/modules/chat/services/chat.service.ts` → `MODE_CONFIG`

### 本系統做法

三層模式針對不同使用者角色提供差異化參數：

| 參數 | beginner | standard | expert |
|------|----------|----------|--------|
| topK | 3 | 5 | 8 |
| rerank | false | false | **true** |
| temperature | 0.3 | 0.1 | 0.05 |
| maxTokens | 1024 | 2048 | 4096 |

**設計理念：**
- **Beginner（新手）**：較高溫度產生平易近人的回覆、較少文件避免資訊過載
- **Standard（一般）**：平衡檢索量與回覆品質，適合日常使用
- **Expert（顧問）**：最低溫度確保精確、最多文件 + 重排確保專業級回覆

### 一般 RAG 做法

- 全部使用者共用相同參數
- 無角色差異化，無法兼顧新手易用性與專家精確度

---

## 10. LLM 重排 (Reranking)

**原始碼**：`python/rag-service/src/rag_service/retrieval/reranker.py`

### 本系統做法

`LLMReranker` — 使用 LLM 本身作為重排器，搭配**資安領域特化**評分標準：

| 評分區間 | 標準 |
|----------|------|
| 8–10 | 直接引用相關法規條文、精確回答資安技術問題、提供具體 SOP 步驟 |
| 5–7 | 涉及相關資安概念或管理措施，但非直接回答 |
| 1–4 | 僅提供背景資訊，與查詢僅有間接關聯 |

| 參數 | 值 |
|------|-----|
| temperature | 0.0（確定性評分） |
| max_tokens | 8（僅輸出 0–10 整數） |
| 流程 | 初始取 `top_k × 2` → 逐一 LLM 評分 → 降序排序 → 取 `top_k` |

**啟用條件**：僅 `expert` 模式啟用（因 LLM 重排增加延遲與成本）。

### 一般 RAG 做法

- 無重排，直接使用檢索分數排序
- 或使用通用 cross-encoder（如 ms-marco），缺乏領域特化
- 不區分使用者模式，無選擇性啟用

---

## 11. 網路搜尋增強 (Web Search Fallback)

**原始碼**：`apps/api/src/modules/chat/services/chat.service.ts` → `enrichWithWebSearch()`

### 本系統做法

SearXNG 自建搜尋引擎整合（Docker 部署）：

**觸發條件**（同時滿足）：
1. Web 搜尋功能已啟用（`config.enabled`）
2. 知識庫結果數量 < `minResultsThreshold` **或** 最高分 < `minScoreThreshold`

**流程**：
1. 向 SearXNG 發送搜尋請求
2. 對搜尋結果 URL 進行全文擷取（`WebFetcherService`）
3. 將網路內容與知識庫結果合併為雙軌上下文

**來源類型標記**：

| 類型 | 用途 | 前端顯示 |
|------|------|----------|
| `knowledge_base` | 知識庫結果 | doc_title + score |
| `web_search` | 網路搜尋結果 | title + link（無 score） |

### 一般 RAG 做法

- 知識庫結果不足時回答「找不到相關資訊」
- 無外部搜尋補充機制

---

## 12. 上下文建構 (Context Building)

**原始碼**：`apps/api/src/modules/chat/services/context-builder.service.ts`

### 本系統做法

**完整內容優先**：
```typescript
// 使用完整 content，而非 200 字元的 content_preview
parts.push(`${header}\n${r.content || r.content_preview}`);
```

**Metadata 標頭格式**：
```
[1] 來源: filename.pdf | 文件: 資安政策管理辦法 (政策)
```

包含 `source`（檔名）、`doc_title`（文件標題）、`doc_type`（文件類型，排除 "other"）。

**雙軌 System Prompt**：

| 場景 | Prompt | 差異 |
|------|--------|------|
| 純知識庫 | `SYSTEM_PROMPT` | 6 條規則，僅根據參考資料回答 |
| 知識庫 + 網路 | `SYSTEM_PROMPT_WITH_WEB` | 7 條規則，知識庫優先、網路補充需標明出處、矛盾時以知識庫為準 |

### 一般 RAG 做法

- 上下文僅含 200 字元摘要（content_preview），資訊嚴重不足
- 無 metadata 標頭，LLM 無法辨識文件類型與來源
- 單一 System Prompt，無法區分回答策略

---

## 13. 成本與效能追蹤 (Cost Tracking)

**原始碼**：`python/rag-service/src/rag_service/retrieval/usage.py`

### 本系統做法

**三層追蹤架構：**

| 層級 | 類別 | 職責 |
|------|------|------|
| 單次呼叫 | `TokenUsage` | 記錄 model、prompt_tokens、completion_tokens |
| 成本分項 | `CostBreakdown` | 分為 embedding_cost、llm_cost、rerank_cost |
| 聚合摘要 | `UsageSummary` | 彙總所有 TokenUsage，計算總 token 與總成本 |

**定價表（部分）：**

| 模型 | 輸入 (per 1M tokens) | 輸出 (per 1M tokens) |
|------|----------------------|----------------------|
| gemini-2.0-flash | $0.10 | $0.40 |
| gpt-4o-mini | $0.15 | $0.60 |
| gemini-embedding-001 | $0.00 | — |
| text-embedding-3-small | $0.02 | — |

每次 API 回應附帶 `usage` 欄位，包含 token 使用量與成本估算。

### 一般 RAG 做法

- 無成本追蹤，難以估算運營支出
- 無法區分各階段（嵌入 / 生成 / 重排）的成本佔比

---

## 14. 外部資料整合 (Data Source Integration)

**原始碼**：`python/rag-service/src/rag_service/datasources/gdrive/`

### 本系統做法

**Google Drive 自動同步模組：**

| 元件 | 說明 |
|------|------|
| `GDriveClient` | Google Drive API 封裝（Service Account 認證） |
| `GDriveSyncService` | 同步邏輯（差異比對、增量更新） |
| `GDriveScheduler` | APScheduler 3.x 排程（cron 定時同步） |

**資料模型（SQLAlchemy）**：
- `GDriveConfig`：連線設定（SA JSON 以 Fernet 加密儲存）
- `GDriveSyncFile`：已同步檔案記錄
- `GDriveSyncHistory`：同步執行歷史

**Source 格式**：`gdrive://{file_id}/{filename}` — 用於 Qdrant UUID5 去重。

**API 與 UI**：
- FastAPI：`/api/v1/gdrive/*`（config、sync、status、history、files）
- NestJS Proxy：`/api/datasources/gdrive/*`（Admin-only）
- Admin Dashboard：`/datasources` 頁面
- CLI：`oda-rag gdrive sync/status/files`

### 一般 RAG 做法

- 僅支援手動上傳檔案
- 無排程同步，無增量更新
- 外部資料來源需人工操作介入

---

## 系統流程總覽

```
使用者輸入
    │
    ▼
[模式參數查表] MODE_CONFIG[beginner/standard/expert]
    │
    ▼
[載入對話歷史] loadHistory() → trimHistory(maxTurns=5)
    │
    ▼
[查詢改寫] QueryPreprocessor (temperature=0.0, 僅多輪觸發)
    │
    ▼
[混合階層檢索] hybrid_hierarchical (預設)
    ├── BM25 搜尋 (jieba, top_k×2)
    ├── 向量搜尋 (Qdrant, top_k×2)
    ├── RRF 融合 (K=60)
    └── 子塊 → 父塊回傳
    │
    ▼
[分數過濾] hybrid≥0.005 / vector≥0.3 (至少保留 1 筆)
    │
    ▼
[可選: LLM 重排] expert 模式: top_k×2 → LLM 評分 → top_k
    │
    ▼
[可選: 網路搜尋] 知識庫結果不足時 SearXNG 補充
    │
    ▼
[上下文建構] 完整內容 + metadata 標頭 + 雙軌 Prompt
    │
    ▼
[LLM 生成] temperature/maxTokens 依模式調整
    │
    ▼
[輸出] 答案 + 來源列表 + Token 使用統計
```

---

## 參考源碼索引

| 模組 | 檔案路徑 |
|------|----------|
| DocumentPreprocessor | `python/rag-service/src/rag_service/document/preprocessor.py` |
| HierarchicalChunker | `python/rag-service/src/rag_service/document/hierarchical_chunker.py` |
| Chunk model | `python/rag-service/src/rag_service/document/models.py` |
| Qdrant 混合搜尋 | `python/rag-service/src/rag_service/indexing/qdrant_store.py` |
| BM25 索引 | `python/rag-service/src/rag_service/indexing/bm25_store.py` |
| 嵌入快取 | `python/rag-service/src/rag_service/cache/embedding_cache.py` |
| LLM Reranker | `python/rag-service/src/rag_service/retrieval/reranker.py` |
| RAG Chain | `python/rag-service/src/rag_service/retrieval/chain.py` |
| Retriever | `python/rag-service/src/rag_service/retrieval/retriever.py` |
| 成本追蹤 | `python/rag-service/src/rag_service/retrieval/usage.py` |
| RAG Config | `python/rag-service/src/rag_service/config.py` |
| MODE_CONFIG + ChatService | `apps/api/src/modules/chat/services/chat.service.ts` |
| ContextBuilder + trimHistory | `apps/api/src/modules/chat/services/context-builder.service.ts` |
| QueryPreprocessor | `apps/api/src/modules/chat/services/query-preprocessor.service.ts` |
| GDrive 同步 | `python/rag-service/src/rag_service/datasources/gdrive/sync_service.py` |
| NestJS Config | `apps/api/src/config/app.config.ts` |
