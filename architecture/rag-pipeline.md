# RAG 管線架構

## 概述

RAG (Retrieval-Augmented Generation) 管線負責將原始文件轉換為可檢索的知識庫，並在使用者查詢時提供相關上下文。

## 管線階段

### 1. 文件載入 (Loading)
- 支援格式：PDF、HTML、Markdown、純文字
- 由 `data-pipeline/loaders/` 處理

### 2. 文件清洗 (Cleaning)
- 去除頁首頁尾、浮水印等雜訊
- 格式標準化
- 由 `data-pipeline/cleaners/` 處理

### 3. 文件切塊 (Chunking)
- 語意切塊策略
- 保留 metadata（來源、頁碼等）
- 由 `data-pipeline/processors/` 處理

### 4. 向量化 (Embedding)
- 將文件片段轉換為向量
- 由 `rag-service/embeddings/` 處理

### 5. 索引建立 (Indexing)
- 將向量存入 Qdrant
- 由 `rag-service/indexing/` 處理

### 6. 檢索 (Retrieval)
- 查詢時從 Qdrant 檢索相關片段
- 支援 hybrid search（向量 + 關鍵字）
- 由 `rag-service/retrieval/` 處理

## 技術選型

- **Google Generative AI (Gemini)** - Embedding 與 LLM 提供者
- **OpenAI** - 備用 Embedding 與 LLM 提供者
- **Qdrant** - 高效能向量資料庫
- **SQLite** - BM25 倒排索引與 Embedding 快取
