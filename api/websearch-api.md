# WebSearch API 文件

> ODA Cyber Konsult WebSearch 系統 API - SearXNG 整合與 RAG 檢索

Base URL (NestJS): `http://localhost:4000/api`
Base URL (FastAPI): `http://localhost:8000/api/v1`

---

## 目錄

- [概覽](#概覽)
- [架構說明](#架構說明)
- [WebSearch Config API](#websearch-config-api)
- [RAG Retrieve API](#rag-retrieve-api)
- [LLM 設定](#llm-設定)
- [Docker 部署](#docker-部署)
- [錯誤回應](#錯誤回應)
- [使用範例](#使用範例)

---

## 概覽

WebSearch 系統整合 SearXNG 元搜尋引擎，作為 RAG 知識庫的補充資料來源。當知識庫檢索結果不足時，系統會自動觸發網路搜尋以提供更全面的資訊。

### 核心功能

- **自動觸發**：根據可設定的閾值自動啟動網路搜尋
- **雙閾值邏輯**：結果數量 < `minResultsThreshold` **或** 最佳分數 < `minScoreThreshold`
- **優雅降級**：SearXNG 服務離線時，系統正常運作（僅返回知識庫結果）
- **內容擷取**：自動抓取網頁內容並提供摘要
- **管理員專用**：Config 設定僅限 Admin 角色

---

## 架構說明

### 觸發條件

WebSearch 會在以下條件**滿足任一項**時自動啟動：

| 條件 | 閾值欄位 | 說明 |
|------|----------|------|
| 結果數量不足 | `minResultsThreshold` | RAG 檢索結果數量 < 設定值 |
| 最佳分數過低 | `minScoreThreshold` | RAG 檢索最佳分數 < 設定值 |

**邏輯運算**: `OR` (只要其中一項條件成立即觸發)

### 整合流程

```
使用者查詢
    ↓
RAG 知識庫檢索
    ↓
檢查觸發條件 (minResultsThreshold OR minScoreThreshold)
    ↓
[條件成立] → SearXNG 網路搜尋 → 網頁內容擷取 → 合併結果
    ↓
[條件不成立] → 直接返回知識庫結果
    ↓
LLM 生成回答
```

### 系統組件

| 組件 | 說明 |
|------|------|
| **SearXNG** | 元搜尋引擎（整合 Google、Bing、DuckDuckGo 等） |
| **WebFetchService** | 網頁內容擷取與清理服務 |
| **RAG Service** | 知識庫檢索與回答生成 |
| **NestJS Proxy** | WebSearch Config 管理 API（Admin 專用） |

---

## WebSearch Config API

### GET `/api/websearch/config`

取得 WebSearch 系統設定（**Admin only**）。

**認證**: JWT Bearer Token (Admin 角色)

**Response (200 OK):**

```json
{
  "success": true,
  "data": {
    "enabled": true,
    "minResultsThreshold": 3,
    "minScoreThreshold": 0.7,
    "maxResults": 5,
    "searxngUrl": "http://localhost:8888",
    "webFetchMaxLength": 3000,
    "webFetchTimeout": 30,
    "categories": ["general"],
    "language": "zh-TW"
  },
  "timestamp": "2026-02-13T10:00:00.000Z"
}
```

**Response 欄位說明:**

| 欄位 | 型別 | 說明 |
|------|------|------|
| `enabled` | boolean | WebSearch 功能開關 |
| `minResultsThreshold` | integer | 最少結果數量閾值（0-50） |
| `minScoreThreshold` | number | 最低分數閾值（0-1） |
| `maxResults` | integer | 網路搜尋最大結果數量（1-20） |
| `searxngUrl` | string | SearXNG 服務位址 |
| `webFetchMaxLength` | integer | 網頁擷取最大長度（500-10000 字元） |
| `webFetchTimeout` | integer | 網頁擷取超時時間（5-60 秒） |
| `categories` | string[] | SearXNG 搜尋分類 |
| `language` | string | 搜尋語言（zh-TW / en-US） |

---

### PUT `/api/websearch/config`

更新 WebSearch 系統設定（**Admin only**）。

**認證**: JWT Bearer Token (Admin 角色)

**Request Body:**

```json
{
  "enabled": true,
  "minResultsThreshold": 3,
  "minScoreThreshold": 0.7,
  "maxResults": 5,
  "searxngUrl": "http://localhost:8888",
  "webFetchMaxLength": 3000,
  "webFetchTimeout": 30,
  "categories": ["general", "news"],
  "language": "zh-TW"
}
```

**Request 驗證規則:**

| 欄位 | 驗證規則 |
|------|----------|
| `enabled` | 選填，必須為 boolean |
| `minResultsThreshold` | 選填，整數 0-50 |
| `minScoreThreshold` | 選填，浮點數 0-1 |
| `maxResults` | 選填，整數 1-20 |
| `searxngUrl` | 選填，必須為有效 URL |
| `webFetchMaxLength` | 選填，整數 500-10000 |
| `webFetchTimeout` | 選填，整數 5-60 |
| `categories` | 選填，字串陣列 |
| `language` | 選填，字串（zh-TW / en-US / en / zh） |

**Response (200 OK):**

```json
{
  "success": true,
  "data": {
    "enabled": true,
    "minResultsThreshold": 3,
    "minScoreThreshold": 0.7,
    "maxResults": 5,
    "searxngUrl": "http://localhost:8888",
    "webFetchMaxLength": 3000,
    "webFetchTimeout": 30,
    "categories": ["general", "news"],
    "language": "zh-TW"
  },
  "timestamp": "2026-02-13T10:05:00.000Z"
}
```

---

## RAG Retrieve API

### POST `/api/v1/rag/retrieve`

執行 RAG 檢索（**僅檢索，不生成回答**），用於測試 WebSearch 觸發邏輯。

**Base URL**: FastAPI (`http://localhost:8000`)

**Request Body:**

| 欄位 | 型別 | 必填 | 預設 | 說明 |
|------|------|------|------|------|
| `question` | string | Y | - | 使用者問題 |
| `top_k` | integer | N | 5 | 檢索結果數量（1-20） |
| `hybrid` | boolean | N | false | 啟用混合搜尋 |
| `hierarchical` | boolean | N | false | 啟用階層式檢索 |
| `rerank` | boolean | N | false | 啟用 LLM 重排 |
| `source_filter` | string | N | null | 按來源文件過濾 |
| `tag_filter` | string | N | null | 按標籤過濾 |

**Request Example:**

```json
{
  "question": "什麼是零信任架構？",
  "top_k": 5,
  "hybrid": true,
  "rerank": false
}
```

**Response (200 OK):**

```json
{
  "results": [
    {
      "source": "/data/documents/zero-trust-guide.pdf",
      "content_preview": "零信任架構（Zero Trust Architecture, ZTA）是一種資安框架...",
      "score": 0.85
    },
    {
      "source": "/data/documents/network-security.pdf",
      "content_preview": "傳統邊界防護已不足以應對現代威脅...",
      "score": 0.72
    }
  ],
  "metadata": {
    "count": 2,
    "best_score": 0.85
  }
}
```

**Response 欄位說明:**

| 欄位 | 型別 | 說明 |
|------|------|------|
| `results` | array | 檢索結果列表 |
| `results[].source` | string | 來源文件路徑 |
| `results[].content_preview` | string | 內容預覽 |
| `results[].score` | number | 相似度分數（0-1） |
| `metadata` | object | 檢索元資料 |
| `metadata.count` | integer | 結果數量 |
| `metadata.best_score` | number | 最佳分數 |

**觸發 WebSearch 判斷:**

```python
# 假設 Config: minResultsThreshold=3, minScoreThreshold=0.7

# 範例 1: count=2 < 3 → 觸發 WebSearch
# 範例 2: count=5, best_score=0.65 < 0.7 → 觸發 WebSearch
# 範例 3: count=5, best_score=0.85 → 不觸發 WebSearch
```

---

## LLM 設定

WebSearch 整合後，LLM 生成回答時會同時使用知識庫與網路搜尋結果。

### 環境變數

| 變數 | 預設 | 說明 |
|------|------|------|
| `LLM_PROVIDER` | `gemini` | LLM 提供者（`gemini` / `openai`） |
| `GEMINI_API_KEY` | - | Google AI API Key（使用 Gemini 時必填） |
| `OPENAI_API_KEY` | - | OpenAI API Key（使用 OpenAI 時必填） |
| `LLM_MODEL_GEMINI` | `gemini-2.0-flash-exp` | Gemini 模型名稱 |
| `LLM_MODEL_OPENAI` | `gpt-4o-mini` | OpenAI 模型名稱 |
| `LLM_TEMPERATURE` | `0.1` | 生成溫度（0-2） |
| `LLM_MAX_TOKENS` | `2048` | 最大生成 Token 數 |

### LLM Prompt 結構

```
System Prompt: 你是一位專業的資安顧問...

Context:
[知識庫結果 1]
[知識庫結果 2]
[網路搜尋結果 1]
[網路搜尋結果 2]

Question: {user_question}
```

---

## Docker 部署

### 正式環境

在 `docker-compose.prod.yml` 中，SearXNG 僅供內部服務使用，不對外開放：

```yaml
services:
  searxng:
    image: searxng/searxng:latest
    container_name: oda-searxng
    networks:
      - oda-network
    # 不映射外部 Port
    environment:
      - SEARXNG_BASE_URL=http://searxng:8888
    volumes:
      - ./config/searxng:/etc/searxng

  rag-service:
    environment:
      - WEBSEARCH_ENABLED=true
      - SEARXNG_URL=http://searxng:8888  # Docker 內部互連
    depends_on:
      - searxng
```

**內部連線**: RAG Service 透過 Docker network (`oda-network`) 存取 SearXNG，外部無法直接訪問。

---

### 開發環境

在 `.env` 中設定本機 SearXNG 位址：

```bash
# WebSearch 設定
WEBSEARCH_ENABLED=true
SEARXNG_URL=http://localhost:8888
WEBSEARCH_MIN_RESULTS=3
WEBSEARCH_MIN_SCORE=0.7
WEBSEARCH_MAX_RESULTS=5
WEBSEARCH_FETCH_TIMEOUT=30
WEBSEARCH_FETCH_MAX_LENGTH=3000
```

**啟動 SearXNG**:

```bash
docker run -d \
  --name searxng \
  -p 8888:8080 \
  -v $(pwd)/config/searxng:/etc/searxng \
  searxng/searxng:latest
```

**測試連線**:

```bash
curl http://localhost:8888/search?q=cybersecurity&format=json
```

---

## 優雅降級

### SearXNG 服務離線

當 SearXNG 服務無法連線時，系統會：

1. **自動降級**: 僅使用 RAG 知識庫結果，不中斷服務
2. **記錄警告**: 在日誌中記錄 WebSearch 失敗（不拋出錯誤）
3. **返回結果**: 正常返回知識庫檢索結果

**範例日誌**:

```
[WARN] WebSearch request failed: Connection refused (http://localhost:8888)
[INFO] Falling back to RAG-only results
```

### Config 設定 `enabled: false`

當管理員將 `enabled` 設為 `false` 時，系統會完全跳過 WebSearch 流程，直接使用知識庫結果。

---

## 錯誤回應

### 401 Unauthorized

```json
{
  "statusCode": 401,
  "message": "Unauthorized"
}
```

### 403 Forbidden

```json
{
  "statusCode": 403,
  "message": "Insufficient permissions. Admin role required.",
  "error": "Forbidden"
}
```

### 400 Bad Request

```json
{
  "statusCode": 400,
  "message": [
    "minResultsThreshold must be between 0 and 50",
    "minScoreThreshold must be between 0 and 1"
  ],
  "error": "Bad Request"
}
```

### 500 Internal Server Error

```json
{
  "statusCode": 500,
  "message": "Failed to update WebSearch config",
  "error": "Internal Server Error"
}
```

---

## 使用範例

### cURL - 取得 Config (Admin)

```bash
curl -X GET http://localhost:4000/api/websearch/config \
  -H "Authorization: Bearer YOUR_ADMIN_JWT_TOKEN"
```

### cURL - 更新 Config (Admin)

```bash
curl -X PUT http://localhost:4000/api/websearch/config \
  -H "Authorization: Bearer YOUR_ADMIN_JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "enabled": true,
    "minResultsThreshold": 3,
    "minScoreThreshold": 0.7,
    "maxResults": 5,
    "webFetchTimeout": 30
  }'
```

### cURL - RAG Retrieve 測試

```bash
curl -X POST http://localhost:8000/api/v1/rag/retrieve \
  -H "Content-Type: application/json" \
  -d '{
    "question": "什麼是零信任架構？",
    "top_k": 5,
    "hybrid": true
  }'
```

### JavaScript (Fetch) - 取得 Config

```javascript
const response = await fetch('http://localhost:4000/api/websearch/config', {
  headers: {
    'Authorization': `Bearer ${adminToken}`,
  },
});

const result = await response.json();
console.log(result.data);
```

### JavaScript (Fetch) - 更新 Config

```javascript
const response = await fetch('http://localhost:4000/api/websearch/config', {
  method: 'PUT',
  headers: {
    'Authorization': `Bearer ${adminToken}`,
    'Content-Type': 'application/json',
  },
  body: JSON.stringify({
    enabled: true,
    minResultsThreshold: 3,
    minScoreThreshold: 0.7,
    maxResults: 5,
  }),
});

const result = await response.json();
console.log('Updated config:', result.data);
```

### Python - RAG Retrieve 測試

```python
import requests

response = requests.post(
    "http://localhost:8000/api/v1/rag/retrieve",
    json={
        "question": "什麼是零信任架構？",
        "top_k": 5,
        "hybrid": True
    }
)

data = response.json()
print(f"Results count: {data['metadata']['count']}")
print(f"Best score: {data['metadata']['best_score']}")

# 判斷是否會觸發 WebSearch
min_results = 3
min_score = 0.7

if data['metadata']['count'] < min_results or data['metadata']['best_score'] < min_score:
    print("✓ Would trigger WebSearch")
else:
    print("✗ Would NOT trigger WebSearch")
```

---

## 注意事項

1. **權限控制**: Config 管理端點（`/api/websearch/config`）僅限 Admin 角色存取。
2. **觸發邏輯**: 雙閾值採用 **OR 邏輯**（任一條件成立即觸發），非 AND。
3. **優雅降級**: SearXNG 離線不影響系統正常運作，會自動回退至知識庫結果。
4. **網頁擷取超時**: 預設 30 秒，可透過 `webFetchTimeout` 調整（5-60 秒）。
5. **內容長度限制**: 預設擷取前 3000 字元，可透過 `webFetchMaxLength` 調整（500-10000）。
6. **語言設定**: 支援 `zh-TW`（繁體中文）、`en-US`（英文）等 SearXNG 標準語言代碼。
7. **分類設定**: SearXNG 支援多種分類（general、news、science、tech 等），可依需求調整。
8. **正式環境安全**: SearXNG 不對外開放，僅供 Docker 內部服務使用。
9. **開發環境**: 本機測試時需手動啟動 SearXNG Docker 容器並映射 Port 8888。
10. **RAG Retrieve**: 此端點不生成回答，僅返回檢索結果與元資料，適合測試觸發邏輯。

---

## 相關文件

- [Chat API Documentation](./chat-api.md)
- [RAG API Documentation](./rag-api.md)
- [Authentication API](./auth-api.md)
- [SearXNG 官方文件](https://docs.searxng.org/)
