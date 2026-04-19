# 非功能需求基準測試計畫 (NFR Baseline Plan)

> **ODA Cyber Konsult**
> 文件類型：非功能需求（NFR）基準測試規劃
> 建立日期：2026-04-18
> 版本：1.0.0

---

## 背景

系統稽核（`docs/01-specs/system-cross-validation-2026-04-18.md`）指出 3 項 NFR
僅有規格宣告，從未實測：

| AC | 宣告值 | 現狀 |
|----|-------|------|
| AC-02-01-02 | RAG 查詢 P95 < 10 秒 | 未壓測，無基準 |
| AC-04-01-02 | PII 偵測召回率 > 95% | 未建標註集，無量測 |
| AC-04-04-02 | 批次處理 > 10 檔案/分鐘 | 未壓測，無基準 |

「Spec 宣稱 + 無測試」是最高風險的技術債——宣告成立但不可驗證。
本計畫確立 Q2 目標，將 3 項 NFR 從「宣告」升級為「可量測 + 可驗證」。

---

## 目標

- **Q2 結束前**完成 3 項 NFR 初次基準建立
- 每項 NFR 產出可重跑的自動化腳本 + baseline 數據 + 告警閾值
- 每季一次回歸測試，任何 > 10% 的退化須以 commit 解釋

---

## 1. RAG 查詢 P95 < 10 秒（AC-02-01-02）

### 量測對象
- 端點：`POST /api/chat`（streaming，量 time-to-first-byte + total）
- 負載情境：
  - 輕量（1 concurrent user，50 req）
  - 中等（10 concurrent users，200 req）
  - 壓力（50 concurrent users，500 req）

### 工具
- [k6](https://k6.io/)（已是 Node/TS 技術棧）
- 或 [Locust](https://locust.io/)（若偏 Python）

### 建議腳本位置
- `tests/perf/rag-chat-p95.js`（k6）
- `scripts/run-perf-chat.sh`

### Baseline 建立步驟
1. 部署乾淨環境（清空快取）
2. Seed 知識庫（152 chunks，本專案 `data/seed-knowledge/`）
3. 準備 20 條代表性查詢（beginner/standard/expert 各 6-7 條）
4. 執行 3 情境，記錄 P50/P95/P99、error rate、TTFB
5. 結果寫入 `tests/perf/results/YYYY-MM-DD-rag-chat.json`

### 通過條件
- 輕量：P95 < 5s
- 中等：P95 < 10s
- 壓力：P95 < 15s（可接受降級；error rate < 1%）

### 告警閾值
- P95 中等情境 > 12s → 必須查因
- error rate > 2% → 立即調查

---

## 2. PII 偵測召回率 > 95%（AC-04-01-02）

### 量測對象
- `python/data-pipeline/src/data_pipeline/cleaners/detector.py::detect()`
- 20 種 entity types（標準 10 + 台灣 4 + 企業密鑰 6）

### 標註資料集建立
建議 ≥ 200 句 / 種 entity（共 ≥ 4000 句）。分層抽樣：

| 類別 | 句數 | 來源 |
|------|-----|------|
| 台灣身分證 | 200 | 合成（格式 + checksum 正負樣本混合） |
| 台灣手機 | 200 | 合成（行動 + 市話、國內外格式） |
| 台灣統編 | 200 | 合成 |
| 台灣地址 | 300 | 合成（縣市 + 街名 + 門牌多樣性） |
| 人名 | 500 | 合成中/英文 + 真實小說摘錄（去識別） |
| 信箱 | 200 | 合成 |
| 電話（非台灣） | 200 | 合成國際格式 |
| 信用卡 | 200 | 合成（Luhn 正確） |
| IP / URL | 400 | 合成 |
| API Key / Token / Cloud Keys | 600 | 合成（AWS/Azure/GCP 範例） |
| 組織 / 地點 | 500 | 合成 + NER 訓練集 |
| DATE_TIME | 200 | 合成（ROC 年 + 西元 + 相對日期） |

> 合成策略：用 LLM 批次生成 + 人工抽查 5%。不可用真實 PII。

### 工具
- 自建 Python 腳本 + pytest
- 指標：Precision（false positive 率）、Recall（漏偵測率）、F1

### 建議腳本位置
- `python/rag-service/tests/perf/test_pii_recall.py`
- `data/pii-eval/`（標註資料集，git 分別管理或加密）

### 通過條件
- **每類** Recall ≥ 95%（個別類別未達 → 揭露弱點 recognizer）
- **整體** F1 ≥ 0.90
- **誤偵測率**：Precision ≥ 90%（避免過度遮蔽破壞可讀性）

### 告警閾值
- 任一類別 Recall < 90% → 必須修 recognizer
- 整體 F1 < 0.85 → 引入第三方校準（如 Presidio 新版本 / 替代模型）

---

## 3. 批次處理 > 10 檔案/分鐘（AC-04-04-02）

### 量測對象
- 端點：`POST /api/v1/upload`（多檔批次）+ `/api/v1/clean`
- 情境：
  - 小檔（100 × 50KB TXT）
  - 中檔（20 × 500KB DOCX）
  - 大檔（10 × 5MB PDF）

### 工具
- k6（upload + polling task status）
- 或 Python asyncio 批次呼叫腳本

### 建議腳本位置
- `tests/perf/cleaning-throughput.py`
- `scripts/run-perf-cleaning.sh`

### 注意事項
- `config.py` 目前 `max_workers=2`，這是吞吐量上限
- 測試時應記錄 CPU/memory，判斷是否為 worker pool 受限
- PDF 解析較重（pdfminer.six），預期吞吐量較低

### 通過條件
- 小檔：≥ 30 files/min
- 中檔：≥ 15 files/min
- 大檔：≥ 5 files/min（AC 的「10 files/min」應修正為檔案類型分層）

> 建議同步更新 PRD AC-04-04-02，改為「小檔 ≥ 30 / 中檔 ≥ 15 / 大檔 ≥ 5 files/min」

---

## 執行排程

| 週 | 里程碑 | 負責 |
|----|-------|------|
| Week 1 | 建立 k6 腳本（AC-02-01-02 初版）；跑首次 baseline | DEV |
| Week 2 | 建立 PII 標註集（前 10 類，共 ~2000 句）；跑首次 recall 評估 | DEV+QA |
| Week 3 | 完成 PII 標註集（後 10 類）；跑整體 F1 評估 | QA |
| Week 4 | 完成 cleaning throughput 腳本；跑 3 情境基準 | DEV |
| Week 5 | 將 3 腳本納入 CI（nightly + weekly）；建立 dashboard | DEV |
| Week 6 | 首次月度 NFR 回歸；寫 retrospective | PM |

---

## 成功定義

1. 3 個自動化腳本存在、可一鍵跑、結果可機讀
2. 3 組 baseline 數據留存於 `tests/perf/results/`
3. RTM v1.5.0 標註 3 項 NFR 為「已量測 + baseline 日期」
4. CI 每週執行一次，退化 > 10% 自動發 issue
5. 發現的弱點同步進入 `CLAUDE_LESSONS.md` 與對應 spec 調整

---

## 相關文件

- `docs/01-specs/system-cross-validation-2026-04-18.md` — NFR 未驗證的原始 finding
- `docs/02-testing/test-strategy.md` — 測試總策略
- `docs/01-specs/RTM.md` — 需求追溯矩陣
