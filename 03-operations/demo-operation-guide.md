# CyberKonsult 系統展示操作指南

> 最後更新：2026-03-24
> 適用版本：ODA Cyber Konsult POC
> 目的：引導展示者操作三層回應模式的差異化功能

---

## 一、系統啟動步驟

### 1.1 基礎設施（Docker）

```bash
# 確認 Docker 容器運行中
docker ps | grep -E "postgres|qdrant|searxng"

# 若未啟動
cd docker && docker compose up -d
```

| 服務 | Port | 驗證方式 |
|------|------|---------|
| PostgreSQL | 5432 | `pg_isready -h localhost -p 5432` |
| Qdrant | 6333 | `curl http://localhost:6333/health` |
| SearXNG | 8080 | Docker 內部，不對外 |

### 1.2 後端服務

```bash
# 在專案根目錄
pnpm dev
```

啟動後各服務：

| 服務 | Port | 驗證方式 |
|------|------|---------|
| NestJS API | 3051 | `curl http://localhost:3051/health` |
| RAG Service | 3502 | `curl http://localhost:3502/health` |
| Admin Dashboard | 5501 | 瀏覽器開啟 |
| Chatbot UI | 5502 | 瀏覽器開啟 |
| Cleaner App | 5503 | 瀏覽器開啟 |

### 1.3 知識庫初始化（首次展示前）

```bash
# 1. 更新提示詞範本
cd apps/api && pnpm prisma:seed

# 2. 匯入種子知識庫
cd ../../data/seed-knowledge && bash ingest.sh
```

---

## 二、預設帳號列表

| 帳號 | 密碼 | 角色 | 展示用途 |
|------|------|------|---------|
| user@oda-cyber.com | OdaPoc2026! | user | 新手模式展示 |
| ituser@oda-cyber.com | OdaPoc2026! | it_user | 一般模式（IT 工程師）展示 |
| consultant@oda-cyber.com | OdaPoc2026! | consultant | 顧問模式展示 |
| admin@oda-cyber.com | OdaPoc2026! | admin | 管理後台展示 |
| cleaner@oda-cyber.com | OdaPoc2026! | data_cleaner | 清洗審核展示（Maker） |
| reviewer@oda-cyber.com | OdaPoc2026! | data_reviewer | 清洗審核展示（Checker） |

> 首次登入後系統會要求變更密碼（資通安全「普」級密碼政策），展示前請先完成密碼變更。

---

## 三、三模式展示腳本

### 展示 1：新手模式（約 3 分鐘）

**操作步驟**：

1. 開啟 Chatbot UI：http://localhost:5502
2. 登入帳號：`user@oda-cyber.com`
3. 確認左上角模式顯示「新手模式」（user 角色預設）
4. 輸入問題：

   > 什麼是釣魚信件？要怎麼判斷？

5. 等待 SSE 串流回應完成

**預期畫面**：

- 回應以白話文呈現，使用日常比喻
- 包含台灣企業常見情境（假冒供應商付款、假發票、偽冒 IT 通知）
- 引用真實新聞案例或統計數據
- 提供步驟化辨識方法（條列式）
- 信心度指示器：🟢 知識庫來源
- 引用來源：顯示知識庫文件名稱

**展示重點話術**：

> 「新手模式針對非技術背景的使用者設計，系統自動用白話文和企業日常情境來說明，不使用專業術語。特別的是，回答中引用了台灣真實的釣魚案例和統計數據，而非通用的教科書內容。」

---

### 展示 2：一般模式 — IT 工程師（約 3 分鐘）

**操作步驟**：

1. 登出，重新登入為：`ituser@oda-cyber.com`
2. 點擊模式選擇器，切換至「標準模式」
3. 輸入問題：

   > 我被主管機關發文要出具個資安全維護計畫，我要怎麼寫？

4. 等待回應完成

**預期畫面**：

- 回應包含可交付的文件結構大綱
- 引用個資法施行細則第 12 條（12 項措施）
- 包含 ISO 27001/27701 架構對照
- 提供查核清單或自評表
- 回應長度明顯多於新手模式（maxTokens: 2048）
- 引用來源：5 份知識庫文件

**展示重點話術**：

> 「切換到一般模式後，同樣的系統檢索了更多文件（5 份 vs 3 份），並且回答直接引用了個資法施行細則第 12 條的具體要求。更重要的是，系統產出了一份可以直接提交給主管機關的文件大綱結構。這不是通用的建議，而是符合台灣法規的實務範本。」

