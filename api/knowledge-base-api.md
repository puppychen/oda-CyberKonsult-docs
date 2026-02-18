# 知識庫瀏覽 API 文件

> 本文件涵蓋知識庫（Qdrant 向量資料庫）的瀏覽、搜尋與管理 API。
>
> **使用範圍**：供系統管理員與資料清洗人員使用，用於瀏覽與管理已匯入 RAG 知識庫的文件。

## 目錄

- [通用說明](#通用說明)
- [列出知識庫文件](#列出知識庫文件)
- [列出所有來源](#列出所有來源)
- [搜尋知識庫](#搜尋知識庫)
- [按來源刪除文件](#按來源刪除文件)

---

## 通用說明

### Base URL

```
http://localhost:4000/api/v1/knowledge-base
```

### 認證

所有端點需要 JWT Bearer Token，且使用者角色須為 `admin` 或 `data_cleaner`。

---

## 列出知識庫文件

分頁瀏覽 Qdrant 中的向量化文件。

- **方法**: `GET`
- **路徑**: `/api/v1/knowledge-base/documents`
- **權限**: `admin`, `data_cleaner`

### 查詢參數

| 參數 | 型別 | 預設值 | 說明 |
|------|------|--------|------|
| `page` | `integer` | 1 | 頁碼（最小 1） |
| `limit` | `integer` | 20 | 每頁筆數（1-100） |
| `source` | `string` | - | 按來源篩選 |
| `tag` | `string` | - | 按標籤篩選 |

### 回應結構

| 欄位 | 型別 | 說明 |
|------|------|------|
| `items` | `KnowledgeBaseDocument[]` | 文件清單 |
| `total` | `integer` | 符合條件的總筆數 |
| `page` | `integer` | 目前頁碼 |
| `limit` | `integer` | 每頁筆數 |

### KnowledgeBaseDocument 結構

| 欄位 | 型別 | 說明 |
|------|------|------|
| `id` | `string` | Qdrant 點 ID |
| `content` | `string` | 文件內容（截斷至 200 字元） |
| `source` | `string` | 來源路徑 |
| `tags` | `string[]` | 標籤列表 |
| `chunk_index` | `integer` | 分塊序號 |
| `content_hash` | `string` | 內容雜湊值 |

### 回應範例

```json
{
  "items": [
    {
      "id": "abc123-def456",
      "content": "資通安全管理法第一條...",
      "source": "docs/security-law.pdf",
      "tags": ["法規", "資安"],
      "chunk_index": 0,
      "content_hash": "a1b2c3d4"
    }
  ],
  "total": 150,
  "page": 1,
  "limit": 20
}
```

---

## 列出所有來源

取得知識庫中所有不重複的文件來源。

- **方法**: `GET`
- **路徑**: `/api/v1/knowledge-base/sources`
- **權限**: `admin`, `data_cleaner`

### 回應結構

| 欄位 | 型別 | 說明 |
|------|------|------|
| `sources` | `string[]` | 來源列表（已排序） |

### 回應範例

```json
{
  "sources": [
    "docs/security-law.pdf",
    "gdrive://abc123/report.docx",
    "uploads/training-data.txt"
  ]
}
```

---

## 搜尋知識庫

使用向量相似度搜尋知識庫中的文件。

- **方法**: `GET`
- **路徑**: `/api/v1/knowledge-base/search`
- **權限**: `admin`, `data_cleaner`

### 查詢參數

| 參數 | 型別 | 預設值 | 說明 |
|------|------|--------|------|
| `q` | `string` | **必填** | 搜尋關鍵字（最少 1 字元） |
| `top_k` | `integer` | 10 | 回傳結果數量（1-50） |
| `source` | `string` | - | 按來源篩選 |
| `tag` | `string` | - | 按標籤篩選 |

### 回應結構

| 欄位 | 型別 | 說明 |
|------|------|------|
| `items` | `KnowledgeBaseSearchItem[]` | 搜尋結果 |
| `total` | `integer` | 結果筆數 |
| `query` | `string` | 原始搜尋字串 |

### KnowledgeBaseSearchItem 結構

| 欄位 | 型別 | 說明 |
|------|------|------|
| `id` | `string` | Qdrant 點 ID |
| `content` | `string` | 文件內容（截斷至 200 字元） |
| `source` | `string` | 來源路徑 |
| `score` | `float` | 相似度分數 |
| `tags` | `string[]` | 標籤列表 |
| `chunk_index` | `integer` | 分塊序號 |

---

## 按來源刪除文件

刪除知識庫中指定來源的所有文件，同時清除 BM25 索引。

- **方法**: `DELETE`
- **路徑**: `/api/v1/knowledge-base/documents/by-source`
- **權限**: `admin`

### 查詢參數

| 參數 | 型別 | 說明 |
|------|------|------|
| `source` | `string` | **必填** — 要刪除的來源 |

### 回應結構

| 欄位 | 型別 | 說明 |
|------|------|------|
| `source` | `string` | 被刪除的來源 |
| `deleted_count` | `integer` | 刪除的文件數量 |

### 回應範例

```json
{
  "source": "docs/old-policy.pdf",
  "deleted_count": 15
}
```

---

> **相關文件**
> - 清洗 API: `docs/api/cleaning-api.md`
> - RAG API: `docs/api/rag-api.md`
> - Google Drive API: `docs/api/gdrive-api.md`
