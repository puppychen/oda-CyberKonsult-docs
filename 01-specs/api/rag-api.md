# RAG API 文件

> ODA Cyber Konsult RAG (Retrieval-Augmented Generation) 系統 API

Base URL: `http://localhost:8000/api/v1`

---

## 目錄

- [RAG 查詢](#rag-查詢)
- [文件匯入](#文件匯入)
- [系統管理](#系統管理)

---

## RAG 查詢

### POST `/rag/retrieve`

僅執行檢索（向量搜尋 + BM25 + 可選 rerank），不觸發 LLM 生成。

**Request Body:**

| 欄位 | 類型 | 必填 | 預設 | 說明 |
|------|------|------|------|------|
| `question` | string | Y | - | 使用者問題 |
| `top_k` | integer | N | 5 | 檢索結果數量 (1-20) |
| `hybrid` | boolean | N | true | 啟用混合搜尋 (向量 + BM25 + RRF) |
| `hierarchical` | boolean | N | false | 啟用階層式檢索 (搜 child 回傳 parent) |
| `rerank` | boolean | N | false | 啟用 LLM 重排 |
| `source_filter` | string | N | null | 按來源文件路徑過濾 |
| `tag_filter` | string | N | null | 按標籤過濾 |

> **模式組合**：`hybrid` 與 `hierarchical` 可同時啟用，系統會自動使用 `hybrid_hierarchical` 模式（BM25 + 向量搜尋 child → 回傳 parent）。

**Request Example:**

```json
{
  "question": "什麼是 PII？",
  "top_k": 5,
  "hybrid": true,
  "hierarchical": true,
  "rerank": false
}
```

**Response:**

```json
{
  "results": [
    {
      "source": "/data/documents/pii-guide.pdf",
      "content_preview": "PII（個人可識別資訊）是指...",
      "score": 0.89,
      "content": "PII（個人可識別資訊）是指可單獨或結合其他資訊用於識別特定個人的資訊。常見的 PII 包括...",
      "doc_title": "PII 保護實務指南",
      "doc_type": "regulation"
    },
    {
      "source": "/data/documents/data-protection.pdf",
      "content_preview": "保護 PII 的方式包括加密...",
      "score": 0.82,
      "content": "保護 PII 的方式包括加密、存取控制、去識別化等技術手段...",
      "doc_title": "資料保護標準作業程序",
      "doc_type": "procedure"
    }
  ],
  "metadata": {
    "count": 5,
    "best_score": 0.89
  }
}
```

**Response 欄位說明：**

| 欄位 | 類型 | 說明 |
|------|------|------|
| `source` | string | 文件來源路徑 |
| `content_preview` | string | 內容前 200 字摘要（供前端顯示） |
| `score` | float | 相關度分數（hybrid 模式為 RRF 分數，vector 模式為 cosine similarity） |
| `content` | string | 完整 chunk 內容（供 LLM context 使用） |
| `doc_title` | string | 文件標題（由 DocumentPreprocessor 擷取） |
| `doc_type` | string | 文件類型（regulation/procedure/standard/guideline/other） |

> **分數量級**：hybrid 模式使用 RRF 分數（最大約 0.033），與 vector 模式 cosine similarity (0-1) 量級不同。NestJS 端會依模式使用不同門檻過濾低分結果。
```

### POST `/rag/query`

> **已由 NestJS 接管**：此端點仍保留供 CLI (`oda-rag`) 使用，但 NestJS 不再直接呼叫此端點。NestJS 改用 `/retrieve` 取得檢索結果後直接呼叫 LLM API 生成回答。

執行 RAG 查詢，返回基於知識庫的生成回答。

**Request Body:**

| 欄位 | 類型 | 必填 | 預設 | 說明 |
|------|------|------|------|------|
| `question` | string | Y | - | 使用者問題 |
| `top_k` | integer | N | 5 | 檢索結果數量 (1-20) |
| `hybrid` | boolean | N | true | 啟用混合搜尋 (向量 + BM25 + RRF) |
| `hierarchical` | boolean | N | false | 啟用階層式檢索 (搜 child 回傳 parent) |
| `rerank` | boolean | N | false | 啟用 LLM 重排 |
| `source_filter` | string | N | null | 按來源文件路徑過濾 |
| `tag_filter` | string | N | null | 按標籤過濾 |

> **模式組合**：`hybrid` 與 `hierarchical` 可同時啟用，系統會自動使用 `hybrid_hierarchical` 模式。

**Request Example:**

```json
{
  "question": "什麼是 PII？如何保護個人資料？",
  "top_k": 5,
  "hybrid": true,
  "hierarchical": true,
  "rerank": false
}
```

**Response:**

```json
{
  "answer": "PII（個人可識別資訊）是指...",
  "sources": [
    {
      "source": "/data/documents/pii-guide.pdf",
      "content_preview": "PII 的定義包含...",
      "score": 0.89
    }
  ],
  "usage": {
    "total_tokens": 1250,
    "total_cost_usd": 0.000125,
    "details": [
      {
        "model": "gemini-2.0-flash",
        "prompt_tokens": 1000,
        "completion_tokens": 250,
        "cost_usd": 0.000125
      }
    ]
  }
}
```

### POST `/rag/query/stream`

> **已由 NestJS 接管**：此端點仍保留供 CLI (`oda-rag`) 使用，但 NestJS 不再直接呼叫此端點。NestJS 改用 `/retrieve` 取得檢索結果後直接呼叫 LLM API 生成回答。

串流回應查詢。Request Body 同 `/rag/query`。

**Response:** `text/plain` 串流，逐字元回傳生成內容。

### POST `/rag/generate-title`

> **已由 NestJS 接管**：此端點仍保留供 CLI 使用，但 NestJS 不再直接呼叫此端點。NestJS 改用自身的 LLM 呼叫邏輯生成對話標題。

基於對話內容生成標題。

**Request Body:**

| 欄位 | 類型 | 必填 | 說明 |
|------|------|------|------|
| `messages` | array | Y | 對話訊息陣列 (格式同 OpenAI Chat API) |

**Response:**

```json
{
  "title": "PII 保護與法規遵循討論"
}
```

### GET `/rag/stats`

取得 RAG 系統統計資訊。

**Response:**

```json
{
  "collection": {
    "name": "oda_documents",
    "points_count": 1500,
    "vectors_count": 1500,
    "status": "green"
  },
  "bm25": {
    "documents": 1500,
    "vocabulary_size": 8500
  },
  "embedding_cache": {
    "total_entries": 1500
  }
}
```

### DELETE `/rag/documents/{source}`

刪除指定來源的所有文件向量。

**Path Parameters:**

| 參數 | 說明 |
|------|------|
| `source` | 文件來源路徑 |

**Response:**

```json
{
  "source": "/data/documents/old-file.pdf",
  "deleted_count": 25
}
```

---

## 文件匯入

### POST `/ingest`

上傳並匯入檔案至向量資料庫。

**Request:** `multipart/form-data`

| 欄位 | 類型 | 必填 | 說明 |
|------|------|------|------|
| `files` | File[] | Y | 上傳的檔案 (支援 PDF, DOCX, XLSX, TXT, MD, JSON, JSONL, CSV) |
| `tags` | string | N | 逗號分隔的標籤 |
| `hierarchical` | boolean | N | 使用階層式切分 |

**Response:**

```json
{
  "files_processed": 3,
  "chunks_added": 150,
  "chunks_skipped": 0,
  "errors": []
}
```

### POST `/ingest/directory`

批次匯入目錄中所有支援的檔案。

**Request Body:**

| 欄位 | 類型 | 必填 | 預設 | 說明 |
|------|------|------|------|------|
| `directory` | string | Y | - | 目錄路徑 |
| `tags` | string[] | N | [] | 標籤列表 |
| `force` | boolean | N | false | 強制重新匯入 (刪除後重建) |
| `hierarchical` | boolean | N | false | 使用階層式切分 |

**Request Example:**

```json
{
  "directory": "./data/cybersecurity-docs",
  "tags": ["cybersecurity", "compliance"],
  "force": false,
  "hierarchical": true
}
```

**Response:** 同 `POST /ingest`

---

## 系統管理

### CLI 工具

安裝後可使用 `oda-rag` 命令列工具：

```bash
# 匯入文件
oda-rag ingest ./data --tag cybersecurity --hierarchical

# 查詢
oda-rag query "什麼是 PII？" --hybrid --top-k 5

# 系統狀態
oda-rag status

# 列出來源/標籤
oda-rag list --sources
oda-rag list --tags

# 重建 BM25 索引
oda-rag bm25 build
```

---

## 檢索模式說明

| 模式 | 參數 | 說明 |
|------|------|------|
| **Vector** | `hybrid: false` | 純向量相似度搜尋 (Cosine) |
| **Hybrid** | `hybrid: true`（預設） | 向量 + BM25 關鍵字搜尋，使用 RRF (Reciprocal Rank Fusion) 融合 |
| **Hierarchical** | `hierarchical: true, hybrid: false` | 純向量搜尋 Child chunks，回傳 Parent chunks（更多上下文） |
| **Hybrid Hierarchical** | `hybrid: true, hierarchical: true` | BM25 + 向量搜尋 Child chunks → 回傳 Parent chunks（推薦模式） |
| **Rerank** | `rerank: true` | 可與任何模式搭配，LLM 對結果重新評分 |

> **NestJS Chatbot 預設模式**：NestJS ChatService 預設使用 `hybrid: true, hierarchical: true`，即 Hybrid Hierarchical 模式，兼顧精確關鍵字匹配與完整上下文。

---

## 支援的文件格式

| 格式 | 副檔名 | 解析方式 |
|------|--------|----------|
| PDF | `.pdf` | pdfminer.six |
| Word | `.docx`, `.doc` | python-docx |
| Excel | `.xlsx`, `.xls` | openpyxl |
| 純文字 | `.txt` | 原生讀取 (自動偵測編碼) |
| Markdown | `.md` | 原生讀取 |
| JSON | `.json` | 結構化提取 |
| JSONL | `.jsonl` | 逐行解析 |
| CSV | `.csv` | 表格轉文字 |

---

## 環境變數

| 變數 | 預設 | 說明 |
|------|------|------|
| `RAG_GOOGLE_API_KEY` | - | Google AI API Key (Gemini) |
| `RAG_OPENAI_API_KEY` | - | OpenAI API Key |
| `RAG_EMBEDDING_PROVIDER` | `gemini` | Embedding 提供者 (`gemini` / `openai`) |
| `RAG_EMBEDDING_DIMENSION` | `768` | 向量維度 |
| `RAG_LLM_PROVIDER` | `gemini` | LLM 提供者 (`gemini` / `openai`) |
| `RAG_LLM_TEMPERATURE` | `0.1` | 生成溫度 |
| `RAG_CHUNK_SIZE` | `800` | Child Chunk 大小 (字元，~400 中文字) |
| `RAG_CHUNK_OVERLAP` | `150` | Child Chunk 重疊 (字元，~75 中文字) |
| `RAG_PARENT_CHUNK_SIZE` | `3500` | Parent Chunk 大小 (字元，~1750 中文字) |
| `RAG_PARENT_CHUNK_OVERLAP` | `500` | Parent Chunk 重疊 (字元，~250 中文字) |
| `RAG_BM25_ENABLED` | `true` | 啟用 BM25 索引 |
| `RAG_RETRIEVAL_TOP_K` | `5` | 預設檢索數量 |

完整環境變數列表請參考 `.env.example`。

---

## 錯誤回應

所有 API 錯誤回傳標準 HTTP 狀態碼：

| 狀態碼 | 說明 |
|--------|------|
| `400` | 請求參數錯誤 |
| `503` | RAG 服務未初始化 (缺少 API Key 或依賴) |
| `500` | 內部伺服器錯誤 |
