# RAG 系統建構指南

> AI-Readable Blueprint for Building Production-Ready RAG Systems

<!-- AI-PARSEABLE-METADATA-START -->
- VERSION: 1.0.0
- UPDATED: 2026-01-20
- REFERENCE_IMPL: rag-core
- LANGUAGE: Python 3.11+
- COMPLEXITY: Production-Grade
<!-- AI-PARSEABLE-METADATA-END -->

---

## 目錄

- [Part 1: 基礎概念與架構設計](#part-1-基礎概念與架構設計)
  - [1. RAG 系統架構總覽](#1-rag-系統架構總覽)
  - [2. 技術選型決策矩陣](#2-技術選型決策矩陣)
- [Part 2: 核心模組實作指南](#part-2-核心模組實作指南)
  - [3. 配置管理系統](#3-配置管理系統)
  - [4. Embedding 提供者](#4-embedding-提供者)
  - [5. Embedding 快取機制](#5-embedding-快取機制)
  - [6. 文件處理管道](#6-文件處理管道)
  - [7. 向量儲存層](#7-向量儲存層qdrant)
- [Part 3: 進階檢索策略](#part-3-進階檢索策略)
  - [8. 混合檢索與 RRF 融合](#8-混合檢索與-rrf-融合)
  - [9. 階層式檢索](#9-階層式檢索)
  - [10. LLM Reranking](#10-llm-reranking)
- [Part 4: RAG 管道整合](#part-4-rag-管道整合)
  - [11. RAGChain 核心流程](#11-ragchain-核心流程)
  - [12. Prompt 模板設計](#12-prompt-模板設計)
- [Part 5: 成本與效能管理](#part-5-成本與效能管理)
  - [13. Token 使用量追蹤](#13-token-使用量追蹤)
  - [14. 定價表與成本計算](#14-定價表與成本計算)
  - [15. 效能優化最佳實踐](#15-效能優化最佳實踐)
- [Part 6: 介面設計](#part-6-介面設計)
  - [16. REST API 設計](#16-rest-api-設計fastapi)
  - [17. CLI 工具設計](#17-cli-工具設計typer)
- [Part 7: 專案模板](#part-7-專案模板)
  - [18. 專案結構模板](#18-專案結構模板)

---

# Part 1: 基礎概念與架構設計

## 1. RAG 系統架構總覽

<!-- AI-SECTION: architecture_overview -->

### 1.1 核心概念

**RAG（Retrieval-Augmented Generation）** 是一種結合資訊檢索與大型語言模型生成的技術架構：

1. **Retrieval（檢索）**：從知識庫中找出與問題相關的文件片段
2. **Augmentation（增強）**：將檢索到的內容作為上下文
3. **Generation（生成）**：讓 LLM 基於上下文生成回答

### 1.2 系統組件圖

```
┌─────────────────────────────────────────────────────────────────┐
│                         應用介面層                               │
│  ┌─────────────────┐      ┌─────────────────┐                   │
│  │   REST API      │      │   CLI 工具       │                   │
│  │   (FastAPI)     │      │   (Typer)       │                   │
│  └────────┬────────┘      └────────┬────────┘                   │
└───────────┼─────────────────────────┼───────────────────────────┘
            │                         │
            └────────────┬────────────┘
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                      RAG 核心層 (rag/)                           │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │
│  │  RAGChain   │  │  Retriever  │  │  Reranker   │              │
│  │   管道編排   │  │   文件檢索   │  │  LLM 重排   │              │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘              │
│         │                │                │                      │
│  ┌──────┴─────────────────┴────────────────┴──────┐             │
│  │                 Prompt Manager                  │             │
│  │            提示詞模板與上下文格式化              │             │
│  └─────────────────────────────────────────────────┘             │
│                                                                  │
│  ┌─────────────────────────────────────────────────┐            │
│  │  Token Usage Tracker / Cost Calculator          │            │
│  │            使用量追蹤與成本計算                  │            │
│  └─────────────────────────────────────────────────┘            │
└─────────────────────────────────────────────────────────────────┘
                         │
            ┌────────────┼────────────┐
            ▼            ▼            ▼
┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
│   Embedding 層   │ │     LLM 層       │ │   Cache 層       │
│  ┌────────────┐  │ │  ┌────────────┐  │ │  ┌────────────┐  │
│  │  Gemini    │  │ │  │  OpenAI    │  │ │  │  SQLite    │  │
│  │  OpenAI    │  │ │  │  Gemini    │  │ │  │  Embedding │  │
│  └────────────┘  │ │  └────────────┘  │ │  │  Cache     │  │
└──────────────────┘ └──────────────────┘ │  └────────────┘  │
                                          └──────────────────┘
            │
            ▼
┌─────────────────────────────────────────────────────────────────┐
│                      儲存層                                      │
│  ┌─────────────────────────┐  ┌─────────────────────────┐       │
│  │     Qdrant              │  │     BM25Store           │       │
│  │     向量資料庫           │  │     關鍵字索引          │       │
│  │     (Docker)            │  │     (SQLite)            │       │
│  └─────────────────────────┘  └─────────────────────────┘       │
└─────────────────────────────────────────────────────────────────┘
            ▲
            │
┌─────────────────────────────────────────────────────────────────┐
│                    文件處理層 (document/)                        │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌──────────┐  │
│  │  Loader    │  │  Chunker   │  │Hierarchical│  │Categorizer│ │
│  │  多格式載入 │  │  文件切分   │  │  階層切分   │  │ 自動分類  │ │
│  └────────────┘  └────────────┘  └────────────┘  └──────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### 1.3 資料流程

<!-- AI-SECTION: data_flow -->

```
[匯入階段 Ingestion]

文件 (.txt/.md/.json/.csv)
    │
    ▼
┌─────────────┐
│ DocumentLoader │  ← 自動偵測格式、編碼
└──────┬──────┘
       │
       ▼
┌─────────────┐
│ Chunker     │  ← FixedSize (1000 字) 或 Hierarchical (Parent-Child)
└──────┬──────┘
       │
       ▼
┌─────────────┐
│ Categorizer │  ← [可選] LLM 自動分類
└──────┬──────┘
       │
       ▼
┌─────────────┐        ┌─────────────┐
│ Embedding   │───────→│ Cache Check │  ← SHA-256 Hash 檢查
│ Provider    │        └──────┬──────┘
└──────┬──────┘               │
       │                      │ (Cache Miss)
       │←─────────────────────┘
       ▼
┌─────────────┐        ┌─────────────┐
│ QdrantStore │───────→│ BM25Store   │  ← 同步建立關鍵字索引
│  (向量)     │        │  (關鍵字)   │
└─────────────┘        └─────────────┘


[查詢階段 Query]

使用者問題
    │
    ▼
┌─────────────┐
│ RAGChain    │  ← 協調整個流程
└──────┬──────┘
       │
       ▼
┌─────────────┐
│ Retriever   │  ← 選擇檢索模式
└──────┬──────┘
       │
       ├─────────────────────────────────────┐
       ▼                                     ▼
┌─────────────┐                       ┌─────────────┐
│ Vector      │                       │ Keyword     │
│ Search      │                       │ Search      │
│ (Qdrant)    │                       │ (BM25)      │
└──────┬──────┘                       └──────┬──────┘
       │                                     │
       └──────────────┬──────────────────────┘
                      ▼
               ┌─────────────┐
               │ RRF Fusion  │  ← Reciprocal Rank Fusion
               └──────┬──────┘
                      │
                      ▼
               ┌─────────────┐
               │ Category    │  ← [可選] 分類增強
               │ Boost       │
               └──────┬──────┘
                      │
                      ▼
               ┌─────────────┐
               │ LLM         │  ← [可選] LLM 重新排序
               │ Reranker    │
               └──────┬──────┘
                      │
                      ▼
               ┌─────────────┐
               │ Context     │  ← 格式化上下文
               │ Formatter   │
               └──────┬──────┘
                      │
                      ▼
               ┌─────────────┐
               │ LLM         │  ← 生成回答
               │ Generation  │
               └──────┬──────┘
                      │
                      ▼
               ┌─────────────┐
               │ RAGResponse │  ← 回答 + 來源 + 使用量
               └─────────────┘
```

### 1.4 模組分層架構

<!-- ARCHITECTURE-LAYERS -->

| 層級 | 模組 | 職責 |
|------|------|------|
| **應用層** | `api/`, `cli/` | 對外介面：REST API、CLI 命令 |
| **核心層** | `rag/` | RAG 管道：檢索、重排、生成、使用量追蹤 |
| **抽象層** | `embedding/`, `llm/` | 提供者抽象：支援切換 Gemini/OpenAI |
| **儲存層** | `vectorstore/`, `bm25/`, `cache/` | 持久化：向量、關鍵字索引、快取 |
| **處理層** | `document/` | 文件處理：載入、切分、分類 |
| **配置層** | `config.py` | 統一配置管理 |

---

## 2. 技術選型決策矩陣

<!-- AI-SECTION: tech_decisions -->

### 2.1 向量資料庫選擇

<!-- DECISION: vector_db | qdrant | milvus,chromadb,weaviate | performance,docker,features -->

| 資料庫 | Docker 部署 | 資料規模 | 學習曲線 | 效能 | 推薦場景 |
|--------|-------------|----------|----------|------|----------|
| **Qdrant** | 優秀 | 萬~億級 | 中等 | 優秀 | **生產環境首選** |
| Milvus | 良好 | 億級以上 | 較陡 | 優秀 | 大規模企業部署 |
| Weaviate | 優秀 | 萬~千萬級 | 中等 | 良好 | 知識圖譜場景 |
| ChromaDB | 簡易 | 原型~中型 | 低 | 普通 | 快速原型開發 |

**選擇 Qdrant 的理由**：

1. **Rust 核心**：高效能、低延遲查詢
2. **Docker 部署簡單**：單一容器，資源佔用 < 2GB
3. **Payload 過濾**：支援複雜條件過濾（tags, source, category）
4. **Full-text Index**：內建多語言分詞，支援中文
5. **REST + gRPC**：Python SDK 完善

### 2.2 Embedding 模型選擇

<!-- DECISION: embedding_model | gemini-embedding-001 | openai-text-3-small,cohere-multilingual | accuracy,price,chinese -->

| 模型 | 中文支援 | 維度 | 價格/1M tokens | MTEB 排名 |
|------|----------|------|----------------|-----------|
| **Gemini embedding-001** | 優秀 | 3072 (可調) | $0.15 | 第 1 名 |
| OpenAI text-embedding-3-small | 良好 | 1536 | $0.02 | 第 8 名 |
| OpenAI text-embedding-3-large | 優秀 | 3072 | $0.13 | 第 1 名 |
| Cohere multilingual-v3 | 優秀 | 1024 | $0.10 | 第 3 名 |

**選擇 Gemini 的理由**：

1. **MTEB 多語言排行榜第一**
2. **維度可調**：3072 可降至 1536 節省空間
3. **100+ 語言支援**：繁體中文效果優異
4. **免費額度充足**：開發測試友好

### 2.3 LLM 模型選擇

<!-- DECISION: llm_model | gpt-4o-mini | gemini-2.0-flash,gpt-4o | cost,stability,quality -->

| 模型 | Input/1M | Output/1M | 特點 | 推薦用途 |
|------|----------|-----------|------|----------|
| **GPT-4o-mini** | $0.15 | $0.60 | 穩定、成本效益佳 | **預設推薦** |
| Gemini 2.0 Flash | $0.10 | $0.40 | 最便宜、速度快 | 成本敏感場景 |
| GPT-4o | $2.50 | $10.00 | 最高品質 | 品質優先場景 |
| Gemini 1.5 Pro | $1.25 | $5.00 | 長上下文 (1M tokens) | 長文件處理 |

**模組化設計**：

- 預設使用 OpenAI GPT-4o-mini（穩定性）
- 可切換至 Gemini 2.0 Flash（成本）
- 高品質場景可切換至 GPT-4o

### 2.4 關鍵配置參數

<!-- CONFIG: default_values -->

| 參數 | 預設值 | 範圍 | 說明 |
|------|--------|------|------|
| `embedding_dimension` | 1536 | 768-3072 | 向量維度 |
| `chunk_size` | 1000 | 500-2000 | 切分大小（字元）|
| `chunk_overlap` | 200 | 50-500 | 切分重疊 |
| `retrieval_top_k` | 5 | 1-20 | 檢索數量 |
| `llm_temperature` | 0.7 | 0.0-2.0 | 生成溫度 |
| `bm25_k1` | 1.5 | 1.0-2.0 | BM25 詞頻飽和度 |
| `bm25_b` | 0.75 | 0.0-1.0 | BM25 文件長度正規化 |

---

# Part 2: 核心模組實作指南

## 3. 配置管理系統

<!-- AI-SECTION: config_management -->
<!-- IMPLEMENTS: Settings -->
<!-- CODE-PATTERN: pydantic-settings -->

### 3.1 設計模式

使用 **Pydantic Settings** 統一管理配置，支援：
- 環境變數自動載入
- `.env` 檔案支援
- 型別驗證與範圍檢查
- LRU 快取避免重複解析

### 3.2 配置結構

```python
from enum import Enum
from functools import lru_cache
from typing import Literal

from pydantic import Field
from pydantic_settings import BaseSettings, SettingsConfigDict


class EmbeddingProvider(str, Enum):
    """支援的 Embedding 提供者"""
    GEMINI = "gemini"
    OPENAI = "openai"


class LLMProvider(str, Enum):
    """支援的 LLM 提供者"""
    OPENAI = "openai"
    GEMINI = "gemini"


class Settings(BaseSettings):
    """應用程式設定，從環境變數載入"""

    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        case_sensitive=False,
        extra="ignore",
    )

    # ==========================================================================
    # API Keys
    # ==========================================================================
    google_api_key: str = Field(default="", description="Google API key")
    openai_api_key: str = Field(default="", description="OpenAI API key")

    # ==========================================================================
    # Embedding Configuration
    # ==========================================================================
    embedding_provider: EmbeddingProvider = Field(
        default=EmbeddingProvider.GEMINI,
        description="預設 Embedding 提供者",
    )
    gemini_embedding_model: str = Field(
        default="models/gemini-embedding-001",
        description="Gemini embedding 模型名稱",
    )
    openai_embedding_model: str = Field(
        default="text-embedding-3-small",
        description="OpenAI embedding 模型名稱",
    )
    embedding_dimension: int = Field(
        default=1536,
        description="Embedding 向量維度",
    )

    # ==========================================================================
    # LLM Configuration
    # ==========================================================================
    llm_provider: LLMProvider = Field(
        default=LLMProvider.OPENAI,
        description="預設 LLM 提供者",
    )
    openai_model: str = Field(
        default="gpt-4o-mini",
        description="OpenAI 生成模型",
    )
    gemini_model: str = Field(
        default="gemini-2.0-flash",
        description="Gemini 生成模型",
    )
    llm_temperature: float = Field(
        default=0.7,
        ge=0.0,
        le=2.0,
        description="LLM 生成溫度",
    )
    llm_max_tokens: int = Field(
        default=2048,
        description="LLM 最大回應 tokens",
    )

    # ==========================================================================
    # Qdrant Configuration
    # ==========================================================================
    qdrant_host: str = Field(default="localhost")
    qdrant_port: int = Field(default=6333)
    qdrant_collection_name: str = Field(default="consultation_records")

    # ==========================================================================
    # RAG Configuration
    # ==========================================================================
    retrieval_top_k: int = Field(default=5, ge=1, le=20)
    chunk_size: int = Field(default=1000)
    chunk_overlap: int = Field(default=200)

    # ==========================================================================
    # BM25 Configuration
    # ==========================================================================
    bm25_enabled: bool = Field(default=True)
    bm25_k1: float = Field(default=1.5, ge=1.0, le=2.0)
    bm25_b: float = Field(default=0.75, ge=0.0, le=1.0)
    bm25_tokenizer: Literal["jieba", "simple"] = Field(default="jieba")


@lru_cache
def get_settings() -> Settings:
    """取得快取的設定實例"""
    return Settings()
```

### 3.3 環境變數範本 (.env.example)

```bash
# API Keys
GOOGLE_API_KEY=your-google-api-key
OPENAI_API_KEY=your-openai-api-key

# Embedding
EMBEDDING_PROVIDER=gemini
EMBEDDING_DIMENSION=1536

# LLM
LLM_PROVIDER=openai
OPENAI_MODEL=gpt-4o-mini
LLM_TEMPERATURE=0.7

# Qdrant
QDRANT_HOST=localhost
QDRANT_PORT=6333
QDRANT_COLLECTION_NAME=consultation_records

# RAG
RETRIEVAL_TOP_K=5
CHUNK_SIZE=1000
CHUNK_OVERLAP=200

# BM25
BM25_ENABLED=true
BM25_K1=1.5
BM25_B=0.75
BM25_TOKENIZER=jieba
```

---

## 4. Embedding 提供者

<!-- AI-SECTION: embedding_provider -->
<!-- IMPLEMENTS: EmbeddingProvider -->
<!-- DEPENDS-ON: cache/embedding_cache -->

### 4.1 抽象基類設計

```python
from abc import ABC, abstractmethod


class EmbeddingProvider(ABC):
    """Embedding 提供者抽象基類"""

    @property
    @abstractmethod
    def dimension(self) -> int:
        """回傳向量維度"""
        pass

    @property
    @abstractmethod
    def model_name(self) -> str:
        """回傳模型名稱"""
        pass

    @abstractmethod
    def embed_text(self, text: str) -> list[float]:
        """嵌入單一文本"""
        pass

    @abstractmethod
    def embed_texts(self, texts: list[str]) -> list[list[float]]:
        """批次嵌入多個文本"""
        pass

    def embed_query(self, query: str) -> list[float]:
        """嵌入查詢（可能使用不同的 task_type）"""
        return self.embed_text(query)

    def embed_query_with_usage(self, query: str) -> tuple[list[float], int]:
        """嵌入查詢並回傳 token 使用量"""
        embedding = self.embed_query(query)
        estimated_tokens = len(query) // 4 + 1
        return embedding, estimated_tokens
```

### 4.2 Gemini 實作要點

<!-- CODE-PATTERN: gemini-embedding -->

```python
import time
from typing import Optional
import google.generativeai as genai

from ..cache import EmbeddingCache
from ..config import get_settings
from .base import EmbeddingProvider


class GeminiEmbeddingProvider(EmbeddingProvider):
    """Google Gemini Embedding 提供者"""

    def __init__(
        self,
        api_key: Optional[str] = None,
        model_name: Optional[str] = None,
        dimension: Optional[int] = None,
        use_cache: bool = True,
    ):
        settings = get_settings()

        self._api_key = api_key or settings.google_api_key
        if not self._api_key:
            raise ValueError("需要 Google API Key")

        self._model_name = model_name or settings.gemini_embedding_model
        self._dimension = dimension or settings.embedding_dimension
        self._use_cache = use_cache

        # 初始化快取
        self._cache = EmbeddingCache() if use_cache else None

        # 快取統計
        self._cache_hits = 0
        self._cache_misses = 0

        # 配置 API
        genai.configure(api_key=self._api_key)

    @property
    def dimension(self) -> int:
        return self._dimension

    @property
    def model_name(self) -> str:
        return self._model_name

    @property
    def cache_stats(self) -> dict:
        """快取命中統計"""
        total = self._cache_hits + self._cache_misses
        hit_rate = self._cache_hits / total if total > 0 else 0
        return {
            "hits": self._cache_hits,
            "misses": self._cache_misses,
            "hit_rate": f"{hit_rate:.1%}",
        }

    def _embed_text_api(self, text: str, task_type: str = "RETRIEVAL_DOCUMENT") -> list[float]:
        """呼叫 Gemini API 取得 embedding"""
        result = genai.embed_content(
            model=self._model_name,
            content=text,
            task_type=task_type,
            output_dimensionality=self._dimension,
        )
        return result["embedding"]

    def embed_text(self, text: str) -> list[float]:
        """嵌入單一文本（使用快取）"""
        if not self._use_cache or self._cache is None:
            return self._embed_text_api(text)

        # 檢查快取
        content_hash = EmbeddingCache.compute_hash(text)
        cached = self._cache.get(content_hash, self._model_name, self._dimension)

        if cached is not None:
            self._cache_hits += 1
            return cached

        # 快取未命中 - 計算 embedding
        self._cache_misses += 1
        embedding = self._embed_text_api(text)
        self._cache.set(content_hash, self._model_name, self._dimension, embedding)
        return embedding

    def embed_texts(self, texts: list[str], delay: float = 0.1, max_retries: int = 5) -> list[list[float]]:
        """批次嵌入（含速率限制處理）"""
        embeddings = []
        api_calls_made = 0

        for text in texts:
            # 先檢查快取
            if self._use_cache and self._cache is not None:
                content_hash = EmbeddingCache.compute_hash(text)
                cached = self._cache.get(content_hash, self._model_name, self._dimension)

                if cached is not None:
                    self._cache_hits += 1
                    embeddings.append(cached)
                    continue

            # 快取未命中 - 需要 API 呼叫
            self._cache_misses += 1
            retries = 0

            while retries < max_retries:
                try:
                    embedding = self._embed_text_api(text)
                    embeddings.append(embedding)

                    # 快取結果
                    if self._use_cache and self._cache is not None:
                        content_hash = EmbeddingCache.compute_hash(text)
                        self._cache.set(content_hash, self._model_name, self._dimension, embedding)

                    api_calls_made += 1

                    # API 呼叫間加入延遲
                    if api_calls_made > 0:
                        time.sleep(delay)
                    break

                except Exception as e:
                    if "429" in str(e) or "quota" in str(e).lower():
                        # 速率限制 - 指數退避
                        retries += 1
                        wait_time = 2 ** retries  # 2, 4, 8, 16, 32 秒
                        time.sleep(wait_time)
                    else:
                        raise
            else:
                raise Exception(f"重試 {max_retries} 次後仍失敗")

        return embeddings

    def embed_query(self, query: str) -> list[float]:
        """
        嵌入查詢（使用 RETRIEVAL_QUERY task_type）

        注意：查詢 embedding 不使用快取，因為：
        1. 查詢通常是唯一的
        2. task_type 與文件不同
        """
        return self._embed_text_api(query, task_type="RETRIEVAL_QUERY")
```

### 4.3 關鍵實作要點

<!-- PERFORMANCE: embedding_optimization -->

| 要點 | 說明 |
|------|------|
| **Task Type 區分** | 文件使用 `RETRIEVAL_DOCUMENT`，查詢使用 `RETRIEVAL_QUERY` |
| **快取策略** | 文件 embedding 快取，查詢 embedding 不快取 |
| **速率限制** | 指數退避重試（2, 4, 8, 16, 32 秒）|
| **維度調整** | 使用 `output_dimensionality` 參數降維 |
| **批次延遲** | API 呼叫間延遲 0.1 秒，避免速率限制 |

---

## 5. Embedding 快取機制

<!-- AI-SECTION: embedding_cache -->
<!-- IMPLEMENTS: EmbeddingCache -->
<!-- CODE-PATTERN: sqlite-cache -->

### 5.1 SQLite 持久化設計

```python
import hashlib
import json
import sqlite3
from datetime import datetime
from pathlib import Path
from typing import Optional


class EmbeddingCache:
    """SQLite 持久化 Embedding 快取"""

    def __init__(self, db_path: Optional[str] = None):
        """
        初始化 Embedding 快取

        Args:
            db_path: SQLite 資料庫路徑，預設 ~/.rag-care/embedding_cache.db
        """
        if db_path is None:
            cache_dir = Path.home() / ".rag-care"
            cache_dir.mkdir(parents=True, exist_ok=True)
            db_path = str(cache_dir / "embedding_cache.db")

        self.db_path = db_path
        self._init_schema()

    def _init_schema(self):
        """初始化資料庫結構"""
        with sqlite3.connect(self.db_path) as conn:
            conn.execute("""
                CREATE TABLE IF NOT EXISTS embeddings (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    content_hash TEXT NOT NULL,
                    model TEXT NOT NULL,
                    dimension INTEGER NOT NULL,
                    embedding BLOB NOT NULL,
                    created_at TEXT NOT NULL,
                    UNIQUE(content_hash, model, dimension)
                )
            """)
            conn.execute("""
                CREATE INDEX IF NOT EXISTS idx_content_hash
                ON embeddings(content_hash, model, dimension)
            """)
            conn.commit()

    @staticmethod
    def compute_hash(text: str) -> str:
        """計算文本的 SHA-256 雜湊值（前 16 字元）"""
        return hashlib.sha256(text.encode("utf-8")).hexdigest()[:16]

    def get(self, content_hash: str, model: str, dimension: int) -> Optional[list[float]]:
        """取得快取的 embedding"""
        with sqlite3.connect(self.db_path) as conn:
            cursor = conn.execute(
                """
                SELECT embedding FROM embeddings
                WHERE content_hash = ? AND model = ? AND dimension = ?
                """,
                (content_hash, model, dimension)
            )
            row = cursor.fetchone()
            if row:
                return json.loads(row[0])
            return None

    def set(self, content_hash: str, model: str, dimension: int, embedding: list[float]):
        """儲存 embedding 到快取"""
        embedding_blob = json.dumps(embedding)
        created_at = datetime.now().isoformat()

        with sqlite3.connect(self.db_path) as conn:
            conn.execute(
                """
                INSERT OR REPLACE INTO embeddings
                (content_hash, model, dimension, embedding, created_at)
                VALUES (?, ?, ?, ?, ?)
                """,
                (content_hash, model, dimension, embedding_blob, created_at)
            )
            conn.commit()

    def stats(self) -> dict:
        """取得快取統計資訊"""
        with sqlite3.connect(self.db_path) as conn:
            cursor = conn.execute("SELECT COUNT(*) FROM embeddings")
            total = cursor.fetchone()[0]

            cursor = conn.execute("""
                SELECT model, dimension, COUNT(*) as count
                FROM embeddings
                GROUP BY model, dimension
            """)
            by_model = [
                {"model": row[0], "dimension": row[1], "count": row[2]}
                for row in cursor.fetchall()
            ]

        return {
            "total_entries": total,
            "by_model": by_model,
            "db_path": self.db_path,
        }
```

### 5.2 快取策略

<!-- PERFORMANCE: cache_strategy -->

| 類型 | 是否快取 | 理由 |
|------|----------|------|
| **文件 Embedding** | 是 | 內容固定，常重複處理 |
| **查詢 Embedding** | 否 | 查詢通常唯一，task_type 不同 |

**快取 Key 結構**：`(content_hash, model, dimension)`

**預期效果**：
- 重複匯入時間降低 90%
- API 成本節省 60-80%（視重複內容比例）

---

## 6. 文件處理管道

<!-- AI-SECTION: document_processing -->
<!-- IMPLEMENTS: DocumentLoader, Chunker -->

### 6.1 Document 資料結構

```python
from dataclasses import dataclass
import hashlib


@dataclass
class Document:
    """文件資料結構"""

    content: str
    source: str
    metadata: dict = None
    content_hash: str = ""

    def __post_init__(self):
        if self.metadata is None:
            self.metadata = {}
        if not self.content_hash:
            self.content_hash = hashlib.sha256(
                self.content.encode("utf-8")
            ).hexdigest()[:16]
```

### 6.2 多格式載入器

```python
from pathlib import Path
from typing import Optional
import json
import csv


class DocumentLoader:
    """多格式文件載入器"""

    def __init__(self, content_key: str = "content"):
        """
        Args:
            content_key: JSON/CSV 中表示內容的欄位名稱
        """
        self.content_key = content_key

    def load_file(self, file_path: str) -> list[Document]:
        """載入單一檔案"""
        path = Path(file_path)
        suffix = path.suffix.lower()

        if suffix in [".txt", ".md", ".markdown"]:
            return self._load_text(path)
        elif suffix == ".json":
            return self._load_json(path)
        elif suffix == ".jsonl":
            return self._load_jsonl(path)
        elif suffix == ".csv":
            return self._load_csv(path)
        else:
            raise ValueError(f"不支援的檔案格式: {suffix}")

    def load_directory(self, dir_path: str, recursive: bool = True) -> list[Document]:
        """載入目錄下所有檔案"""
        path = Path(dir_path)
        documents = []

        pattern = "**/*" if recursive else "*"
        for file_path in path.glob(pattern):
            if file_path.is_file() and file_path.suffix.lower() in [
                ".txt", ".md", ".markdown", ".json", ".jsonl", ".csv"
            ]:
                try:
                    documents.extend(self.load_file(str(file_path)))
                except Exception as e:
                    print(f"載入 {file_path} 失敗: {e}")

        return documents

    def _load_text(self, path: Path) -> list[Document]:
        """載入純文字檔"""
        content = path.read_text(encoding="utf-8")
        return [Document(
            content=content,
            source=str(path),
            metadata={"format": "text"},
        )]

    def _load_json(self, path: Path) -> list[Document]:
        """載入 JSON 檔"""
        data = json.loads(path.read_text(encoding="utf-8"))

        if isinstance(data, list):
            return [
                Document(
                    content=item.get(self.content_key, str(item)),
                    source=str(path),
                    metadata={"format": "json", "index": i},
                )
                for i, item in enumerate(data)
            ]
        else:
            return [Document(
                content=data.get(self.content_key, str(data)),
                source=str(path),
                metadata={"format": "json"},
            )]

    def _load_jsonl(self, path: Path) -> list[Document]:
        """載入 JSON Lines 檔"""
        documents = []
        for i, line in enumerate(path.read_text(encoding="utf-8").splitlines()):
            if line.strip():
                item = json.loads(line)
                documents.append(Document(
                    content=item.get(self.content_key, str(item)),
                    source=str(path),
                    metadata={"format": "jsonl", "line": i},
                ))
        return documents

    def _load_csv(self, path: Path) -> list[Document]:
        """載入 CSV 檔"""
        documents = []
        with open(path, encoding="utf-8") as f:
            reader = csv.DictReader(f)
            for i, row in enumerate(reader):
                content = row.get(self.content_key, "")
                if content:
                    documents.append(Document(
                        content=content,
                        source=str(path),
                        metadata={"format": "csv", "row": i, **row},
                    ))
        return documents
```

### 6.3 Chunk 資料結構

```python
from dataclasses import dataclass, field
import hashlib
import uuid


@dataclass
class Chunk:
    """文件切片"""

    content: str
    metadata: dict
    source: str
    chunk_index: int
    total_chunks: int
    tags: list[str] = field(default_factory=list)
    categories: list[str] = field(default_factory=list)
    content_hash: str = ""

    @property
    def id(self) -> str:
        """唯一 ID（包含 content hash）"""
        return f"{self.source}::{self.content_hash}::chunk_{self.chunk_index}"

    @property
    def point_id(self) -> str:
        """Qdrant 相容的 UUID（基於 ID 的確定性 UUID）"""
        hash_bytes = hashlib.sha256(self.id.encode()).digest()[:16]
        return str(uuid.UUID(bytes=hash_bytes))
```

### 6.4 FixedSizeChunker

```python
class FixedSizeChunker:
    """固定大小切分器"""

    def __init__(
        self,
        chunk_size: int = 1000,
        chunk_overlap: int = 200,
        separator: str = "\n",
    ):
        if chunk_overlap >= chunk_size:
            raise ValueError("chunk_overlap 必須小於 chunk_size")

        self.chunk_size = chunk_size
        self.chunk_overlap = chunk_overlap
        self.separator = separator

    def _find_split_point(self, text: str, start: int, end: int) -> int:
        """找最佳切分點（優先在換行或句號處切分）"""
        search_start = max(start, end - 100)

        # 尋找換行符
        last_sep = text.rfind(self.separator, search_start, end)
        if last_sep > start:
            return last_sep + len(self.separator)

        # 尋找句號
        for punct in ["。", ".", "！", "!", "？", "?"]:
            last_punct = text.rfind(punct, search_start, end)
            if last_punct > start:
                return last_punct + 1

        # 尋找空格
        last_space = text.rfind(" ", search_start, end)
        if last_space > start:
            return last_space + 1

        return end

    def chunk(self, document: Document) -> list[Chunk]:
        """切分文件"""
        text = document.content.strip()

        if not text:
            return []

        if len(text) <= self.chunk_size:
            return [Chunk(
                content=text,
                metadata=document.metadata.copy(),
                source=document.source,
                chunk_index=0,
                total_chunks=1,
                content_hash=document.content_hash,
            )]

        chunks = []
        start = 0

        while start < len(text):
            end = min(start + self.chunk_size, len(text))

            if end < len(text):
                end = self._find_split_point(text, start, end)

            chunk_text = text[start:end].strip()

            if chunk_text:
                chunks.append(Chunk(
                    content=chunk_text,
                    metadata=document.metadata.copy(),
                    source=document.source,
                    chunk_index=len(chunks),
                    total_chunks=0,
                    content_hash=document.content_hash,
                ))

            if end >= len(text):
                break

            start = end - self.chunk_overlap
            if start <= 0:
                break

        # 更新 total_chunks
        for chunk in chunks:
            chunk.total_chunks = len(chunks)

        return chunks
```

### 6.5 內容去重機制

<!-- CODE-PATTERN: content-deduplication -->

**設計原理**：

1. **Document Hash**：`SHA-256(content)[:16]`
2. **Chunk ID**：`{source}::{content_hash}::chunk_{index}`
3. **Point ID**：`UUID(SHA-256(chunk_id)[:16])` - Qdrant 相容

**去重流程**：

```
載入檔案 → 計算 content_hash
    ↓
查詢 Qdrant：該 source 是否存在？
    ↓
├─ 不存在 → 新增 chunks
├─ 存在且 hash 相同 → 跳過（內容未變）
└─ 存在但 hash 不同 → 刪除舊 chunks → 新增新 chunks
```

---

## 7. 向量儲存層（Qdrant）

<!-- AI-SECTION: vector_store -->
<!-- IMPLEMENTS: QdrantStore -->
<!-- DEPENDS-ON: embedding/base, bm25/bm25_store -->

### 7.1 QdrantStore 核心結構

```python
from datetime import datetime
from typing import Any, Optional

from qdrant_client import QdrantClient
from qdrant_client.http.models import (
    Distance,
    FieldCondition,
    Filter,
    MatchAny,
    MatchText,
    MatchValue,
    PointStruct,
    TextIndexParams,
    TokenizerType,
    VectorParams,
)

from ..config import get_settings
from ..document import Chunk
from ..embedding.base import EmbeddingProvider


class QdrantStore:
    """Qdrant 向量儲存"""

    def __init__(
        self,
        embedding_provider: EmbeddingProvider,
        collection_name: Optional[str] = None,
        host: Optional[str] = None,
        port: Optional[int] = None,
        bm25_store: Optional["BM25Store"] = None,
    ):
        settings = get_settings()

        self.embedding_provider = embedding_provider
        self.collection_name = collection_name or settings.qdrant_collection_name
        self._host = host or settings.qdrant_host
        self._port = port or settings.qdrant_port
        self.bm25_store = bm25_store

        qdrant_url = f"http://{self._host}:{self._port}"
        self._client = QdrantClient(url=qdrant_url)

    def ensure_collection(self, with_fulltext_index: bool = True) -> bool:
        """確保 collection 存在"""
        collections = self._client.get_collections().collections
        exists = any(c.name == self.collection_name for c in collections)

        if not exists:
            self._client.create_collection(
                collection_name=self.collection_name,
                vectors_config=VectorParams(
                    size=self.embedding_provider.dimension,
                    distance=Distance.COSINE,
                ),
            )

            if with_fulltext_index:
                self._create_fulltext_index()

            return True
        return False

    def _create_fulltext_index(self):
        """建立全文索引"""
        try:
            self._client.create_payload_index(
                collection_name=self.collection_name,
                field_name="content",
                field_schema=TextIndexParams(
                    type="text",
                    tokenizer=TokenizerType.MULTILINGUAL,
                    min_token_len=2,
                    max_token_len=20,
                    lowercase=True,
                ),
            )
        except Exception:
            pass  # 索引可能已存在
```

### 7.2 Payload 結構設計

```python
payload = {
    # 核心欄位
    "content": chunk.content,
    "source": chunk.source,
    "chunk_index": chunk.chunk_index,
    "total_chunks": chunk.total_chunks,

    # 標籤與分類
    "tags": chunk.tags,
    "categories": chunk.categories,

    # 去重用
    "content_hash": chunk.content_hash,
    "ingested_at": "2026-01-20 10:00:00",

    # 階層式結構
    "level": "child",  # "parent" 或 "child"
    "parent_id": "uuid-of-parent",  # 僅 child 有

    # 自訂 metadata
    **chunk.metadata,
}
```

### 7.3 智慧更新機制

```python
def add_chunks_smart(
    self,
    chunks: list[Chunk],
    batch_size: int = 100,
    show_progress: bool = True,
) -> dict[str, int]:
    """
    智慧新增 chunks（含去重）

    Returns:
        {"added": n, "updated": n, "skipped": n}
    """
    if not chunks:
        return {"added": 0, "updated": 0, "skipped": 0}

    self.ensure_collection()

    # 依 source 分組
    chunks_by_source: dict[str, list[Chunk]] = {}
    for chunk in chunks:
        if chunk.source not in chunks_by_source:
            chunks_by_source[chunk.source] = []
        chunks_by_source[chunk.source].append(chunk)

    stats = {"added": 0, "updated": 0, "skipped": 0}
    chunks_to_add: list[Chunk] = []

    for source, source_chunks in chunks_by_source.items():
        new_hash = source_chunks[0].content_hash
        existing_hash = self.get_source_hash(source)

        if existing_hash == new_hash:
            # 內容未變，跳過
            stats["skipped"] += len(source_chunks)
        elif existing_hash is not None:
            # 內容變更，刪除舊的並新增新的
            self.delete_by_source(source)
            chunks_to_add.extend(source_chunks)
            stats["updated"] += len(source_chunks)
        else:
            # 新來源，新增
            chunks_to_add.extend(source_chunks)
            stats["added"] += len(source_chunks)

    # 批次新增
    if chunks_to_add:
        for i in range(0, len(chunks_to_add), batch_size):
            batch = chunks_to_add[i:i + batch_size]
            self._add_batch(batch)

    return stats

def _add_batch(self, chunks: list[Chunk]) -> int:
    """批次新增 chunks"""
    texts = [chunk.content for chunk in chunks]
    embeddings = self.embedding_provider.embed_texts(texts)

    ingested_at = datetime.now().strftime("%Y-%m-%d %H:%M:%S")

    points = []
    for chunk, embedding in zip(chunks, embeddings):
        payload = {
            "content": chunk.content,
            "source": chunk.source,
            "chunk_index": chunk.chunk_index,
            "total_chunks": chunk.total_chunks,
            "tags": chunk.tags,
            "categories": chunk.categories,
            "content_hash": chunk.content_hash,
            "ingested_at": ingested_at,
            **chunk.metadata,
        }

        # 階層式支援
        if hasattr(chunk, "level"):
            payload["level"] = chunk.level
        if hasattr(chunk, "parent_id") and chunk.parent_id:
            payload["parent_id"] = chunk.parent_id

        points.append(PointStruct(
            id=chunk.point_id,
            vector=embedding,
            payload=payload,
        ))

    self._client.upsert(
        collection_name=self.collection_name,
        points=points,
    )

    return len(points)
```

### 7.4 搜尋方法

```python
def search(
    self,
    query: str,
    top_k: int = 5,
    score_threshold: Optional[float] = None,
    tags: Optional[list[str]] = None,
    source: Optional[str] = None,
) -> list[dict[str, Any]]:
    """向量相似度搜尋"""
    query_embedding = self.embedding_provider.embed_query(query)

    # 建構過濾條件
    conditions = []
    if tags:
        conditions.append(FieldCondition(
            key="tags",
            match=MatchAny(any=tags),
        ))
    if source:
        conditions.append(FieldCondition(
            key="source",
            match=MatchValue(value=source),
        ))

    query_filter = Filter(must=conditions) if conditions else None

    results = self._client.query_points(
        collection_name=self.collection_name,
        query=query_embedding,
        limit=top_k,
        score_threshold=score_threshold,
        query_filter=query_filter,
    ).points

    return [
        {
            "id": str(result.id),
            "content": result.payload.get("content", ""),
            "source": result.payload.get("source", ""),
            "score": result.score,
            "tags": result.payload.get("tags", []),
            "categories": result.payload.get("categories", []),
            "metadata": {
                k: v for k, v in result.payload.items()
                if k not in ["content", "source", "tags", "categories"]
            },
        }
        for result in results
    ]
```

---

# Part 3: 進階檢索策略

## 8. 混合檢索與 RRF 融合

<!-- AI-SECTION: hybrid_search -->
<!-- IMPLEMENTS: hybrid_search, _rrf_fusion -->

### 8.1 混合搜尋架構

```
        Query
         │
    ┌────┴─────┐
    ▼          ▼
┌─────────┐ ┌─────────┐
│ Vector  │ │ Keyword │
│ Search  │ │ Search  │
│(Qdrant) │ │ (BM25)  │
└────┬────┘ └────┬────┘
     │          │
     └────┬─────┘
          ▼
    ┌──────────┐
    │   RRF    │  ← Reciprocal Rank Fusion
    │  Fusion  │     score = Σ weight / (k + rank)
    └────┬─────┘
         ▼
    ┌──────────┐
    │ Ranked   │
    │ Results  │
    └──────────┘
```

### 8.2 RRF 融合實作

```python
def _rrf_fusion(
    self,
    vector_results: list[dict],
    keyword_results: list[dict],
    vector_weight: float = 0.7,
    keyword_weight: float = 0.3,
    k: int = 60,
) -> list[dict[str, Any]]:
    """
    Reciprocal Rank Fusion 融合向量與關鍵字搜尋結果

    RRF score = sum(weight / (k + rank))

    Args:
        vector_results: 向量搜尋結果
        keyword_results: 關鍵字搜尋結果
        vector_weight: 向量搜尋權重（0-1）
        keyword_weight: 關鍵字搜尋權重（0-1）
        k: RRF 常數（預設 60，參考原始論文）
    """
    scores: dict[str, float] = {}
    result_data: dict[str, dict] = {}

    # 計算向量搜尋分數
    for rank, result in enumerate(vector_results):
        doc_id = result["id"]
        scores[doc_id] = scores.get(doc_id, 0) + vector_weight / (k + rank + 1)
        if doc_id not in result_data:
            result_data[doc_id] = result

    # 計算關鍵字搜尋分數
    for rank, result in enumerate(keyword_results):
        doc_id = result["id"]
        scores[doc_id] = scores.get(doc_id, 0) + keyword_weight / (k + rank + 1)
        if doc_id not in result_data:
            result_data[doc_id] = result

    # 依分數排序
    sorted_ids = sorted(scores.keys(), key=lambda x: scores[x], reverse=True)

    # 建構最終結果
    fused_results = []
    for doc_id in sorted_ids:
        result = result_data[doc_id].copy()
        result["rrf_score"] = scores[doc_id]
        fused_results.append(result)

    return fused_results
```

### 8.3 混合搜尋方法

```python
def hybrid_search(
    self,
    query: str,
    top_k: int = 5,
    vector_weight: float = 0.7,
    keyword_weight: float = 0.3,
    score_threshold: Optional[float] = None,
    tags: Optional[list[str]] = None,
    source: Optional[str] = None,
    use_bm25: bool = True,
) -> list[dict[str, Any]]:
    """混合搜尋（向量 + 關鍵字）"""

    # 取得更多結果以便融合
    fetch_k = top_k * 3

    # 向量搜尋
    vector_results = self.search(
        query=query,
        top_k=fetch_k,
        score_threshold=score_threshold,
        tags=tags,
        source=source,
    )

    # 關鍵字搜尋
    if use_bm25 and self.bm25_store:
        keyword_results = self._bm25_search(
            query=query,
            top_k=fetch_k,
            tags=tags,
            source=source,
        )
    else:
        keyword_results = self.keyword_search(
            query=query,
            top_k=fetch_k,
            tags=tags,
            source=source,
        )

    # RRF 融合
    fused = self._rrf_fusion(
        vector_results=vector_results,
        keyword_results=keyword_results,
        vector_weight=vector_weight,
        keyword_weight=keyword_weight,
    )

    return fused[:top_k]
```

### 8.4 BM25 關鍵字搜尋

<!-- CODE-PATTERN: bm25-scoring -->

**BM25 公式**：

```
score(D, Q) = Σ IDF(qi) * (tf(qi, D) * (k1 + 1)) /
                          (tf(qi, D) + k1 * (1 - b + b * |D| / avgdl))

其中：
- k1 = 1.5 (詞頻飽和度)
- b = 0.75 (文件長度正規化)
- IDF = log((N - df + 0.5) / (df + 0.5) + 1)
```

**SQLite 儲存結構**：

```sql
-- 文件表
CREATE TABLE documents (
    id TEXT PRIMARY KEY,
    content TEXT NOT NULL,
    tokens TEXT NOT NULL,  -- JSON 陣列
    token_count INTEGER NOT NULL,
    source TEXT,
    created_at TEXT NOT NULL
);

-- 詞彙表（倒排索引）
CREATE TABLE vocabulary (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    term TEXT UNIQUE NOT NULL,
    df INTEGER DEFAULT 0,  -- document frequency
    idf REAL DEFAULT 0     -- 預計算的 IDF
);

-- 詞-文件頻率表
CREATE TABLE term_docs (
    term_id INTEGER NOT NULL,
    doc_id TEXT NOT NULL,
    tf INTEGER NOT NULL,
    PRIMARY KEY (term_id, doc_id)
);

-- 索引
CREATE INDEX idx_term_docs_doc ON term_docs(doc_id);
CREATE INDEX idx_vocabulary_term ON vocabulary(term);
```

---

## 9. 階層式檢索

<!-- AI-SECTION: hierarchical_retrieval -->
<!-- IMPLEMENTS: HierarchicalChunker, search_with_parent -->

### 9.1 Parent-Child 結構設計

```
Document (原始文件 10000 字)
    │
    ├─ Parent Chunk 1 (2000 字) ← 完整上下文
    │  ├─ Child Chunk 1.1 (500 字) ← 用於檢索
    │  ├─ Child Chunk 1.2 (500 字) ← 用於檢索
    │  └─ Child Chunk 1.3 (500 字) ← 用於檢索
    │
    └─ Parent Chunk 2 (2000 字)
       ├─ Child Chunk 2.1 (500 字)
       └─ Child Chunk 2.2 (500 字)
```

**設計理念**：

- **Child chunks**：小區塊用於精確檢索（500 字）
- **Parent chunks**：大區塊提供完整上下文（2000 字）
- **檢索時**：搜尋 child，返回對應 parent 內容

### 9.2 HierarchicalChunker 實作

```python
from dataclasses import dataclass
from typing import Optional


@dataclass
class HierarchicalChunk(Chunk):
    """階層式 Chunk"""

    level: str = "child"  # "parent" 或 "child"
    parent_id: Optional[str] = None
    parent_content: Optional[str] = None


class HierarchicalChunker:
    """階層式切分器"""

    def __init__(
        self,
        parent_chunk_size: int = 2000,
        child_chunk_size: int = 500,
        parent_overlap: int = 200,
        child_overlap: int = 100,
    ):
        if child_chunk_size >= parent_chunk_size:
            raise ValueError("child_chunk_size 必須小於 parent_chunk_size")

        self.parent_chunker = FixedSizeChunker(
            chunk_size=parent_chunk_size,
            chunk_overlap=parent_overlap,
        )
        self.child_chunker = FixedSizeChunker(
            chunk_size=child_chunk_size,
            chunk_overlap=child_overlap,
        )

    def chunk(self, document: Document) -> list[HierarchicalChunk]:
        """建立階層式 chunks"""
        all_chunks: list[HierarchicalChunk] = []

        # Step 1: 建立 parent chunks
        parent_chunks_raw = self.parent_chunker.chunk(document)

        for parent_idx, parent_raw in enumerate(parent_chunks_raw):
            # 建立 parent
            parent_chunk = HierarchicalChunk(
                content=parent_raw.content,
                metadata=parent_raw.metadata.copy(),
                source=parent_raw.source,
                chunk_index=parent_idx,
                total_chunks=len(parent_chunks_raw),
                content_hash=document.content_hash,
                level="parent",
                parent_id=None,
            )
            all_chunks.append(parent_chunk)

            # Step 2: 從 parent 建立 child chunks
            parent_doc = Document(
                content=parent_raw.content,
                source=parent_raw.source,
                metadata=parent_raw.metadata.copy(),
                content_hash=document.content_hash,
            )

            child_chunks_raw = self.child_chunker.chunk(parent_doc)

            for child_idx, child_raw in enumerate(child_chunks_raw):
                child_chunk = HierarchicalChunk(
                    content=child_raw.content,
                    metadata=child_raw.metadata.copy(),
                    source=child_raw.source,
                    chunk_index=parent_idx * len(child_chunks_raw) + child_idx,
                    total_chunks=0,
                    content_hash=document.content_hash,
                    level="child",
                    parent_id=parent_chunk.point_id,
                )
                all_chunks.append(child_chunk)

        return all_chunks
```

### 9.3 階層式搜尋

```python
def search_with_parent(
    self,
    query: str,
    top_k: int = 5,
    score_threshold: Optional[float] = None,
    tags: Optional[list[str]] = None,
    source: Optional[str] = None,
    hybrid: bool = False,
    vector_weight: float = 0.7,
    keyword_weight: float = 0.3,
) -> list[dict[str, Any]]:
    """階層式搜尋（child + parent context）"""

    query_embedding = self.embedding_provider.embed_query(query)

    # 只搜尋 child level
    conditions = [
        FieldCondition(key="level", match=MatchValue(value="child"))
    ]

    if tags:
        conditions.append(FieldCondition(key="tags", match=MatchAny(any=tags)))
    if source:
        conditions.append(FieldCondition(key="source", match=MatchValue(value=source)))

    query_filter = Filter(must=conditions)

    # 搜尋 child chunks
    results = self._client.query_points(
        collection_name=self.collection_name,
        query=query_embedding,
        limit=top_k,
        score_threshold=score_threshold,
        query_filter=query_filter,
    ).points

    child_results = [
        {
            "id": str(result.id),
            "content": result.payload.get("content", ""),
            "source": result.payload.get("source", ""),
            "score": result.score,
            "tags": result.payload.get("tags", []),
            "categories": result.payload.get("categories", []),
            "metadata": {k: v for k, v in result.payload.items()
                        if k not in ["content", "source", "tags", "categories"]},
        }
        for result in results
    ]

    # 取得對應的 parent content
    for result in child_results:
        parent_id = result["metadata"].get("parent_id")
        if parent_id:
            parent = self.get_by_id(parent_id)
            if parent:
                result["parent_content"] = parent["content"]
            else:
                result["parent_content"] = None
        else:
            result["parent_content"] = None

    return child_results
```

---

## 10. LLM Reranking

<!-- AI-SECTION: llm_reranking -->
<!-- IMPLEMENTS: LLMReranker -->

### 10.1 Reranker 設計

```python
from dataclasses import dataclass


@dataclass
class RerankScore:
    """重排分數"""
    index: int
    score: float
    reason: str


RERANK_SYSTEM_PROMPT = """你是一個文件相關性評估專家。請為每個文件評分：
- 10: 完全相關，直接回答問題
- 7-9: 高度相關，包含重要資訊
- 4-6: 部分相關
- 1-3: 低度相關
- 0: 完全不相關

請以 JSON 格式回傳結果。"""


class LLMReranker:
    """LLM 重排器"""

    def __init__(self, llm: LLMProvider, top_k: int = 5):
        self.llm = llm
        self.top_k = top_k

    def rerank_with_usage(
        self,
        question: str,
        results: list[RetrievalResult],
        top_k: Optional[int] = None,
    ) -> tuple[list[tuple[RetrievalResult, RerankScore]], TokenUsage]:
        """重排並追蹤使用量"""

        if not results:
            return [], TokenUsage()

        k = top_k or self.top_k

        # 格式化文件
        docs_text = ""
        for i, result in enumerate(results):
            content_preview = result.content[:500]
            if len(result.content) > 500:
                content_preview += "..."
            docs_text += f"\n--- 文件 {i} ---\n來源: {result.source}\n內容: {content_preview}\n"

        prompt = f"""問題: {question}

請評估以下文件的相關性（0-10 分）：

{docs_text}

請以 JSON 格式回傳：
```json
{{
  "scores": [
    {{"index": 0, "score": <分數>, "reason": "<理由>"}},
    ...
  ]
}}
```"""

        try:
            response = self.llm.generate(
                prompt=prompt,
                system_prompt=RERANK_SYSTEM_PROMPT,
                temperature=0.1,  # 低溫度確保一致性
                max_tokens=1000,
            )

            scores = self._parse_scores(response.content, len(results))
            usage = TokenUsage.from_dict(response.usage, model=response.model)

            # 依分數排序
            ranked = sorted(
                zip(results, scores),
                key=lambda x: x[1].score,
                reverse=True,
            )

            return ranked[:k], usage

        except Exception as e:
            # 失敗時返回預設分數
            return [
                (r, RerankScore(index=i, score=5.0, reason=f"評分失敗: {e}"))
                for i, r in enumerate(results)
            ][:k], TokenUsage()
```

### 10.2 成本權衡

<!-- PERFORMANCE: rerank_tradeoff -->

| 項目 | 值 |
|------|-----|
| 額外 Token 消耗 | ~500-1000 tokens/次 |
| 延遲增加 | ~500-1000ms |
| 精準度提升 | 約 10-20% |
| 適用場景 | 需要高精準度的場景 |

**使用建議**：

- 需要高精準度時啟用
- 成本敏感場景禁用
- 初始 fetch_k 設為 top_k * 2

---

# Part 4: RAG 管道整合

## 11. RAGChain 核心流程

<!-- AI-SECTION: rag_chain -->
<!-- IMPLEMENTS: RAGChain -->

### 11.1 RAGChain 類別結構

```python
@dataclass
class RAGResponse:
    """RAG 回應"""
    answer: str
    sources: list[RetrievalResult]
    query: str
    rerank_scores: list[RerankScore] = field(default_factory=list)
    filtered_count: int = 0
    usage: Optional[UsageSummary] = None


class RAGChain:
    """RAG 管道"""

    def __init__(
        self,
        retriever: Retriever,
        llm: LLMProvider,
        system_prompt: Optional[str] = None,
        top_k: Optional[int] = None,
        reranker: Optional[LLMReranker] = None,
    ):
        self.retriever = retriever
        self.llm = llm
        self.system_prompt = system_prompt or RAG_SYSTEM_PROMPT
        self.top_k = top_k
        self.reranker = reranker
```

### 11.2 完整查詢流程

```python
def query(
    self,
    question: str,
    top_k: Optional[int] = None,
    score_threshold: Optional[float] = None,
    tags: Optional[list[str]] = None,
    source: Optional[str] = None,
    rerank: bool = False,
    hybrid: bool = False,
    vector_weight: float = 0.7,
    keyword_weight: float = 0.3,
    hierarchical: bool = False,
    category_boost: bool = False,
) -> RAGResponse:
    """
    完整 RAG 查詢流程

    Args:
        question: 使用者問題
        top_k: 檢索數量
        score_threshold: 相似度閾值
        tags: 標籤過濾
        source: 來源過濾
        rerank: 是否啟用 LLM 重排
        hybrid: 是否使用混合搜尋
        vector_weight: 向量搜尋權重
        keyword_weight: 關鍵字搜尋權重
        hierarchical: 是否使用階層式檢索
        category_boost: 是否啟用分類增強
    """
    k = top_k or self.top_k

    # 若啟用重排，多取一些結果
    fetch_k = k * 2 if rerank and self.reranker else k

    # Step 1: 檢索
    if hierarchical:
        results, embedding_usage = self.retriever.retrieve_hierarchical_with_usage(
            query=question,
            top_k=fetch_k,
            score_threshold=score_threshold,
            tags=tags,
            source=source,
            hybrid=hybrid,
            vector_weight=vector_weight,
            keyword_weight=keyword_weight,
            category_boost=category_boost,
        )
    else:
        results, embedding_usage = self.retriever.retrieve_with_usage(
            query=question,
            top_k=fetch_k,
            score_threshold=score_threshold,
            tags=tags,
            source=source,
            hybrid=hybrid,
            vector_weight=vector_weight,
            keyword_weight=keyword_weight,
            category_boost=category_boost,
        )

    # 處理無結果
    if not results:
        return RAGResponse(
            answer=build_no_results_response(question),
            sources=[],
            query=question,
            usage=UsageSummary(
                llm_usage=TokenUsage(),
                rerank_usage=None,
                embedding_usage=embedding_usage,
                costs=calculate_total_cost(
                    llm_usage=TokenUsage(),
                    rerank_usage=None,
                    embedding_usage=embedding_usage,
                ),
            ),
        )

    # Step 2: 重排（可選）
    rerank_scores = []
    rerank_usage: Optional[TokenUsage] = None
    if rerank and self.reranker:
        ranked, rerank_usage = self.reranker.rerank_with_usage(question, results, top_k=k)
        results = [r for r, _ in ranked]
        rerank_scores = [s for _, s in ranked]

    # 限制到 top_k
    final_results = results[:k]
    filtered_count = len(results) - len(final_results)

    # Step 3: 格式化上下文
    results_dict = [
        {
            "content": r.content,
            "source": r.source,
            "score": r.score,
            "tags": r.tags,
            "parent_content": r.parent_content,
        }
        for r in final_results
    ]
    context = format_context(
        results_dict,
        max_results=k,
        use_parent_context=hierarchical,
    )

    # Step 4: LLM 生成
    prompt = build_rag_prompt(question, context)
    response = self.llm.generate(
        prompt=prompt,
        system_prompt=self.system_prompt,
    )

    # Step 5: 追蹤使用量
    llm_usage = TokenUsage.from_dict(response.usage, model=response.model)

    usage_summary = UsageSummary(
        llm_usage=llm_usage,
        rerank_usage=rerank_usage,
        embedding_usage=embedding_usage,
        costs=calculate_total_cost(
            llm_usage=llm_usage,
            rerank_usage=rerank_usage,
            embedding_usage=embedding_usage,
        ),
    )

    return RAGResponse(
        answer=response.content,
        sources=final_results,
        query=question,
        rerank_scores=rerank_scores[:len(final_results)] if rerank_scores else [],
        filtered_count=filtered_count,
        usage=usage_summary,
    )
```

---

## 12. Prompt 模板設計

<!-- AI-SECTION: prompt_templates -->

### 12.1 系統提示詞

```python
RAG_SYSTEM_PROMPT = """你是一個專業的諮詢記錄檢索助手。

你的任務是：
1. 仔細閱讀提供的諮詢記錄參考資料
2. 根據參考資料回答使用者的問題
3. 如果參考資料中沒有相關資訊，請明確告知

回答規則：
- 只根據提供的參考資料回答，不要編造內容
- 如有需要，適當引用來源
- 回答要清晰、有條理
- 使用繁體中文"""
```

### 12.2 查詢提示詞模板

```python
def build_rag_prompt(question: str, context: str) -> str:
    """建構 RAG 查詢提示詞"""
    return f"""以下是與問題相關的諮詢記錄參考資料：

{context}

---

使用者問題：{question}

請根據上述參考資料回答問題："""


def format_context(
    results: list[dict],
    max_results: int = 5,
    use_parent_context: bool = False,
) -> str:
    """格式化檢索結果為上下文"""
    context_parts = []

    for i, result in enumerate(results[:max_results]):
        score = result.get("score", 0)
        source = result.get("source", "")
        content = result.get("content", "")
        parent_content = result.get("parent_content", "")

        part = f"[資料 {i+1}] (相似度: {score:.2%}) 來自: {source}\n"

        if use_parent_context and parent_content:
            part += f"完整背景:\n{parent_content}\n\n精確片段:\n{content}"
        else:
            part += f"內容:\n{content}"

        context_parts.append(part)

    return "\n\n---\n\n".join(context_parts)


def build_no_results_response(question: str) -> str:
    """建構無結果回應"""
    return f"抱歉，在諮詢記錄中找不到與「{question}」相關的資訊。請嘗試使用不同的關鍵字或問題描述。"
```

---

# Part 5: 成本與效能管理

## 13. Token 使用量追蹤

<!-- AI-SECTION: usage_tracking -->
<!-- IMPLEMENTS: TokenUsage, UsageSummary -->

### 13.1 資料結構

```python
from dataclasses import dataclass, field
from typing import Optional


@dataclass
class TokenUsage:
    """單次 API 呼叫的 Token 使用量"""

    prompt_tokens: int = 0
    completion_tokens: int = 0
    total_tokens: int = 0
    model: str = ""

    @classmethod
    def from_dict(cls, data: Optional[dict], model: str = "") -> "TokenUsage":
        """從 LLM response 建立"""
        if not data:
            return cls(model=model)
        return cls(
            prompt_tokens=data.get("prompt_tokens", 0),
            completion_tokens=data.get("completion_tokens", 0),
            total_tokens=data.get("total_tokens", 0),
            model=model,
        )


@dataclass
class CostBreakdown:
    """成本分解"""

    llm_cost: float = 0.0
    rerank_cost: float = 0.0
    embedding_cost: float = 0.0
    total_cost: float = 0.0


@dataclass
class UsageSummary:
    """完整使用量摘要"""

    llm_usage: TokenUsage = field(default_factory=TokenUsage)
    rerank_usage: Optional[TokenUsage] = None
    embedding_usage: TokenUsage = field(default_factory=TokenUsage)
    costs: CostBreakdown = field(default_factory=CostBreakdown)

    @property
    def total_tokens(self) -> int:
        """總 Token 數"""
        total = self.llm_usage.total_tokens + self.embedding_usage.total_tokens
        if self.rerank_usage:
            total += self.rerank_usage.total_tokens
        return total
```

---

## 14. 定價表與成本計算

<!-- AI-SECTION: pricing -->
<!-- IMPLEMENTS: calculate_total_cost -->

### 14.1 定價表

```python
# 價格：每 1M tokens（USD）
PRICING_TABLE: dict[str, dict[str, float]] = {
    # OpenAI Models
    "gpt-4o": {"input": 2.50, "output": 10.00},
    "gpt-4o-mini": {"input": 0.15, "output": 0.60},
    "gpt-4-turbo": {"input": 10.00, "output": 30.00},
    "gpt-3.5-turbo": {"input": 0.50, "output": 1.50},

    # OpenAI Embedding
    "text-embedding-3-small": {"input": 0.02, "output": 0.0},
    "text-embedding-3-large": {"input": 0.13, "output": 0.0},

    # Gemini Models
    "gemini-2.0-flash": {"input": 0.10, "output": 0.40},
    "gemini-1.5-flash": {"input": 0.075, "output": 0.30},
    "gemini-1.5-pro": {"input": 1.25, "output": 5.00},

    # Gemini Embedding（免費）
    "text-embedding-004": {"input": 0.0, "output": 0.0},
    "embedding-001": {"input": 0.0, "output": 0.0},
}
```

### 14.2 成本計算

```python
def calculate_token_cost(usage: TokenUsage) -> float:
    """計算 Token 成本"""
    pricing = get_model_pricing(usage.model)

    input_cost = (usage.prompt_tokens / 1_000_000) * pricing["input"]
    output_cost = (usage.completion_tokens / 1_000_000) * pricing["output"]

    return input_cost + output_cost


def calculate_total_cost(
    llm_usage: TokenUsage,
    rerank_usage: Optional[TokenUsage],
    embedding_usage: TokenUsage,
) -> CostBreakdown:
    """計算完整成本分解"""

    llm_cost = calculate_token_cost(llm_usage)
    rerank_cost = calculate_token_cost(rerank_usage) if rerank_usage else 0.0
    embedding_cost = (embedding_usage.total_tokens / 1_000_000) * \
                     get_model_pricing(embedding_usage.model)["input"]

    return CostBreakdown(
        llm_cost=llm_cost,
        rerank_cost=rerank_cost,
        embedding_cost=embedding_cost,
        total_cost=llm_cost + rerank_cost + embedding_cost,
    )
```

---

## 15. 效能優化最佳實踐

<!-- AI-SECTION: performance_optimization -->
<!-- PERFORMANCE: best_practices -->

### 15.1 效能指標基準

| 指標 | 目標值 | 說明 |
|------|--------|------|
| 單次查詢延遲 | 1-3 秒 | 含 rerank 約 2 秒 |
| Embedding 快取命中率 | 60-80% | 重複匯入場景 |
| 向量化吞吐量 | 100-200 chunks/sec | batch_size=100 |
| 搜尋延遲 | 50-200ms | 10K 文件規模 |
| 記憶體佔用 | < 1GB | Qdrant + Python |

### 15.2 優化策略清單

| 優化項目 | 實作方式 | 預期效果 |
|----------|----------|----------|
| **Embedding 快取** | SQLite + SHA-256 hash | API 成本 -60~80% |
| **智慧去重** | content_hash 比對 | 重複匯入時間 -90% |
| **批量處理** | batch_size=100 | 減少 API 往返 |
| **速率限制** | 指數退避重試 | 成功率 ~100% |
| **混合搜尋** | RRF 融合 | 精準度 +20-30% |
| **階層式檢索** | Parent-Child | 上下文完整性 +20% |
| **維度調整** | 1536 vs 3072 | 儲存/速度權衡 |

### 15.3 配置建議

| 場景 | 配置 | 延遲 | 成本/query |
|------|------|------|------------|
| **快速查詢** | `{top_k:5}` | ~200ms | ~$0.001 |
| **精確查詢** | `{hybrid:true, rerank:true}` | ~1.5s | ~$0.01 |
| **完整回答** | `{hierarchical:true}` | ~600ms | ~$0.002 |
| **最佳效果** | `{hybrid:true, hierarchical:true, rerank:true}` | ~2s | ~$0.015 |

---

# Part 6: 介面設計

## 16. REST API 設計（FastAPI）

<!-- AI-SECTION: rest_api -->
<!-- IMPLEMENTS: QueryRequest, QueryResponse -->

### 16.1 端點結構

| 方法 | 路徑 | 功能 |
|------|------|------|
| POST | `/api/query` | RAG 查詢 |
| GET | `/api/stats` | 系統統計 |
| GET | `/api/cache/stats` | 快取統計 |
| GET | `/health` | 健康檢查 |

### 16.2 查詢請求 Schema

```python
from pydantic import BaseModel, Field
from typing import Optional


class QueryRequest(BaseModel):
    """查詢請求"""

    question: str = Field(..., description="查詢問題")
    top_k: int = Field(default=5, ge=1, le=20, description="檢索數量")
    stream: bool = Field(default=False, description="是否串流回應")

    # 搜尋模式
    hybrid: bool = Field(default=False, description="混合搜尋")
    vector_weight: float = Field(default=0.7, ge=0, le=1, description="向量權重")
    keyword_weight: float = Field(default=0.3, ge=0, le=1, description="關鍵字權重")
    hierarchical: bool = Field(default=False, description="階層式檢索")
    rerank: bool = Field(default=False, description="LLM 重排")
    category_boost: bool = Field(default=False, description="分類增強")

    # 過濾條件
    tags: Optional[list[str]] = Field(default=None, description="標籤過濾")
    source: Optional[str] = Field(default=None, description="來源過濾")
    score_threshold: Optional[float] = Field(default=None, description="相似度閾值")

    # 自訂
    system_prompt: Optional[str] = Field(default=None, description="自訂系統提示詞")
```

### 16.3 查詢回應 Schema

```python
class SourceInfo(BaseModel):
    """來源資訊"""

    content: str
    source: str
    score: float
    tags: list[str] = []
    categories: list[str] = []
    category_boost: Optional[float] = None
    rerank_score: Optional[float] = None
    parent_content: Optional[str] = None


class UsageSummaryInfo(BaseModel):
    """使用量摘要"""

    llm_usage: dict
    rerank_usage: Optional[dict] = None
    embedding_usage: dict
    costs: dict
    total_tokens: int


class QueryResponse(BaseModel):
    """查詢回應"""

    answer: str
    sources: list[SourceInfo]
    query: str
    filtered_count: int = 0
    usage: Optional[UsageSummaryInfo] = None
```

### 16.4 API 實作範例

```python
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from fastapi.responses import StreamingResponse

app = FastAPI(title="RAG Care API", version="1.0.0")

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_methods=["*"],
    allow_headers=["*"],
)


@app.post("/api/query", response_model=QueryResponse)
async def query(request: QueryRequest):
    """RAG 查詢端點"""

    try:
        response = rag_chain.query(
            question=request.question,
            top_k=request.top_k,
            hybrid=request.hybrid,
            vector_weight=request.vector_weight,
            keyword_weight=request.keyword_weight,
            hierarchical=request.hierarchical,
            rerank=request.rerank,
            category_boost=request.category_boost,
            tags=request.tags,
            source=request.source,
            score_threshold=request.score_threshold,
        )

        return QueryResponse(
            answer=response.answer,
            sources=[
                SourceInfo(
                    content=s.content,
                    source=s.source,
                    score=s.score,
                    tags=s.tags,
                    categories=s.categories,
                    category_boost=s.category_boost,
                    parent_content=s.parent_content,
                )
                for s in response.sources
            ],
            query=response.query,
            filtered_count=response.filtered_count,
            usage=response.usage.to_dict() if response.usage else None,
        )

    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))


@app.get("/health")
async def health():
    """健康檢查"""
    return {"status": "healthy"}
```

---

## 17. CLI 工具設計（Typer）

<!-- AI-SECTION: cli_design -->

### 17.1 命令結構

```bash
# 資料匯入
rag-care ingest ./data [--force] [--hierarchical] [--tag TAG]

# 查詢
rag-care query "問題" [--hybrid] [--hierarchical] [--top-k 5]

# 資料統計
rag-care list [--sources | --tags | --categories]

# 系統狀態
rag-care status

# 啟動 API
rag-care serve [-p 8000] [--reload]

# BM25 索引管理
rag-care bm25 build
rag-care bm25 stats
```

### 17.2 CLI 實作範例

```python
import typer
from rich.console import Console
from rich.table import Table

app = typer.Typer(help="RAG Care CLI")
console = Console()


@app.command()
def ingest(
    path: str = typer.Argument(..., help="資料路徑"),
    force: bool = typer.Option(False, "--force", help="強制重新匯入"),
    hierarchical: bool = typer.Option(False, "--hierarchical", help="階層式切分"),
    parent_size: int = typer.Option(2000, "--parent-size", help="Parent chunk 大小"),
    child_size: int = typer.Option(500, "--child-size", help="Child chunk 大小"),
    tag: list[str] = typer.Option([], "--tag", "-t", help="標籤"),
):
    """匯入資料到向量庫"""

    # 載入文件
    loader = DocumentLoader()
    documents = loader.load_directory(path)

    console.print(f"[cyan]載入 {len(documents)} 個文件[/cyan]")

    # 切分
    if hierarchical:
        chunker = HierarchicalChunker(
            parent_chunk_size=parent_size,
            child_chunk_size=child_size,
        )
    else:
        chunker = DocumentChunker()

    chunks = chunker.chunk_documents(documents)

    # 添加標籤
    for chunk in chunks:
        chunk.tags.extend(tag)

    # 匯入
    if force:
        stats = store.add_chunks(chunks)
        console.print(f"[green]新增 {stats} 個 chunks[/green]")
    else:
        stats = store.add_chunks_smart(chunks)
        console.print(f"[green]新增: {stats['added']}, 更新: {stats['updated']}, 跳過: {stats['skipped']}[/green]")


@app.command()
def query(
    question: str = typer.Argument(..., help="查詢問題"),
    top_k: int = typer.Option(5, "--top-k", "-k", help="檢索數量"),
    hybrid: bool = typer.Option(False, "--hybrid", help="混合搜尋"),
    hierarchical: bool = typer.Option(False, "--hierarchical", help="階層式檢索"),
    rerank: bool = typer.Option(False, "--rerank", help="LLM 重排"),
):
    """查詢諮詢記錄"""

    response = rag_chain.query(
        question=question,
        top_k=top_k,
        hybrid=hybrid,
        hierarchical=hierarchical,
        rerank=rerank,
    )

    console.print("\n[bold]回答：[/bold]")
    console.print(response.answer)

    console.print("\n[bold]參考來源：[/bold]")
    for i, source in enumerate(response.sources):
        console.print(f"  {i+1}. {source.source} (相似度: {source.score:.2%})")

    if response.usage:
        console.print(f"\n[dim]Token 使用: {response.usage.total_tokens}, 成本: ${response.usage.costs.total_cost:.6f}[/dim]")


if __name__ == "__main__":
    app()
```

---

# Part 7: 專案模板

## 18. 專案結構模板

<!-- AI-SECTION: project_template -->

### 18.1 完整目錄結構

```
my-rag-project/
├── src/
│   ├── __init__.py
│   ├── config.py                 # Pydantic Settings 配置管理
│   │
│   ├── api/                      # REST API (FastAPI)
│   │   ├── __init__.py
│   │   ├── main.py               # FastAPI 應用入口
│   │   ├── routes/
│   │   │   ├── __init__.py
│   │   │   └── query.py          # 查詢端點
│   │   └── schemas.py            # Pydantic 請求/回應模型
│   │
│   ├── cli/                      # CLI 工具 (Typer)
│   │   ├── __init__.py
│   │   ├── app.py                # Typer 應用入口
│   │   ├── ingest.py             # 匯入命令
│   │   ├── query.py              # 查詢命令
│   │   └── list.py               # 列表命令
│   │
│   ├── rag/                      # RAG 核心邏輯
│   │   ├── __init__.py
│   │   ├── chain.py              # RAG 管道
│   │   ├── retriever.py          # 文件檢索器
│   │   ├── reranker.py           # LLM 重排器
│   │   ├── category_matcher.py   # 分類匹配
│   │   ├── prompts.py            # 提示詞模板
│   │   ├── usage.py              # Token 使用量追蹤
│   │   └── pricing.py            # 成本計算
│   │
│   ├── embedding/                # Embedding 提供者
│   │   ├── __init__.py
│   │   ├── base.py               # 抽象基類
│   │   ├── gemini_embed.py       # Gemini 實作
│   │   └── openai_embed.py       # OpenAI 實作
│   │
│   ├── llm/                      # LLM 提供者
│   │   ├── __init__.py
│   │   ├── base.py               # 抽象基類
│   │   ├── gemini_llm.py         # Gemini 實作
│   │   └── openai_llm.py         # OpenAI 實作
│   │
│   ├── vectorstore/              # 向量儲存
│   │   ├── __init__.py
│   │   └── qdrant_store.py       # Qdrant 整合
│   │
│   ├── document/                 # 文件處理
│   │   ├── __init__.py
│   │   ├── loader.py             # 多格式載入器
│   │   ├── chunker.py            # 文件切分器
│   │   ├── hierarchical_chunker.py  # 階層式切分
│   │   ├── categorizer.py        # 自動分類
│   │   └── detector.py           # 格式偵測
│   │
│   ├── bm25/                     # BM25 關鍵字搜尋
│   │   ├── __init__.py
│   │   ├── bm25_store.py         # BM25 索引
│   │   └── tokenizer.py          # 分詞器
│   │
│   └── cache/                    # 快取層
│       ├── __init__.py
│       └── embedding_cache.py    # Embedding 快取
│
├── data/                         # 資料目錄
│   └── .gitkeep
│
├── docs/                         # 文件
│   ├── api.md                    # API 文件
│   └── PROJECT_NOTE.md           # 技術決策記錄
│
├── tests/                        # 測試
│   ├── __init__.py
│   ├── test_chunker.py
│   ├── test_bm25_store.py
│   └── test_rag_chain.py
│
├── pyproject.toml                # 專案配置
├── .env.example                  # 環境變數範本
├── docker-compose.yml            # Docker 配置
└── README.md                     # 專案說明
```

### 18.2 pyproject.toml 範例

```toml
[project]
name = "my-rag-project"
version = "1.0.0"
description = "RAG 系統"
requires-python = ">=3.11"
dependencies = [
    # Core
    "pydantic>=2.0.0",
    "pydantic-settings>=2.0.0",

    # Embedding & LLM
    "google-generativeai>=0.3.0",
    "openai>=1.0.0",

    # Vector DB
    "qdrant-client>=1.7.0",

    # API
    "fastapi>=0.100.0",
    "uvicorn>=0.20.0",

    # CLI
    "typer>=0.9.0",
    "rich>=13.0.0",

    # Chinese NLP
    "jieba>=0.42.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.0.0",
    "pytest-asyncio>=0.24.0",
    "ruff>=0.8.0",
    "mypy>=1.13.0",
]

[project.scripts]
rag-care = "src.cli.app:app"
```

### 18.3 docker-compose.yml 範例

```yaml
version: '3.8'

services:
  qdrant:
    image: qdrant/qdrant:v1.7.4
    container_name: rag-qdrant
    ports:
      - "6333:6333"
      - "6334:6334"
    volumes:
      - qdrant_storage:/qdrant/storage
    environment:
      - QDRANT__LOG_LEVEL=INFO
    restart: unless-stopped

  rag-api:
    build: .
    container_name: rag-api
    ports:
      - "8000:8000"
    environment:
      - QDRANT_HOST=qdrant
      - QDRANT_PORT=6333
    env_file:
      - .env
    depends_on:
      - qdrant
    restart: unless-stopped

volumes:
  qdrant_storage:
```

---

## 附錄：快速檢查清單

### AI 實作檢查清單

在實作 RAG 系統時，請確認以下項目：

- [ ] **配置管理**
  - [ ] 使用 Pydantic Settings
  - [ ] 支援 .env 檔案
  - [ ] LRU 快取設定實例

- [ ] **Embedding**
  - [ ] 實作抽象基類
  - [ ] 支援快取（SQLite）
  - [ ] 速率限制處理（指數退避）
  - [ ] Token 使用量追蹤

- [ ] **文件處理**
  - [ ] 支援多格式載入
  - [ ] 智慧切分（尋找斷點）
  - [ ] Content Hash 去重
  - [ ] 確定性 UUID 生成

- [ ] **向量儲存**
  - [ ] Qdrant Collection 初始化
  - [ ] Full-text Index 建立
  - [ ] 批量 upsert
  - [ ] 智慧更新（add_chunks_smart）

- [ ] **檢索策略**
  - [ ] 向量搜尋
  - [ ] BM25 關鍵字搜尋
  - [ ] RRF 混合融合
  - [ ] 階層式檢索（Parent-Child）
  - [ ] LLM Reranking

- [ ] **成本管理**
  - [ ] Token 使用量追蹤
  - [ ] 定價表維護
  - [ ] 成本分解計算

- [ ] **介面**
  - [ ] REST API（FastAPI）
  - [ ] CLI 工具（Typer）
  - [ ] 健康檢查端點

---

*本文件基於 rag-core 專案（約 7,785 行 Python 代碼）的深度分析撰寫，涵蓋完整的 RAG 系統架構、設計決策、效能優化與實作細節。*