---

### 展示 3：顧問模式（約 4 分鐘）

**操作步驟**：

1. 登出，重新登入為：`consultant@oda-cyber.com`
2. 點擊模式選擇器，切換至「顧問模式」
3. 輸入問題：

   > 資安法新法修正哪些條款？

4. 等待回應完成

**預期畫面**：

- 包含新舊條文逐條對照分析
- 區分公務機關與特定非公務機關的差異
- 對應 ISO 27001 具體條款
- 提供顧問輔導客戶的注意事項
- 回應篇幅最長（maxTokens: 4096）
- 引用來源數量最多（topK: 8）
- Reranking 已啟用（更精準的文件排序）

**展示重點話術**：

> 「顧問模式啟用了 Reranking 精準排序，檢索了 8 份知識庫文件，提供最深度的分析。回答不只列出修法重點，還做了新舊條文對照、區分了公務機關和非公務機關的差異、並且對應到 ISO 27001 的具體條款。這正是資安顧問在輔導客戶時需要的完整分析。」

---

### 模式差異總結（展示時可用）

| 比較項目 | 新手模式 | 一般模式 | 顧問模式 |
|---------|---------|---------|---------|
| 檢索文件數 | 3 | 5 | 8 |
| Reranking | 關 | 關 | 開 |
| 回應風格 | 白話+比喻 | 法規引用+範本 | 條文對照+策略分析 |
| 回應長度 | 短（~1024 tokens） | 中（~2048 tokens） | 長（~4096 tokens） |
| 信心度 | 🟢 知識庫 | 🟢 知識庫 | 🟢 知識庫 |
| 適合對象 | 非技術背景 | IT/MIS 工程師 | 資安顧問 |

---

## 四、常見問題排除

### 問題 1：檢索無結果（回應提示「資料不足」）

```bash
# 確認知識庫已匯入
curl -s http://localhost:3502/api/v1/rag/stats | jq .

# 若 total_documents 為 0，重新匯入
cd data/seed-knowledge && bash ingest.sh
```

### 問題 2：服務未啟動

```bash
# 檢查所有服務狀態
curl -s http://localhost:3051/health | jq .
curl -s http://localhost:3502/health | jq .
```

### 問題 3：模式無法切換

- 確認登入角色是否有權限切換至目標模式
- user/basic_user 無法切換至 expert 模式
- 重新整理頁面後再試

### 問題 4：SSE 串流中斷

- 檢查 NestJS API 日誌：`pnpm --filter @oda-cyber/api dev`
- 確認 LLM API Key 已設定（Gemini 或 OpenAI）

### 問題 5：密碼過期強制變更

- 首次登入或 90 天後系統會強制要求變更密碼
- 密碼要求：8 碼以上、含大小寫+數字+特殊字元

---

## 五、知識庫管理

### 新增文件

```bash
# 方式 1：API 直接匯入
curl -X POST http://localhost:3502/api/v1/ingest \
  -F "files=@新文件.md" \
  -F "tags=新標籤" \
  -F "hierarchical=true"

# 方式 2：放入 seed-knowledge 目錄後重新匯入
cp 新文件.md data/seed-knowledge/法規標準/
cd data/seed-knowledge && bash ingest.sh
```

### 刪除特定來源

```bash
curl -X DELETE "http://localhost:3502/api/v1/knowledge-base/documents/by-source" \
  -H "Content-Type: application/json" \
  -d '{"source": "data/seed-knowledge/資安意識/釣魚信件防護指南.md"}'
```

### 查看知識庫內容

- Cleaner App（http://localhost:5503）→ 知識庫管理
- 三個 Tab：文件管理 / 語意搜尋 / 來源分析

---

## 六、附錄：系統技術規格快覽

| 項目 | 規格 |
|------|------|
| RAG 檢索 | Hybrid Search（Vector + BM25 + RRF） |
| 向量模型 | Google Embedding / OpenAI |
| LLM | Gemini / OpenAI |
| 分塊策略 | 階層式（Parent 3500 字元 / Child 800 字元） |
| PII 偵測 | Presidio + spaCy（20 種實體） |
| 認證 | JWT + 資通安全「普」級密碼政策 |
| 授權 | RBAC 7 角色 |
