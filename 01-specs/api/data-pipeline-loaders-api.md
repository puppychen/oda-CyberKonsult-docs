# Data Pipeline - Document Loaders API 文件

> 最後更新: 2026-02-11 (markitdown 整合)

## 概述

Document Loaders 模組提供統一介面來解析多種格式的文件，包括 PDF、DOCX、Excel、PPTX、HTML、Markdown 等。所有解析器都實作 `DocumentParser` 抽象類別，並透過 `ParserFactory` 自動選擇適當的解析器。

自 v0.2.0 起，本模組採用**混合整合架構 (Hybrid Approach)**：核心格式由專屬解析器處理（保留結構化資料），額外格式與未知格式由 Microsoft [markitdown](https://github.com/microsoft/markitdown)（MIT）作為 fallback 處理。

## 架構設計

```
ParserFactory
├── PDFParser (.pdf)             ← pdfminer.six (MIT)
├── DocxParser (.docx, .doc)     ← python-docx
├── ExcelParser (.xlsx)          ← openpyxl (read_only)
├── TextParser (.txt, .md, .json, .html, .csv) ← beautifulsoup4 (HTML)
├── MarkitdownParser (.pptx, .xls, .epub, .msg) ← markitdown
└── [fallback] MarkitdownParser  ← 未知副檔名自動嘗試
```

### 設計決策

| 決策 | 理由 |
|------|------|
| PDF 用 pdfminer.six 而非 PyMuPDF | PyMuPDF 為 AGPL 授權，商用需額外付費；pdfminer.six 為 MIT |
| 保留 python-docx / openpyxl | 需要結構化表格 (`ParsedTable`) 做表格級 PII 偵測 |
| HTML 用 beautifulsoup4 而非正則 | bs4 對巢狀標籤、特殊字元、`<script>`/`<style>` 移除更可靠 |
| markitdown 僅作 fallback | markitdown 輸出為純 Markdown 字串，無原生結構化表格/元資料 |
| 安裝 `markitdown[pptx,xls]` | 避免 `[all]` 引入 magika + onnxruntime（~40MB 額外依賴） |

---

## 支援格式一覽

| 副檔名 | 解析器 | 底層函式庫 | 結構化表格 | 元資料 |
|--------|--------|-----------|:----------:|:------:|
| `.pdf` | PDFParser | pdfminer.six | 簡易偵測 | 有 |
| `.docx` | DocxParser | python-docx | 有 | 有 |
| `.doc` | DocxParser | python-docx | 有 | 有 |
| `.xlsx` | ExcelParser | openpyxl | 有 | 無 |
| `.xls` | MarkitdownParser | markitdown (pandas+xlrd) | Markdown 表格 | 無 |
| `.pptx` | MarkitdownParser | markitdown (python-pptx) | Markdown 表格 | 無 |
| `.epub` | MarkitdownParser | markitdown | 無 | 無 |
| `.msg` | MarkitdownParser | markitdown (olefile) | 無 | 無 |
| `.txt` | TextParser | 標準庫 | 無 | 無 |
| `.md` | TextParser | 標準庫 | 無 | 無 |
| `.json` | TextParser | 標準庫 | 無 | 無 |
| `.html` | TextParser | beautifulsoup4 | 無 | 無 |
| `.csv` | TextParser | 標準庫 | 無 | 無 |
| 其他 | MarkitdownParser (fallback) | markitdown | Markdown 表格 | 無 |

---

## 核心類別

### `ParserFactory`

工廠類別，負責根據檔案類型建立對應的解析器。當 markitdown 已安裝時，未知副檔名會自動 fallback 到 MarkitdownParser。

#### 方法

##### `create(file_path: Path) -> DocumentParser`

根據檔案副檔名建立適當的解析器。

```python
from pathlib import Path
from data_pipeline.loaders import ParserFactory

factory = ParserFactory()

# 已知格式 → 專屬解析器
parser = factory.create(Path("report.pdf"))      # → PDFParser
parser = factory.create(Path("slides.pptx"))     # → MarkitdownParser

# 未知格式 → markitdown fallback（若已安裝）
parser = factory.create(Path("data.odt"))        # → MarkitdownParser
```

**參數:**
- `file_path` (Path): 文件檔案路徑

**回傳:**
- `DocumentParser`: 可處理該檔案類型的解析器實例

**異常:**
- `ValueError`: 檔案類型不支援且 markitdown 未安裝時拋出

---

##### `parse(file_path: Path) -> ParsedDocument`

便捷方法，直接解析檔案並回傳結果。

```python
from pathlib import Path
from data_pipeline.loaders import ParserFactory

factory = ParserFactory()
doc = factory.parse(Path("document.pdf"))
print(f"Content: {doc.content[:100]}...")
print(f"Pages: {doc.metadata.page_count}")
```

**參數:**
- `file_path` (Path): 文件檔案路徑

**回傳:**
- `ParsedDocument`: 解析後的文件

**異常:**
- `FileNotFoundError`: 檔案不存在
- `ValueError`: 檔案類型不支援或格式無效

---

##### `supported_extensions` (屬性)

取得所有支援的檔案副檔名列表。

```python
factory = ParserFactory()
print(factory.supported_extensions)
# ['.csv', '.doc', '.docx', '.epub', '.html', '.json', '.md', '.msg',
#  '.pdf', '.pptx', '.txt', '.xls', '.xlsx']
```

---

### `DocumentParser` (抽象基類)

所有解析器的基類，定義統一介面。

#### 抽象方法

##### `parse(file_path: Path) -> ParsedDocument`

解析指定路徑的文件。

---

##### `supported_extensions` (屬性)

回傳此解析器支援的副檔名列表。

---

#### 實作方法

##### `can_parse(file_path: Path) -> bool`

檢查此解析器是否能處理指定檔案。

```python
from pathlib import Path
from data_pipeline.loaders import PDFParser

parser = PDFParser()
print(parser.can_parse(Path("doc.pdf")))  # True
print(parser.can_parse(Path("doc.docx")))  # False
```

---

## 解析器實作

### `PDFParser`

使用 pdfminer.six（MIT 授權）解析 PDF 文件。透過 layout analysis 逐頁擷取文字，並以定界符模式（tab / pipe / 多空格）進行簡易表格偵測。

**支援格式:** `.pdf`

**底層函式庫:** `pdfminer.six`（純 Python，MIT 授權）

**功能:**
- 逐頁擷取文字內容（layout analysis + fallback extract_text）
- 簡易表格偵測（tab / pipe / 多空格分隔的連續列）
- 讀取元資料（Title、Author、CreationDate）
- 完整 CJK（中日韓）字元支援（內建 CMap）

**範例:**

```python
from pathlib import Path
from data_pipeline.loaders import PDFParser

parser = PDFParser()
doc = parser.parse(Path("report.pdf"))

print(f"頁數: {doc.metadata.page_count}")
print(f"標題: {doc.metadata.title}")
print(f"表格數量: {len(doc.tables)}")
print(f"第一頁內容: {doc.pages[0][:200]}")
```

**表格偵測邏輯:**

pdfminer.six 無內建表格偵測（不同於 PyMuPDF），因此 PDFParser 使用啟發式方法：
1. 擷取每行文字及其 Y 座標
2. 偵測是否含 tab (`\t`)、pipe (`|`) 或連續多空格 (` {2,}`) 作為分隔符
3. 連續且欄數一致的分隔行組成表格
4. 至少 2 列（1 表頭 + 1 資料列）才視為有效表格

> **注意:** 此簡易偵測適用於結構明確的表格（如 Markdown 風格、CSV 風格），但對於僅靠空間對齊的複雜 PDF 表格（如掃描式 PDF）可能無法偵測。

---

### `DocxParser`

使用 python-docx 解析 Word 文件。

**支援格式:** `.docx`, `.doc`

**功能:**
- 擷取所有段落文字
- 解析文件中的表格（完整結構化 `ParsedTable`）
- 讀取核心屬性（作者、標題、建立時間）

**範例:**

```python
from pathlib import Path
from data_pipeline.loaders import DocxParser

parser = DocxParser()
doc = parser.parse(Path("proposal.docx"))

print(f"作者: {doc.metadata.author}")
print(f"內容長度: {len(doc.content)}")
print(f"表格: {len(doc.tables)} 個")

for table in doc.tables:
    print(f"表頭: {table.headers}")
    print(f"資料列數: {len(table.rows)}")
```

---

### `ExcelParser`

使用 openpyxl 解析 Excel 試算表（`.xlsx` 格式）。

**支援格式:** `.xlsx`

> **注意:** 舊版 `.xls` 格式由 MarkitdownParser 處理。

**功能:**
- 將每個工作表轉換為 `ParsedTable`
- 自動識別表頭（第一列）
- 合併所有工作表內容為文字

**特性:**
- 使用 `read_only=True` 模式節省記憶體
- 第一列自動視為表頭
- 空白列會自動略過
- 工作表索引對應到 `ParsedTable.page`

**範例:**

```python
from pathlib import Path
from data_pipeline.loaders import ExcelParser

parser = ExcelParser()
doc = parser.parse(Path("data.xlsx"))

print(f"工作表數量: {doc.metadata.page_count}")
print(f"表格數量: {len(doc.tables)}")

for idx, table in enumerate(doc.tables):
    print(f"\n表格 {idx + 1}:")
    print(f"  欄位: {table.headers}")
    print(f"  資料列: {len(table.rows)}")
```

---

### `TextParser`

解析純文字及標記語言文件。HTML 檔案使用 BeautifulSoup 進行標籤清理。

**支援格式:** `.txt`, `.md`, `.json`, `.html`, `.csv`

**功能:**
- 自動偵測編碼（UTF-8、Big5、GB2312 等）
- HTML 檔案使用 BeautifulSoup 移除 `<script>`、`<style>`、`<noscript>` 及 HTML 註解
- 保留原始文字格式

**編碼偵測順序:**
1. UTF-8
2. UTF-8 with BOM
3. Big5
4. GB2312
5. CP950
6. Latin-1

**HTML 清理行為:**

```python
# 輸入
html = """
<html>
<head><style>body { color: red; }</style></head>
<body>
  <script>alert('xss')</script>
  <p>重要內容</p>
  <!-- 這是註解 -->
  <noscript>請啟用 JavaScript</noscript>
</body>
</html>
"""

# 輸出（自動清除所有標籤、script、style、noscript、註解）
# → "重要內容"
```

**範例:**

```python
from pathlib import Path
from data_pipeline.loaders import TextParser

parser = TextParser()

# 解析 Markdown
doc = parser.parse(Path("README.md"))
print(doc.content)

# 解析 HTML（BeautifulSoup 自動清除標籤）
html_doc = parser.parse(Path("page.html"))
print(html_doc.content)  # 純文字，無 HTML 標籤
```

---

### `MarkitdownParser`

使用 Microsoft markitdown 解析額外格式（PPTX、XLS、EPUB、MSG），同時作為未知格式的 fallback。輸出的 Markdown 表格會自動轉換為 `ParsedTable`。

**支援格式:** `.pptx`, `.xls`, `.epub`, `.msg`（+ 任何 markitdown 能處理的格式作為 fallback）

**底層函式庫:** `markitdown[pptx,xls]`（MIT 授權）

**功能:**
- 透過 `markitdown.convert()` 將檔案轉為 Markdown
- 自動解析 Markdown 表格（`| ... |` 格式）為 `ParsedTable`
- 語言偵測
- 作為 ParserFactory 的 fallback 層

**限制:**
- 輸出為 Markdown 字串，無原生結構化元資料
- 表格偵測依賴 Markdown 格式（非所有格式都會產出 Markdown 表格）
- `page_count` 始終為 0（markitdown 不提供頁面概念）

**Markdown 表格轉換邏輯:**

```markdown
# markitdown 輸出的 Markdown 表格格式
| 姓名 | 年齡 | 城市 |
|------|------|------|
| 王小明 | 30 | 台北 |
| 李小華 | 25 | 高雄 |
```

會自動轉換為：

```python
ParsedTable(
    headers=["姓名", "年齡", "城市"],
    rows=[["王小明", "30", "台北"], ["李小華", "25", "高雄"]],
    page=0,
)
```

**範例:**

```python
from pathlib import Path
from data_pipeline.loaders import MarkitdownParser

parser = MarkitdownParser()

# 解析 PowerPoint
doc = parser.parse(Path("slides.pptx"))
print(f"內容長度: {len(doc.content)}")
print(f"表格數量: {len(doc.tables)}")

# 解析舊版 Excel
doc = parser.parse(Path("legacy.xls"))
print(doc.content[:200])
```

**直接透過工廠使用:**

```python
from pathlib import Path
from data_pipeline.loaders import ParserFactory

factory = ParserFactory()

# PPTX 自動路由到 MarkitdownParser
doc = factory.parse(Path("presentation.pptx"))

# 未知格式也會嘗試 markitdown
doc = factory.parse(Path("archive.epub"))
```

---

## 工具類別

### `LanguageDetector`

語言偵測工具，帶有 LRU 快取機制。

#### 靜態方法

##### `detect(text: str) -> str`

偵測文字的語言。

```python
from data_pipeline.loaders import LanguageDetector

lang = LanguageDetector.detect("這是一段中文文字")
print(lang)  # 'zh'

lang = LanguageDetector.detect("This is English text")
print(lang)  # 'en'
```

**參數:**
- `text` (str): 要分析的文字內容

**回傳:**
- `str`: ISO 639-1 語言代碼（例如: `zh`, `en`, `ja`）

**預設行為:**
- 文字過短（< 10 字元）: 回傳 `"zh"`
- 偵測失敗: 回傳 `"zh"`

**快取:**
- 使用 `@lru_cache(maxsize=1024)` 快取相同文字的偵測結果
- 提升重複內容的處理效能

---

## 資料模型

所有解析器回傳 `ParsedDocument` 物件，包含：

### `ParsedDocument`

```python
class ParsedDocument(BaseModel):
    content: str                    # 完整文字內容
    metadata: DocumentMetadata      # 文件元資料
    tables: list[ParsedTable]       # 擷取的表格列表
    pages: list[str]                # 逐頁內容（PDF）或分段內容
```

### `DocumentMetadata`

```python
class DocumentMetadata(BaseModel):
    filename: str       # 檔案名稱
    file_type: str      # 檔案類型（pdf, docx, excel, pptx 等）
    file_size: int      # 檔案大小（位元組）
    page_count: int     # 頁數或工作表數（markitdown 固定為 0）
    language: str       # 偵測到的語言代碼
    title: str          # 標題（如有）
    author: str         # 作者（如有）
    created_at: str     # 建立時間
    extra: dict         # 額外的自訂欄位
```

### `ParsedTable`

```python
class ParsedTable(BaseModel):
    headers: list[str]          # 表頭列
    rows: list[list[str]]       # 資料列
    page: int                   # 所在頁數或工作表索引
```

---

## 使用範例

### 基本用法

```python
from pathlib import Path
from data_pipeline.loaders import ParserFactory

factory = ParserFactory()

# 解析文件（自動選擇解析器）
doc = factory.parse(Path("/path/to/document.pdf"))

print(f"語言: {doc.metadata.language}")
print(f"檔案大小: {doc.metadata.file_size} bytes")
print(f"內容摘要: {doc.content[:200]}...")
```

### 批次處理多個文件

```python
from pathlib import Path
from data_pipeline.loaders import ParserFactory

factory = ParserFactory()
doc_dir = Path("/path/to/documents")

for file_path in doc_dir.iterdir():
    if file_path.suffix.lower() in factory.supported_extensions:
        try:
            doc = factory.parse(file_path)
            print(f"OK {file_path.name}: {len(doc.content)} chars, "
                  f"{len(doc.tables)} tables")
        except Exception as e:
            print(f"FAIL {file_path.name}: {e}")
```

### 處理表格資料

```python
from pathlib import Path
from data_pipeline.loaders import ParserFactory

factory = ParserFactory()
doc = factory.parse(Path("data.xlsx"))

for idx, table in enumerate(doc.tables):
    print(f"\n表格 {idx + 1} (頁面 {table.page}):")
    print("表頭:", " | ".join(table.headers))
    print("-" * 50)
    for row in table.rows[:5]:  # 前 5 列
        print(" | ".join(row))
```

### 手動選擇解析器

```python
from pathlib import Path
from data_pipeline.loaders import PDFParser, MarkitdownParser

# 直接使用特定解析器
pdf_parser = PDFParser()
doc = pdf_parser.parse(Path("report.pdf"))

# 使用 markitdown 解析 PPTX
md_parser = MarkitdownParser()
doc = md_parser.parse(Path("slides.pptx"))
```

### 利用 Fallback 處理未知格式

```python
from pathlib import Path
from data_pipeline.loaders import ParserFactory

factory = ParserFactory()

# 未知格式會自動嘗試 markitdown
# 成功 → 回傳 ParsedDocument
# 失敗 → 拋出 ValueError
try:
    doc = factory.parse(Path("unknown.odt"))
    print(f"成功解析: {len(doc.content)} chars")
except ValueError as e:
    print(f"無法解析: {e}")
```

---

## RAG DocumentLoader 整合

`rag_service.document.loader.DocumentLoader` 在 RAG 匯入時使用 ParserFactory：

```
DocumentLoader.load(file_path)
  ├── .txt/.md/.json/.jsonl/.csv → 原生載入器（不經 ParserFactory）
  ├── .pdf/.docx/.xlsx/.pptx/.xls/.html/.epub/.msg → ParserFactory
  └── 其他 → 嘗試純文字讀取
```

**支援的 Upload API 白名單:**

```python
ALLOWED_TYPES = {
    ".pdf", ".docx", ".doc", ".xlsx", ".xls", ".pptx",
    ".txt", ".md", ".json", ".csv", ".html", ".htm",
}
```

**支援的 Google Drive MIME 類型:**

| MIME Type | 副檔名 |
|-----------|--------|
| `application/pdf` | `.pdf` |
| `application/vnd.openxmlformats-officedocument.wordprocessingml.document` | `.docx` |
| `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet` | `.xlsx` |
| `application/vnd.openxmlformats-officedocument.presentationml.presentation` | `.pptx` |
| `application/vnd.ms-excel` | `.xls` |
| `text/csv` | `.csv` |
| `text/plain` | `.txt` |
| `text/markdown` | `.md` |
| `text/html` | `.html` |
| `application/json` | `.json` |

---

## 錯誤處理

### 常見異常

| 異常類型 | 發生時機 | 處理建議 |
|---------|---------|---------|
| `FileNotFoundError` | 檔案不存在 | 檢查路徑是否正確 |
| `ValueError` (Unsupported file type) | 檔案類型不支援且無 fallback | 安裝 markitdown 啟用 fallback |
| `ValueError` (Invalid file) | 檔案格式損壞或無效 | 驗證檔案完整性 |
| `ValueError` (markitdown failed) | markitdown 無法轉換 | 該格式可能不在 markitdown 支援範圍 |

### 錯誤處理範例

```python
from pathlib import Path
from data_pipeline.loaders import ParserFactory

factory = ParserFactory()

try:
    doc = factory.parse(Path("document.pptx"))
except FileNotFoundError:
    print("檔案不存在")
except ValueError as e:
    if "markitdown failed" in str(e):
        print("markitdown 無法轉換此檔案")
    elif "Unsupported" in str(e):
        print(f"不支援的類型，支援: {factory.supported_extensions}")
    else:
        print(f"檔案格式無效: {e}")
```

---

## 效能考量

### 各解析器效能特性

| 解析器 | 速度 | 記憶體 | 備註 |
|--------|------|--------|------|
| PDFParser (pdfminer.six) | 中等 | 中等 | 純 Python，比 PyMuPDF 慢 2-5x，但 MIT 授權 |
| DocxParser | 快 | 中等 | python-docx 載入整份文件到記憶體 |
| ExcelParser | 快 | 低 | openpyxl `read_only=True` 串流讀取 |
| TextParser | 極快 | 低 | 標準庫 + bs4（僅 HTML） |
| MarkitdownParser | 中等 | 中等 | 依賴底層轉換器（pandas for xls, python-pptx for pptx） |

### 語言偵測快取

`LanguageDetector.detect()` 使用 LRU 快取 (maxsize=1024)，重複文字的偵測幾乎無延遲。

### 建議

1. 大型 Excel 檔案建議用 `.xlsx`（ExcelParser + openpyxl read_only），避免 `.xls`（markitdown + pandas 較重）
2. PDF 表格偵測為啟發式方法，複雜表格建議先轉 XLSX 再解析
3. 文字編碼偵測會嘗試多種編碼，UTF-8 檔案效能最佳
4. markitdown 轉換為 lazy-import，不使用時不佔記憶體

---

## 依賴套件

| 套件 | 版本 | 授權 | 用途 |
|------|------|------|------|
| `pdfminer.six` | >= 20231228 | MIT | PDF 解析（取代 PyMuPDF） |
| `python-docx` | >= 1.1.0 | MIT | DOCX 解析 |
| `openpyxl` | >= 3.1.0 | MIT | XLSX 解析 |
| `beautifulsoup4` | >= 4.12.0 | MIT | HTML 標籤清理 |
| `markitdown[pptx,xls]` | >= 0.1.4 | MIT | PPTX/XLS/EPUB/MSG 解析 + fallback |
| `langdetect` | >= 1.0.9 | Apache 2.0 | 語言偵測 |

> **已移除:** `pymupdf` (AGPL) — 於 2026-02-11 替換為 `pdfminer.six` (MIT)

---

## 程式碼位置

- 基礎路徑: `python/data-pipeline/src/data_pipeline/loaders/`
- 模組檔案:

| 檔案 | 說明 |
|------|------|
| `base.py` | 抽象基類 `DocumentParser` |
| `factory.py` | 工廠類別 `ParserFactory`（含 markitdown fallback 邏輯） |
| `pdf.py` | PDF 解析器（pdfminer.six） |
| `docx.py` | DOCX 解析器（python-docx） |
| `excel.py` | Excel 解析器（openpyxl） |
| `text.py` | 文字/HTML 解析器（beautifulsoup4） |
| `markitdown_parser.py` | markitdown 包裝解析器 |
| `language.py` | 語言偵測器 |

---

## 變更紀錄

### v0.2.0 (2026-02-11) — markitdown 混合整合

**新增:**
- `MarkitdownParser` — 支援 PPTX、XLS、EPUB、MSG 格式
- ParserFactory fallback 機制 — 未知副檔名自動嘗試 markitdown
- Markdown 表格自動轉換為 `ParsedTable`
- Upload API 新增 `.pptx`、`.xls`、`.doc`、`.csv` 白名單
- Google Drive MIME 新增 PPTX (`application/vnd.openxmlformats-officedocument.presentationml.presentation`) 及 XLS (`application/vnd.ms-excel`)

**變更:**
- PDFParser: PyMuPDF (AGPL) → pdfminer.six (MIT)
- TextParser._strip_html: 正則 → beautifulsoup4
- ExcelParser: `.xls` 格式改由 MarkitdownParser 處理

**移除:**
- `pymupdf` 依賴（AGPL 授權問題）

### v0.1.0 (2026-02-05) — 初版

- PDFParser (PyMuPDF)、DocxParser、ExcelParser、TextParser
- ParserFactory 基本路由
- LanguageDetector 語言偵測
