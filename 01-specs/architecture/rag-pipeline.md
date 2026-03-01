# RAG 管線架構

## 概述

RAG (Retrieval-Augmented Generation) 管線負責將原始文件轉換為可檢索的知識庫，並在使用者查詢時提供相關上下文。

## 管線階段

### 1. 文件載入 (Loading)
- 支援格式：PDF、DOCX、XLSX、HTML、PPTX、EPUB、MSG、TXT、MD、JSON、JSONL、CSV
- 由 `rag-service/document/loader.py` 處理
- 使用 `data-pipeline/loaders/ParserFactory` 處理 PDF/DOCX/XLSX 等複雜格式
- 原生處理 TXT/MD/JSON/JSONL/CSV（含自動編碼偵測：UTF-8/BIG5/GB2312/CP950）

### 2. 文件前處理 (Preprocessing)
- 由 `rag-service/document/preprocessor.py` 的 `DocumentPreprocessor` 處理
- **移除重複頁首頁尾**：依分頁符號或連續空行切分頁面，超過 60% 頁面出現的相同行自動移除
- **移除浮水印**：正則移除獨立行的「機密」「草稿」「CONFIDENTIAL」「DRAFT」「內部使用」等
- **擷取文件標題**：從檔名（去除數字前綴）或前 3 行非空行提取
- **分類文件類型**：基於前 3000 字元的關鍵字頻率分類為：法規/SOP/技術指南/稽核報告/政策/other
- **正規化空白行**：連續 3+ 空行壓縮為 2 行
- 可透過 `preprocess=False` 關閉

### 3. 文件切塊 (Chunking)
- **固定大小切塊**：預設 800 字元（~400 中文字），overlap 150 字元（~75 中文字）
- **階層式切塊**：Parent 3500 字元（~1750 中文字，overlap 500）→ Child 800 字元（overlap 150）
- 支援 CJK 句子邊界偵測，避免在句子中間切斷
- Chunk 帶有 `level` 欄位：`"parent"` / `"child"` / `"standalone"`
- 保留 metadata（來源、頁碼、doc_title、doc_type 等）
- 由 `rag-service/document/` 下的 chunker 模組處理

### 4. 向量化 (Embedding)
- 將文件片段轉換為向量
- 支援 Gemini 與 OpenAI 兩種 provider
- 內建 SQLite embedding cache 避免重複運算
- 由 `rag-service/embeddings/` 處理

### 5. 索引建立 (Indexing)
- 向量存入 Qdrant（支援 Cosine similarity）
- BM25 倒排索引存入 SQLite
- Qdrant payload indexes：source、tags、content(fulltext)、level、doc_type
- 使用 UUID5 確定性去重
- 由 `rag-service/indexing/` 處理

### 6. 檢索 (Retrieval)
- 4 種檢索模式：
  - **Vector**：純向量相似度搜尋
  - **Hybrid**（預設）：向量 + BM25 關鍵字搜尋，RRF 融合
  - **Hierarchical**：純向量搜尋 child → 回傳 parent
  - **Hybrid Hierarchical**（推薦）：BM25 + 向量搜尋 child → 回傳 parent
- 可選 LLM Reranking（資安領域專用評分標準）
- 由 `rag-service/retrieval/` 處理

## 技術選型

- **Google Generative AI (Gemini)** - Embedding 與 LLM 提供者
- **OpenAI** - 備用 Embedding 與 LLM 提供者
- **Qdrant** - 高效能向量資料庫
- **SQLite** - BM25 倒排索引與 Embedding 快取
- **pdfminer.six** - PDF 解析（MIT 授權）
- **markitdown** - PPTX/XLS/EPUB/MSG 等格式的備用解析器

## 環境變數

核心 chunking 參數（均可透過環境變數覆蓋）：

| 變數 | 預設值 | 說明 |
|------|--------|------|
| `RAG_CHUNK_SIZE` | 800 | Child chunk 大小（字元） |
| `RAG_CHUNK_OVERLAP` | 150 | Child chunk 重疊（字元） |
| `RAG_PARENT_CHUNK_SIZE` | 3500 | Parent chunk 大小（字元） |
| `RAG_PARENT_CHUNK_OVERLAP` | 500 | Parent chunk 重疊（字元） |
