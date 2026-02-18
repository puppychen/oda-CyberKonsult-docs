---
# YAML Frontmatter - AI 可解析的元資料
document_type: system_reproduction
version: "1.0.0"
project_name: "RAG 資料清洗平台"
ai_parseable: true
last_updated: "2026-01-25"
spec_version: "1.0"
languages: ["zh-TW", "en"]
tech_stack:
  backend: ["Python 3.11+", "FastAPI", "SQLAlchemy 2.0", "Celery", "Presidio"]
  frontend: ["React 18", "TypeScript", "Tailwind CSS", "TanStack Query"]
  database: ["PostgreSQL 16", "Redis 7"]
  storage: ["MinIO"]
  nlp: ["spaCy 3.7+", "Microsoft Presidio"]
---

# RAG 資料清洗平台 - AI 系統複刻指南

> **文件目的**：本文件提供完整的系統規格，讓 AI（如 Claude Code）能 100% 複刻此系統。

---

## @SECTION: METADATA

### 專案識別

| 項目 | 值 |
|------|-----|
| 專案名稱 | RAG 資料清洗平台 |
| 版本 | 0.1.0 |
| 主要語言 | Python 3.11+ / TypeScript 5.3+ |
| 授權 | MIT |

### 技術棧摘要

```yaml
backend:
  framework: FastAPI 0.109+
  orm: SQLAlchemy 2.0 (async)
  task_queue: Celery 5.3+
  pii_detection: Microsoft Presidio 2.2+
  nlp_engine: spaCy 3.7+ (zh_core_web_trf, en_core_web_trf)

frontend:
  framework: React 18.2+
  language: TypeScript 5.3+
  styling: Tailwind CSS 3.4+
  state: TanStack Query 5.17+
  router: React Router 6.21+

infrastructure:
  database: PostgreSQL 16
  cache: Redis 7
  storage: MinIO (S3-compatible)
  container: Docker Compose 3.8
```

---

## @SECTION: DIRECTORY_STRUCTURE

### 完整目錄樹

```
@STRUCTURE: root
rag-data-clean/
├── docker-compose.yml          # Docker 容器編排
├── .env.example                # 環境變數範本
├── PROJECT_CONTEXT.md          # 專案概況文件
├── docs/                       # 文件目錄
│   ├── RAG_CLEANER_COOKBOOK.md # 本文件
│   ├── anonymization-guide.md  # 脫敏指南
│   ├── quick-start.md          # 快速開始
│   └── examples/               # 規則範例
│       ├── academic-research.json
│       ├── full-compliance.json
│       └── medical-research.json
├── backend/                    # FastAPI 後端
│   ├── Dockerfile
│   ├── pyproject.toml          # Python 依賴
│   ├── alembic.ini             # 資料庫遷移配置
│   ├── alembic/                # 遷移腳本
│   └── app/                    # 應用程式碼
│       ├── __init__.py
│       ├── main.py             # FastAPI 進入點
│       ├── api/                # API 路由
│       │   ├── __init__.py
│       │   └── v1/
│       │       ├── __init__.py
│       │       ├── upload.py   # 檔案上傳 API
│       │       ├── clean.py    # 清洗任務 API
│       │       ├── tasks.py    # 任務狀態 API
│       │       ├── rules.py    # 規則管理 API
│       │       ├── download.py # 下載 API
│       │       └── websocket.py # WebSocket API
│       ├── core/               # 核心設定
│       │   ├── __init__.py
│       │   ├── config.py       # 應用程式設定
│       │   ├── database.py     # 資料庫連線
│       │   ├── celery_app.py   # Celery 設定
│       │   ├── security.py     # 安全工具
│       │   └── init_db.py      # 資料庫初始化
│       ├── models/             # ORM 模型
│       │   ├── __init__.py
│       │   ├── base.py         # 基礎模型
│       │   ├── user.py         # 使用者模型
│       │   ├── file.py         # 檔案模型
│       │   ├── rule.py         # 規則模型
│       │   ├── task.py         # 任務模型
│       │   └── audit.py        # 稽核日誌模型
│       ├── schemas/            # Pydantic Schema
│       │   ├── __init__.py
│       │   ├── file.py         # 檔案 Schema
│       │   ├── clean.py        # 清洗 Schema
│       │   ├── rule.py         # 規則 Schema
│       │   └── task.py         # 任務 Schema
│       ├── repositories/       # 資料存取層
│       │   ├── __init__.py
│       │   ├── base.py         # 基礎 Repository
│       │   ├── user.py
│       │   ├── file.py
│       │   ├── rule.py
│       │   ├── task.py
│       │   └── audit.py
│       ├── services/           # 業務邏輯層
│       │   ├── __init__.py
│       │   ├── detector/       # PII 偵測模組
│       │   │   ├── __init__.py
│       │   │   ├── engine.py   # 偵測引擎
│       │   │   └── custom_recognizers/
│       │   │       ├── __init__.py
│       │   │       ├── tw_id_recognizer.py      # 台灣身分證
│       │   │       ├── tw_phone_recognizer.py   # 台灣電話
│       │   │       ├── tw_business_recognizer.py # 統一編號
│       │   │       └── api_key_recognizer.py    # API 金鑰
│       │   ├── anonymizer/     # 脫敏模組
│       │   │   ├── __init__.py
│       │   │   ├── engine.py   # 脫敏引擎
│       │   │   ├── relation_keeper.py # 關聯保留器
│       │   │   └── strategies/
│       │   │       ├── __init__.py
│       │   │       ├── base.py         # 策略基類
│       │   │       ├── mask.py         # 完全遮蔽
│       │   │       ├── partial_mask.py # 部分遮蔽
│       │   │       ├── pseudonymize.py # 假名化
│       │   │       ├── generalize.py   # 泛化
│       │   │       ├── keep_labeled.py # 保留標籤
│       │   │       └── encrypt.py      # 加密替換
│       │   ├── parser/         # 文件解析模組
│       │   │   ├── __init__.py
│       │   │   ├── base.py     # 解析器基類
│       │   │   ├── factory.py  # 解析器工廠
│       │   │   ├── language.py # 語言偵測
│       │   │   ├── pdf.py      # PDF 解析
│       │   │   ├── docx.py     # Word 解析
│       │   │   ├── excel.py    # Excel 解析
│       │   │   └── text.py     # 純文字解析
│       │   └── audit/          # 稽核模組
│       │       ├── __init__.py
│       │       └── logger.py   # 稽核日誌
│       └── tasks/              # Celery 任務
│           ├── __init__.py
│           └── processor.py    # 處理任務
├── frontend/                   # React 前端
│   ├── Dockerfile
│   ├── package.json
│   ├── tsconfig.json
│   ├── vite.config.ts
│   ├── tailwind.config.js
│   ├── postcss.config.js
│   ├── index.html
│   └── src/
│       ├── main.tsx            # React 進入點
│       ├── App.tsx             # 主應用元件
│       ├── index.css           # 全域樣式
│       ├── vite-env.d.ts
│       ├── types/
│       │   └── index.ts        # TypeScript 型別
│       ├── services/
│       │   └── api.ts          # API 客戶端
│       ├── hooks/
│       │   └── useWebSocket.ts # WebSocket Hook
│       ├── pages/
│       │   ├── HomePage.tsx
│       │   ├── RulesPage.tsx
│       │   └── TasksPage.tsx
│       └── components/
│           ├── Layout.tsx
│           ├── FileUploader/
│           │   └── index.tsx
│           ├── RuleEditor/
│           │   └── index.tsx
│           ├── TaskManager/
│           │   └── index.tsx
│           ├── ResultPreview/
│           │   └── index.tsx
│           └── DiffView/
│               └── index.tsx
└── data/                       # 資料目錄
    ├── input/                  # 輸入檔案
    │   └── .gitkeep
    └── output/                 # 輸出檔案
        └── .gitkeep
```

### 檔案統計

| 類型 | 數量 |
|------|------|
| 後端 Python 檔案 | 63 |
| 前端 TypeScript 檔案 | 15 |
| 配置檔案 | 8 |

---

## @SECTION: BACKEND_ARCHITECTURE

### 分層架構

```
@ARCHITECTURE: layers
┌─────────────────────────────────────────────────────────────┐
│                      API Layer (FastAPI)                     │
│  upload.py | clean.py | tasks.py | rules.py | download.py   │
├─────────────────────────────────────────────────────────────┤
│                     Schema Layer (Pydantic)                  │
│         file.py | clean.py | rule.py | task.py              │
├─────────────────────────────────────────────────────────────┤
│                     Service Layer                            │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │  Detector   │  │ Anonymizer  │  │   Parser    │         │
│  │  (Presidio) │  │  (Engines)  │  │  (Factory)  │         │
│  └─────────────┘  └─────────────┘  └─────────────┘         │
├─────────────────────────────────────────────────────────────┤
│                   Repository Layer (SQLAlchemy)              │
│     user.py | file.py | rule.py | task.py | audit.py        │
├─────────────────────────────────────────────────────────────┤
│                      Model Layer (ORM)                       │
│     User | File | RuleProfile | Task | TaskFile | AuditLog  │
└─────────────────────────────────────────────────────────────┘
```

### @MODULE: detector

PII 偵測引擎，基於 Microsoft Presidio 實作。

#### @CLASS: PIIDetector

```python
@INTERFACE: PIIDetector
class PIIDetector:
    """
    PII 偵測引擎

    Features:
    - 自動語言偵測 (zh-tw, en)
    - 台灣專屬識別器 (身分證、電話、統編)
    - 企業機密偵測 (API Key, Token)
    - 可配置偵測閾值
    """

    LANGUAGE_MAP = {
        "zh-tw": "zh",
        "zh-cn": "zh",
        "en": "en",
    }

    DEFAULT_ENTITIES = [
        "PERSON", "EMAIL_ADDRESS", "PHONE_NUMBER",
        "LOCATION", "DATE_TIME", "CREDIT_CARD",
        "IP_ADDRESS", "TW_ID", "TW_PHONE",
        "API_KEY", "ACCESS_TOKEN",
    ]

    def __init__(
        self,
        languages: Optional[List[str]] = None,  # default: ["zh", "en"]
        default_score_threshold: float = 0.3,
    ): ...

    def detect(
        self,
        text: str,
        language: Optional[str] = None,
        entities: Optional[List[str]] = None,
        score_threshold: Optional[float] = None,
    ) -> DetectionResult: ...

    def detect_batch(
        self,
        texts: List[str],
        language: Optional[str] = None,
        entities: Optional[List[str]] = None,
        score_threshold: Optional[float] = None,
    ) -> List[DetectionResult]: ...
```

#### @CLASS: DetectedEntity

```python
@INTERFACE: DetectedEntity
@dataclass
class DetectedEntity:
    entity_type: str      # 實體類型 (PERSON, TW_ID, etc.)
    text: str             # 原始文字
    start: int            # 起始位置
    end: int              # 結束位置
    score: float          # 信心分數 (0-1)
    recognition_metadata: Dict = field(default_factory=dict)
```

#### 支援的實體類型

```python
@ENUM: EntityType
class EntityType(str, Enum):
    # 標準 PII
    PERSON = "PERSON"
    EMAIL_ADDRESS = "EMAIL_ADDRESS"
    PHONE_NUMBER = "PHONE_NUMBER"
    LOCATION = "LOCATION"
    DATE_TIME = "DATE_TIME"
    CREDIT_CARD = "CREDIT_CARD"
    IBAN_CODE = "IBAN_CODE"
    IP_ADDRESS = "IP_ADDRESS"
    URL = "URL"

    # 台灣專屬
    TW_ID = "TW_ID"                         # 身分證字號
    TW_PHONE = "TW_PHONE"                   # 手機號碼
    TW_UNIFIED_BUSINESS_NO = "TW_UNIFIED_BUSINESS_NO"  # 統一編號

    # 企業機密
    API_KEY = "API_KEY"
    ACCESS_TOKEN = "ACCESS_TOKEN"
    PRIVATE_KEY = "PRIVATE_KEY"
    AWS_ACCESS_KEY = "AWS_ACCESS_KEY"
    AZURE_KEY = "AZURE_KEY"
    GCP_KEY = "GCP_KEY"
```

### @MODULE: anonymizer

脫敏引擎，支援 6 種策略與關聯保留。

#### @CLASS: AnonymizationEngine

```python
@INTERFACE: AnonymizationEngine
class AnonymizationEngine:
    """
    脫敏引擎

    Usage:
        engine = AnonymizationEngine()
        engine.set_rule("PERSON", "pseudonymize", prefix="Patient_")
        engine.set_rule("TW_ID", "mask")
        engine.set_rule("DATE_TIME", "generalize", precision="year")

        result = engine.anonymize("王小明的身分證是A123456789")
        # Output: Patient_001的身分證是**********
    """

    DEFAULT_RULES = {
        "PERSON": EntityRule("PERSON", "pseudonymize"),
        "TW_ID": EntityRule("TW_ID", "mask"),
        "TW_PHONE": EntityRule("TW_PHONE", "partial_mask", {"reveal_first": 4, "reveal_last": 3}),
        "EMAIL_ADDRESS": EntityRule("EMAIL_ADDRESS", "partial_mask"),
        "PHONE_NUMBER": EntityRule("PHONE_NUMBER", "partial_mask", {"reveal_first": 4}),
        "CREDIT_CARD": EntityRule("CREDIT_CARD", "partial_mask", {"reveal_last": 4}),
        "LOCATION": EntityRule("LOCATION", "generalize", {"precision": "city"}),
        "DATE_TIME": EntityRule("DATE_TIME", "generalize", {"precision": "year"}),
        "API_KEY": EntityRule("API_KEY", "mask"),
        "ACCESS_TOKEN": EntityRule("ACCESS_TOKEN", "mask"),
        "PRIVATE_KEY": EntityRule("PRIVATE_KEY", "mask"),
    }

    def __init__(
        self,
        detector: Optional[PIIDetector] = None,
        relation_keeper: Optional[RelationKeeper] = None,
        use_default_rules: bool = True,
    ): ...

    def set_rule(
        self,
        entity_type: str,
        strategy: str,
        enabled: bool = True,
        **options,
    ): ...

    def anonymize(
        self,
        text: str,
        language: Optional[str] = None,
        entities: Optional[List[str]] = None,
    ) -> AnonymizationResult: ...
```

#### @CLASS: RelationKeeper

```python
@INTERFACE: RelationKeeper
class RelationKeeper:
    """
    關聯保留器 - 確保同一原始值映射到相同的脫敏值

    Features:
    - 跨文件一致性
    - Session 管理
    - 映射匯出/匯入
    - 碰撞偵測

    Usage:
        keeper = RelationKeeper(session_id="research_001")
        keeper.add_mapping("王小明", "Patient_001", "PERSON", "pseudonymize")

        # 後續處理其他文件時
        existing = keeper.get_anonymized("王小明", "PERSON")
        # Returns: "Patient_001"
    """

    def add_mapping(
        self,
        original: str,
        anonymized: str,
        entity_type: str,
        strategy: str,
        metadata: Optional[Dict] = None,
    ) -> RelationMapping: ...

    def get_anonymized(self, original: str, entity_type: str) -> Optional[str]: ...
    def get_original(self, anonymized: str, entity_type: str) -> Optional[str]: ...
    def export_mappings(self) -> Dict[str, Any]: ...
    def import_mappings(cls, data: Dict[str, Any]) -> "RelationKeeper": ...
```

#### 脫敏策略

```python
@ENUM: StrategyType
class StrategyType(str, Enum):
    MASK = "mask"               # 完全遮蔽: "王小明" → "***"
    PARTIAL_MASK = "partial_mask"  # 部分遮蔽: "0912345678" → "0912***678"
    PSEUDONYMIZE = "pseudonymize"  # 假名化: "王小明" → "Patient_001"
    GENERALIZE = "generalize"      # 泛化: "1990-05-15" → "1990年代"
    KEEP_LABELED = "keep_labeled"  # 保留標籤: "王小明" → "[PERSON]"
    ENCRYPT = "encrypt"            # 加密替換: 可逆加密

# 策略選項
@OPTIONS: mask
{}  # 無特殊選項

@OPTIONS: partial_mask
{
    "reveal_first": int,  # 保留前幾位
    "reveal_last": int,   # 保留後幾位
    "mask_char": str,     # 遮蔽字元，預設 "*"
}

@OPTIONS: pseudonymize
{
    "prefix": str,        # 前綴，如 "Patient_"
    "maintain_relations": bool,  # 是否維持關聯
}

@OPTIONS: generalize
{
    "precision": str,     # 精度: "year", "decade", "city", "country"
}

@OPTIONS: keep_labeled
{
    "format": str,        # 格式: "bracket" → "[TYPE]", "xml" → "<TYPE>...</TYPE>"
    "include_value": bool,  # 是否包含原值
}

@OPTIONS: encrypt
{
    "key": str,           # 加密金鑰 (選填，自動產生)
}
```

### @MODULE: parser

文件解析器，支援多種格式。

#### @CLASS: ParserFactory

```python
@INTERFACE: ParserFactory
class ParserFactory:
    """
    解析器工廠 - 根據檔案類型自動選擇解析器

    Usage:
        document = ParserFactory.parse("/path/to/file.pdf")
        print(document.content)
        print(document.metadata)
    """

    @classmethod
    def get_parser(cls, file_path: str) -> DocumentParser: ...

    @classmethod
    def parse(cls, file_path: str) -> ParsedDocument: ...

    @classmethod
    def supports(cls, file_path: str) -> bool: ...

    @classmethod
    def get_supported_extensions(cls) -> List[str]: ...
```

#### 支援格式

| 格式 | 副檔名 | 解析器 |
|------|--------|--------|
| PDF | .pdf | PDFParser (pdfminer.six) |
| Word | .docx, .doc | DocxParser (python-docx) |
| Excel | .xlsx, .xls, .csv | ExcelParser (openpyxl, pandas) |
| 純文字 | .txt, .md, .json, .html | TextParser |

#### @CLASS: ParsedDocument

```python
@INTERFACE: ParsedDocument
@dataclass
class ParsedDocument:
    content: str                    # 提取的文字內容
    metadata: DocumentMetadata      # 文件元資料
    tables: List[ParsedTable]       # 提取的表格
    language: Optional[str]         # 偵測到的語言
    sections: List[str]             # 文件章節
    errors: List[str]               # 解析錯誤/警告

    def get_full_text(self, include_tables: bool = True) -> str: ...
```

---

## @SECTION: DATABASE_SCHEMA

### ER 關聯圖

```
@DIAGRAM: ER
┌─────────────┐       ┌─────────────────┐       ┌─────────────┐
│    users    │       │  rule_profiles  │       │    files    │
├─────────────┤       ├─────────────────┤       ├─────────────┤
│ id (PK)     │◄──┬───│ id (PK)         │   ┌───│ id (PK)     │
│ username    │   │   │ name            │   │   │ original_name│
│ email       │   │   │ description     │   │   │ file_type   │
│ hashed_pwd  │   │   │ is_default      │   │   │ size_bytes  │
│ is_active   │   │   │ is_active       │   │   │ storage_path│
│ created_at  │   │   │ user_id (FK)────┼───┘   │ user_id(FK)─┼──┐
│ updated_at  │   │   │ created_at      │       │ uploaded_at │  │
└──────┬──────┘   │   │ updated_at      │       └──────┬──────┘  │
       │          │   └────────┬────────┘              │         │
       │          │            │                       │         │
       │          │            ▼                       │         │
       │          │   ┌─────────────────────┐         │         │
       │          │   │ anonymization_rules │         │         │
       │          │   ├─────────────────────┤         │         │
       │          │   │ id (PK)             │         │         │
       │          │   │ profile_id (FK)─────┼─────────┘         │
       │          │   │ entity_type         │                   │
       │          │   │ strategy            │                   │
       │          │   │ options (JSONB)     │                   │
       │          │   │ enabled             │                   │
       │          │   │ sequence            │                   │
       │          │   └─────────────────────┘                   │
       │          │                                             │
       │          │   ┌─────────────┐       ┌─────────────────┐│
       │          │   │    tasks    │       │   task_files    ││
       │          │   ├─────────────┤       ├─────────────────┤│
       │          └───│ id (PK)     │◄──────│ id (PK)         ││
       │              │ status      │       │ task_id (FK)    ││
       │              │ progress    │       │ file_id (FK)────┼┘
       │              │ message     │       │ status          │
       │              │ files_total │       │ entities_found  │
       │              │ files_proc  │       │ output_path     │
       │              │ total_ent   │       │ processing_time │
       │              │ proc_time   │       │ error_message   │
       │              │ rule_id(FK) │       │ created_at      │
       │              │ session_id  │       │ completed_at    │
       └──────────────│ user_id(FK) │       └─────────────────┘
                      │ created_at  │
                      │ started_at  │
                      │ completed_at│
                      │ error_msg   │
                      └─────────────┘

┌─────────────────┐
│   audit_logs    │
├─────────────────┤
│ id (PK)         │
│ timestamp       │
│ action          │
│ resource_type   │
│ resource_id     │
│ user_id (FK)    │
│ session_id      │
│ ip_address      │
│ details (JSONB) │
│ status          │
│ error_message   │
└─────────────────┘
```

### @TABLE: users

```sql
@DDL: users
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    username VARCHAR(100) NOT NULL UNIQUE,
    email VARCHAR(255) NOT NULL UNIQUE,
    hashed_password VARCHAR(255) NOT NULL,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_users_email ON users(email);
```

### @TABLE: files

```sql
@DDL: files
CREATE TABLE files (
    id VARCHAR(36) PRIMARY KEY,  -- UUID string
    original_name VARCHAR(255) NOT NULL,
    file_type VARCHAR(50) NOT NULL,
    size_bytes BIGINT NOT NULL,
    storage_path VARCHAR(500),
    user_id UUID REFERENCES users(id) ON DELETE SET NULL,
    uploaded_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_files_user_id ON files(user_id);
```

### @TABLE: rule_profiles

```sql
@DDL: rule_profiles
CREATE TABLE rule_profiles (
    id VARCHAR(36) PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    description TEXT,
    is_default BOOLEAN DEFAULT FALSE,
    is_active BOOLEAN DEFAULT TRUE,
    user_id UUID REFERENCES users(id) ON DELETE SET NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_rule_profiles_user_id ON rule_profiles(user_id);
```

### @TABLE: anonymization_rules

```sql
@DDL: anonymization_rules
CREATE TABLE anonymization_rules (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    profile_id VARCHAR(36) NOT NULL REFERENCES rule_profiles(id) ON DELETE CASCADE,
    entity_type VARCHAR(100) NOT NULL,
    strategy VARCHAR(100) NOT NULL,
    options JSONB DEFAULT '{}',
    enabled BOOLEAN DEFAULT TRUE,
    sequence INTEGER DEFAULT 0
);

CREATE INDEX idx_anon_rules_profile_id ON anonymization_rules(profile_id);
```

### @TABLE: tasks

```sql
@DDL: tasks
CREATE TABLE tasks (
    id VARCHAR(36) PRIMARY KEY,
    status VARCHAR(50) DEFAULT 'pending',  -- pending, processing, completed, failed, cancelled
    progress INTEGER DEFAULT 0,
    message TEXT,
    files_total INTEGER DEFAULT 0,
    files_processed INTEGER DEFAULT 0,
    total_entities_found INTEGER DEFAULT 0,
    processing_time_seconds FLOAT,
    rule_id VARCHAR(36) REFERENCES rule_profiles(id) ON DELETE SET NULL,
    session_id VARCHAR(36),
    user_id UUID REFERENCES users(id) ON DELETE SET NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    started_at TIMESTAMP,
    completed_at TIMESTAMP,
    error_message TEXT
);

CREATE INDEX idx_tasks_status ON tasks(status);
CREATE INDEX idx_tasks_rule_id ON tasks(rule_id);
CREATE INDEX idx_tasks_user_id ON tasks(user_id);
CREATE INDEX idx_tasks_session_id ON tasks(session_id);
CREATE INDEX idx_tasks_created_at ON tasks(created_at);
```

### @TABLE: task_files

```sql
@DDL: task_files
CREATE TABLE task_files (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    task_id VARCHAR(36) NOT NULL REFERENCES tasks(id) ON DELETE CASCADE,
    file_id VARCHAR(36) NOT NULL REFERENCES files(id) ON DELETE RESTRICT,
    status VARCHAR(50) DEFAULT 'pending',
    entities_found INTEGER DEFAULT 0,
    output_path VARCHAR(500),
    processing_time_seconds FLOAT,
    error_message TEXT,
    created_at TIMESTAMP DEFAULT NOW(),
    completed_at TIMESTAMP
);

CREATE INDEX idx_task_files_task_id ON task_files(task_id);
CREATE INDEX idx_task_files_file_id ON task_files(file_id);
```

### @TABLE: audit_logs

```sql
@DDL: audit_logs
CREATE TABLE audit_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    timestamp TIMESTAMP DEFAULT NOW(),
    action VARCHAR(100) NOT NULL,  -- file.upload, task.create, rule.update, etc.
    resource_type VARCHAR(50),     -- file, task, rule, user
    resource_id VARCHAR(36),
    user_id UUID REFERENCES users(id) ON DELETE SET NULL,
    session_id VARCHAR(36),
    ip_address VARCHAR(45),        -- IPv6 support
    details JSONB,
    status VARCHAR(50) DEFAULT 'success',
    error_message TEXT
);

CREATE INDEX idx_audit_logs_timestamp ON audit_logs(timestamp);
CREATE INDEX idx_audit_logs_action ON audit_logs(action);
CREATE INDEX idx_audit_logs_resource_type ON audit_logs(resource_type);
CREATE INDEX idx_audit_logs_resource_id ON audit_logs(resource_id);
CREATE INDEX idx_audit_logs_user_id ON audit_logs(user_id);
```

### 預設種子資料

```python
@SEED_DATA: default_rules
DEFAULT_RULES = [
    {
        "id": "default-medical",
        "name": "醫療研究用",
        "description": "適用於臨床研究、病歷分析。人名假名化，日期泛化到年代，保留研究關聯性。",
        "rules": [
            {"entity_type": "PERSON", "strategy": "pseudonymize", "options": {"prefix": "Patient_", "maintain_relations": True}},
            {"entity_type": "TW_ID", "strategy": "mask", "options": {}},
            {"entity_type": "PHONE_NUMBER", "strategy": "partial_mask", "options": {"keep_first": 4, "keep_last": 0}},
            {"entity_type": "EMAIL_ADDRESS", "strategy": "mask", "options": {}},
            {"entity_type": "DATE_TIME", "strategy": "generalize", "options": {"precision": "decade"}},
            {"entity_type": "LOCATION", "strategy": "generalize", "options": {"precision": "city"}},
            {"entity_type": "CREDIT_CARD", "strategy": "mask", "options": {}},
            {"entity_type": "API_KEY", "strategy": "mask", "options": {}},
        ],
    },
    {
        "id": "default-academic",
        "name": "學術研究用",
        "description": "適用於 NLP 訓練、文本分析。保留實體類型標籤，便於模型訓練。",
        "rules": [
            {"entity_type": "PERSON", "strategy": "keep_labeled", "options": {"format": "bracket"}},
            {"entity_type": "TW_ID", "strategy": "keep_labeled", "options": {"format": "bracket"}},
            {"entity_type": "PHONE_NUMBER", "strategy": "keep_labeled", "options": {"format": "bracket"}},
            {"entity_type": "EMAIL_ADDRESS", "strategy": "keep_labeled", "options": {"format": "bracket"}},
            {"entity_type": "DATE_TIME", "strategy": "keep_labeled", "options": {"format": "bracket"}},
            {"entity_type": "LOCATION", "strategy": "keep_labeled", "options": {"format": "bracket"}},
            {"entity_type": "CREDIT_CARD", "strategy": "keep_labeled", "options": {"format": "bracket"}},
            {"entity_type": "API_KEY", "strategy": "keep_labeled", "options": {"format": "bracket"}},
        ],
    },
    {
        "id": "default-compliance",
        "name": "完全合規用",
        "description": "適用於 GDPR/個資法合規。所有 PII 完全遮蔽，最高隱私保護。",
        "rules": [
            {"entity_type": "PERSON", "strategy": "mask", "options": {}},
            {"entity_type": "TW_ID", "strategy": "mask", "options": {}},
            {"entity_type": "PHONE_NUMBER", "strategy": "mask", "options": {}},
            {"entity_type": "EMAIL_ADDRESS", "strategy": "mask", "options": {}},
            {"entity_type": "DATE_TIME", "strategy": "mask", "options": {}},
            {"entity_type": "LOCATION", "strategy": "mask", "options": {}},
            {"entity_type": "CREDIT_CARD", "strategy": "mask", "options": {}},
            {"entity_type": "API_KEY", "strategy": "mask", "options": {}},
            {"entity_type": "IP_ADDRESS", "strategy": "mask", "options": {}},
            {"entity_type": "BANK_ACCOUNT", "strategy": "mask", "options": {}},
        ],
    },
]
```

---

## @SECTION: API_SPECIFICATION

### 基本資訊

```yaml
@API_INFO:
base_url: /api/v1
content_type: application/json
authentication: Bearer Token (JWT) - 目前尚未實作
```

### @ENDPOINT: POST /upload

上傳檔案

```yaml
@REQUEST:
method: POST
path: /api/v1/upload
content_type: multipart/form-data
body:
  files: File[]  # 多檔上傳

@RESPONSE: 200
{
  "success": true,
  "message": "Successfully uploaded 2 file(s)",
  "files": [
    {
      "file_id": "abc123",
      "original_name": "document.pdf",
      "file_type": "pdf",
      "size_bytes": 102400,
      "uploaded_at": "2026-01-25T10:30:00Z"
    }
  ],
  "errors": null
}

@RESPONSE: 400
{
  "detail": {
    "message": "No valid files uploaded",
    "errors": ["Invalid file type: document.exe"]
  }
}
```

### @ENDPOINT: GET /upload/{file_id}

取得檔案資訊

```yaml
@REQUEST:
method: GET
path: /api/v1/upload/{file_id}
params:
  file_id: string

@RESPONSE: 200
{
  "file_id": "abc123",
  "original_name": "document.pdf",
  "file_type": "pdf",
  "size_bytes": 102400,
  "storage_path": "uploads/2026/01/abc123.pdf",
  "uploaded_at": "2026-01-25T10:30:00Z"
}
```

### @ENDPOINT: POST /clean

啟動清洗任務

```yaml
@REQUEST:
method: POST
path: /api/v1/clean
body:
  file_ids: string[]      # 必填，檔案 ID 列表
  rule_id?: string        # 選填，規則 ID
  session_id?: string     # 選填，會話 ID (用於關聯保留)
  output_format?: string  # 選填，輸出格式

@RESPONSE: 200
{
  "success": true,
  "message": "Cleaning task started",
  "task_id": "task-uuid-123",
  "file_count": 3,
  "rule_id": "default-compliance"
}
```

### @ENDPOINT: POST /clean/preview

預覽清洗結果

```yaml
@REQUEST:
method: POST
path: /api/v1/clean/preview
params:
  file_id: string
  rule_id?: string
  sample_size?: int (default: 1000)

@RESPONSE: 200
{
  "file_id": "abc123",
  "file_name": "document.pdf",
  "rule_id": "default-compliance",
  "sample_size": 1000,
  "preview": {
    "detected_entities": [
      {"type": "PERSON", "value": "***", "count": 5},
      {"type": "PHONE_NUMBER", "value": "****", "count": 3}
    ],
    "total_entities": 8
  }
}
```

### @ENDPOINT: GET /clean/{task_id}/result

取得清洗結果

```yaml
@REQUEST:
method: GET
path: /api/v1/clean/{task_id}/result

@RESPONSE: 200
{
  "task_id": "task-uuid-123",
  "status": "completed",
  "progress": 100,
  "files": [
    {
      "file_id": "abc123",
      "original_name": "document.pdf",
      "file_type": "pdf",
      "size_bytes": 102400,
      "status": "completed",
      "entity_count": 15,
      "output_path": "output/task-123/document_cleaned.pdf",
      "processing_time_seconds": 2.5
    }
  ],
  "total_entities": 15,
  "processing_time_seconds": 3.2,
  "rule_id": "default-compliance",
  "created_at": "2026-01-25T10:30:00Z",
  "completed_at": "2026-01-25T10:30:03Z"
}
```

### @ENDPOINT: GET /tasks

列出所有任務

```yaml
@REQUEST:
method: GET
path: /api/v1/tasks
params:
  status_filter?: string  # pending, processing, completed, failed
  limit?: int (default: 50)
  offset?: int (default: 0)

@RESPONSE: 200
{
  "success": true,
  "total": 25,
  "tasks": [
    {
      "task_id": "task-uuid-123",
      "status": "completed",
      "progress": 100,
      "message": "Processing completed",
      "files_total": 3,
      "files_processed": 3,
      "created_at": "2026-01-25T10:30:00Z",
      "started_at": "2026-01-25T10:30:01Z",
      "completed_at": "2026-01-25T10:30:05Z"
    }
  ]
}
```

### @ENDPOINT: GET /tasks/{task_id}

取得任務狀態

```yaml
@REQUEST:
method: GET
path: /api/v1/tasks/{task_id}

@RESPONSE: 200
{
  "task_id": "task-uuid-123",
  "status": "processing",
  "progress": 67,
  "message": "Processing file 2 of 3...",
  "files_total": 3,
  "files_processed": 2,
  "created_at": "2026-01-25T10:30:00Z",
  "started_at": "2026-01-25T10:30:01Z"
}
```

### @ENDPOINT: DELETE /tasks/{task_id}

取消任務

```yaml
@REQUEST:
method: DELETE
path: /api/v1/tasks/{task_id}

@RESPONSE: 200
{
  "success": true,
  "message": "Task task-uuid-123 cancelled"
}

@RESPONSE: 400
{
  "detail": "Cannot cancel task with status: completed"
}
```

### @ENDPOINT: GET /tasks/{task_id}/download

下載任務結果

```yaml
@REQUEST:
method: GET
path: /api/v1/tasks/{task_id}/download

@RESPONSE: 200
Content-Type: application/zip
Content-Disposition: attachment; filename="task_abc123_results.zip"
```

### @ENDPOINT: GET /tasks/{task_id}/report

取得任務報告

```yaml
@REQUEST:
method: GET
path: /api/v1/tasks/{task_id}/report

@RESPONSE: 200
{
  "task_id": "task-uuid-123",
  "report": {
    "summary": {
      "files_processed": 3,
      "files_total": 3,
      "total_entities_found": 45,
      "processing_time_seconds": 5.2,
      "status": "completed"
    },
    "entities_by_file": [
      {"file_id": "abc123", "entity_count": 15, "status": "completed"},
      {"file_id": "def456", "entity_count": 20, "status": "completed"},
      {"file_id": "ghi789", "entity_count": 10, "status": "completed"}
    ],
    "applied_rule": "default-compliance",
    "session_id": "session-001"
  }
}
```

### @ENDPOINT: GET /rules

列出規則

```yaml
@REQUEST:
method: GET
path: /api/v1/rules
params:
  include_defaults?: bool (default: true)

@RESPONSE: 200
{
  "success": true,
  "total": 5,
  "rules": [
    {
      "id": "default-compliance",
      "name": "完全合規用",
      "description": "適用於 GDPR/個資法合規...",
      "rules": [
        {"entity": "PERSON", "strategy": "mask", "options": {}},
        {"entity": "TW_ID", "strategy": "mask", "options": {}}
      ],
      "is_default": true
    }
  ]
}
```

### @ENDPOINT: GET /rules/{rule_id}

取得單一規則

```yaml
@REQUEST:
method: GET
path: /api/v1/rules/{rule_id}

@RESPONSE: 200
{
  "id": "custom-rule-001",
  "name": "自訂規則",
  "description": "我的自訂規則",
  "rules": [
    {"entity": "PERSON", "strategy": "pseudonymize", "options": {"prefix": "User_"}},
    {"entity": "EMAIL_ADDRESS", "strategy": "partial_mask", "options": {}}
  ],
  "is_default": false
}
```

### @ENDPOINT: POST /rules

建立規則

```yaml
@REQUEST:
method: POST
path: /api/v1/rules
body:
  name: string
  description?: string
  rules: [
    {
      entity: string,
      strategy: string,
      options?: object
    }
  ]

@RESPONSE: 201
{
  "id": "generated-uuid",
  "name": "新規則",
  "description": "規則描述",
  "rules": [...],
  "is_default": false
}
```

### @ENDPOINT: PUT /rules/{rule_id}

更新規則

```yaml
@REQUEST:
method: PUT
path: /api/v1/rules/{rule_id}
body:
  name?: string
  description?: string
  rules?: [...]

@RESPONSE: 200
{
  "id": "rule-id",
  "name": "更新後的名稱",
  ...
}

@RESPONSE: 400
{
  "detail": "Cannot modify default rules. Create a copy instead."
}
```

### @ENDPOINT: DELETE /rules/{rule_id}

刪除規則

```yaml
@REQUEST:
method: DELETE
path: /api/v1/rules/{rule_id}

@RESPONSE: 200
{
  "success": true,
  "message": "Rule rule-id deleted"
}
```

### @ENDPOINT: POST /rules/export

匯出規則

```yaml
@REQUEST:
method: POST
path: /api/v1/rules/export
body:
  rule_ids?: string[]  # 選填，未指定則匯出所有自訂規則

@RESPONSE: 200
{
  "success": true,
  "export_version": "1.0",
  "rules": [...]
}
```

### @ENDPOINT: POST /rules/import

匯入規則

```yaml
@REQUEST:
method: POST
path: /api/v1/rules/import
body: [
  {
    "name": "匯入規則",
    "description": "...",
    "rules": [...]
  }
]

@RESPONSE: 200
{
  "success": true,
  "message": "Imported 2 rule(s)",
  "rule_ids": ["new-id-1", "new-id-2"]
}
```

### @ENDPOINT: GET /download/{task_id}

下載所有檔案

```yaml
@REQUEST:
method: GET
path: /api/v1/download/{task_id}
params:
  format?: string (default: "zip")

@RESPONSE: 200
Content-Type: application/zip
Content-Disposition: attachment; filename="task_abc123_results.zip"
```

### @ENDPOINT: GET /download/{task_id}/{file_id}

下載單一檔案

```yaml
@REQUEST:
method: GET
path: /api/v1/download/{task_id}/{file_id}

@RESPONSE: 200
Content-Type: text/plain; charset=utf-8
Content-Disposition: attachment; filename="document_cleaned.txt"
```

### @ENDPOINT: GET /download/{task_id}/report

下載報告

```yaml
@REQUEST:
method: GET
path: /api/v1/download/{task_id}/report
params:
  format?: string (default: "json")  # json, csv, pdf

@RESPONSE: 200
{
  "task_id": "...",
  "generated_at": "2026-01-25T10:35:00Z",
  "summary": {...},
  "timeline": {...},
  "files": [...]
}
```

### @ENDPOINT: WebSocket /ws/tasks/{task_id}

任務即時更新

```yaml
@WEBSOCKET:
path: /ws/tasks/{task_id}

@MESSAGE: connected
{
  "type": "connected",
  "task_id": "task-uuid-123",
  "message": "Connected to task updates"
}

@MESSAGE: task_update
{
  "type": "task_update",
  "task_id": "task-uuid-123",
  "timestamp": "2026-01-25T10:30:02Z",
  "data": {
    "status": "processing",
    "progress": 33,
    "message": "Processing file 1 of 3",
    "files_processed": 1,
    "files_total": 3,
    "current_file": "document.pdf",
    "entities_found": 5
  }
}

@MESSAGE: ping/pong
{
  "type": "ping"  // Server sends
}
{
  "type": "pong"  // Client responds
}
```

### @ENDPOINT: WebSocket /ws/all

全域任務更新

```yaml
@WEBSOCKET:
path: /ws/all

@MESSAGE: broadcast
{
  "type": "broadcast",
  "timestamp": "2026-01-25T10:30:02Z",
  "data": {
    "task_id": "task-uuid-123",
    "status": "completed",
    "progress": 100
  }
}
```

### @ENDPOINT: GET /health

健康檢查

```yaml
@REQUEST:
method: GET
path: /health

@RESPONSE: 200
{
  "status": "healthy",
  "version": "0.1.0"
}
```

---

## @SECTION: FRONTEND_ARCHITECTURE

### 頁面結構

```
@ROUTES:
/                   → HomePage (檔案上傳 + 快速開始)
/rules              → RulesPage (規則管理)
/tasks              → TasksPage (任務列表 + 進度追蹤)
```

### 元件樹

```
@COMPONENT_TREE:
App
├── Layout
│   ├── Header (導航)
│   └── Main (內容區)
├── Pages
│   ├── HomePage
│   │   ├── FileUploader (拖放上傳)
│   │   └── ResultPreview (結果預覽)
│   ├── RulesPage
│   │   └── RuleEditor (規則編輯器)
│   └── TasksPage
│       ├── TaskManager (任務列表)
│       └── DiffView (差異比較)
└── Hooks
    └── useWebSocket (即時更新)
```

### TypeScript 型別定義

```typescript
@TYPES: frontend/src/types/index.ts

export type EntityType =
  | 'PERSON'
  | 'TW_ID'
  | 'PHONE_NUMBER'
  | 'EMAIL_ADDRESS'
  | 'ADDRESS'
  | 'DATE_TIME'
  | 'CREDIT_CARD'
  | 'BANK_ACCOUNT'
  | 'IP_ADDRESS'
  | 'API_KEY'
  | 'MEDICAL_LICENSE'

export type StrategyType =
  | 'mask'
  | 'partial_mask'
  | 'pseudonymize'
  | 'generalize'
  | 'keep_labeled'
  | 'encrypt'

export interface EntityTypeInfo {
  type: EntityType
  label: string      // 中文標籤
  description: string
}

export const ENTITY_TYPES: EntityTypeInfo[] = [
  { type: 'PERSON', label: '人名', description: '姓名、暱稱' },
  { type: 'TW_ID', label: '身分證字號', description: '台灣身分證字號' },
  { type: 'PHONE_NUMBER', label: '電話號碼', description: '手機、市話' },
  { type: 'EMAIL_ADDRESS', label: '電子郵件', description: 'Email 地址' },
  { type: 'ADDRESS', label: '地址', description: '實體地址' },
  { type: 'DATE_TIME', label: '日期時間', description: '日期、時間、生日' },
  { type: 'CREDIT_CARD', label: '信用卡號', description: '信用卡/簽帳卡號碼' },
  { type: 'BANK_ACCOUNT', label: '銀行帳號', description: '銀行帳戶號碼' },
  { type: 'IP_ADDRESS', label: 'IP 位址', description: 'IPv4/IPv6 位址' },
  { type: 'API_KEY', label: 'API 金鑰', description: 'API Key、Token' },
  { type: 'MEDICAL_LICENSE', label: '醫療執照', description: '醫療執照號碼' },
]

export const STRATEGIES: StrategyInfo[] = [
  { type: 'mask', label: '完全遮蔽', description: '以符號完全替換' },
  { type: 'partial_mask', label: '部分遮蔽', description: '保留部分字元' },
  { type: 'pseudonymize', label: '假名化', description: '替換為假資料' },
  { type: 'generalize', label: '泛化', description: '降低精確度' },
  { type: 'keep_labeled', label: '保留標籤', description: '標記但不移除' },
  { type: 'encrypt', label: '加密替換', description: '可逆加密' },
]
```

### API 客戶端

```typescript
@API_CLIENT: frontend/src/services/api.ts

import axios from 'axios'

const API_BASE_URL = import.meta.env.VITE_API_URL || 'http://localhost:8000'

const api = axios.create({
  baseURL: `${API_BASE_URL}/api/v1`,
  headers: {
    'Content-Type': 'application/json',
  },
})

// 主要 API 函數
export const uploadFiles = async (files: File[]): Promise<UploadResponse>
export const startCleaningTask = async (request: CleanRequest): Promise<CleanResponse>
export const getTaskStatus = async (taskId: string): Promise<TaskStatus>
export const listTasks = async (status?: string): Promise<{ tasks: TaskStatus[] }>
export const listRules = async (): Promise<{ rules: RuleProfile[] }>
export const createRule = async (rule: Omit<RuleProfile, 'id' | 'is_default'>): Promise<RuleProfile>
export const updateRule = async (ruleId: string, rule: ...): Promise<RuleProfile>
export const deleteRule = async (ruleId: string): Promise<void>
export const getTaskResult = async (taskId: string): Promise<TaskResult>
export const downloadFile = async (taskId: string, fileId: string): Promise<Blob>
export const downloadAllFiles = async (taskId: string): Promise<Blob>
```

---

## @SECTION: DEPLOYMENT

### Docker Compose 服務定義

```yaml
@DOCKER_COMPOSE: docker-compose.yml

version: '3.8'

services:
  # 後端 API
  backend:
    build: ./backend
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql://postgres:postgres@db:5432/rag_clean
      - REDIS_URL=redis://redis:6379/0
      - MINIO_ENDPOINT=minio:9000
      - MINIO_ACCESS_KEY=minioadmin
      - MINIO_SECRET_KEY=minioadmin
      - MINIO_BUCKET=rag-clean
    depends_on:
      - db
      - redis
      - minio
    volumes:
      - ./backend/app:/app/app
      - ./data/input:/app/data/input
      - ./data/output:/app/data/output

  # Celery Worker
  celery-worker:
    build: ./backend
    command: celery -A app.tasks.celery_app worker --loglevel=info
    environment:
      - DATABASE_URL=postgresql://postgres:postgres@db:5432/rag_clean
      - REDIS_URL=redis://redis:6379/0
      - MINIO_ENDPOINT=minio:9000
    depends_on:
      - db
      - redis
      - minio

  # 前端
  frontend:
    build: ./frontend
    ports:
      - "3000:3000"
    environment:
      - VITE_API_URL=http://localhost:8000
    depends_on:
      - backend

  # PostgreSQL
  db:
    image: postgres:16-alpine
    ports:
      - "5432:5432"
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=postgres
      - POSTGRES_DB=rag_clean
    volumes:
      - postgres_data:/var/lib/postgresql/data

  # Redis
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data

  # MinIO
  minio:
    image: minio/minio:latest
    ports:
      - "9000:9000"
      - "9001:9001"
    environment:
      - MINIO_ROOT_USER=minioadmin
      - MINIO_ROOT_PASSWORD=minioadmin
    command: server /data --console-address ":9001"
    volumes:
      - minio_data:/data

  # Nginx (Production 專用)
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    profiles:
      - production

volumes:
  postgres_data:
  redis_data:
  minio_data:
```

### 環境變數清單

```bash
@ENV: .env.example

# Database
DATABASE_URL=postgresql://postgres:postgres@db:5432/rag_clean

# Redis
REDIS_URL=redis://redis:6379/0

# MinIO (S3-compatible storage)
MINIO_ENDPOINT=minio:9000
MINIO_ACCESS_KEY=minioadmin
MINIO_SECRET_KEY=minioadmin
MINIO_BUCKET=rag-clean

# Application
APP_ENV=development
DEBUG=true
SECRET_KEY=your-secret-key-here-change-in-production

# File Processing
INPUT_DIR=/app/data/input
OUTPUT_DIR=/app/data/output
MAX_FILE_SIZE_MB=100

# Celery
CELERY_BROKER_URL=redis://redis:6379/0
CELERY_RESULT_BACKEND=redis://redis:6379/0
```

### Backend Dockerfile

```dockerfile
@DOCKERFILE: backend/Dockerfile

FROM python:3.11-slim

WORKDIR /app

# 系統依賴
RUN apt-get update && apt-get install -y \
    build-essential \
    libpq-dev \
    poppler-utils \
    tesseract-ocr \
    tesseract-ocr-chi-tra \
    && rm -rf /var/lib/apt/lists/*

# Python 依賴
COPY pyproject.toml ./
RUN pip install --no-cache-dir --upgrade pip && \
    pip install --no-cache-dir .

# 下載 spaCy 模型
RUN python -m spacy download en_core_web_trf && \
    python -m spacy download zh_core_web_trf

# 複製應用程式碼
COPY app ./app

# 建立資料目錄
RUN mkdir -p /app/data/input /app/data/output

EXPOSE 8000

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--reload"]
```

### 系統依賴

```yaml
@DEPENDENCIES:
python_packages:
  # Web Framework
  - fastapi>=0.109.0
  - uvicorn[standard]>=0.27.0
  - python-multipart>=0.0.6

  # Database
  - sqlalchemy>=2.0.0
  - asyncpg>=0.29.0
  - alembic>=1.13.0

  # Redis & Celery
  - redis>=5.0.0
  - celery>=5.3.0

  # MinIO (S3)
  - minio>=7.2.0

  # PII Detection
  - presidio-analyzer>=2.2.0
  - presidio-anonymizer>=2.2.0
  - spacy>=3.7.0

  # Document Parsing
  - pdfminer.six>=20231228
  - python-docx>=1.1.0
  - openpyxl>=3.1.0
  - pandas>=2.1.0

  # NLP & Language Detection
  - langdetect>=1.0.9

  # Validation & Settings
  - pydantic>=2.5.0
  - pydantic-settings>=2.1.0

spacy_models:
  - en_core_web_trf  # 英文 Transformer 模型
  - zh_core_web_trf  # 中文 Transformer 模型

npm_packages:
  - react@^18.2.0
  - react-dom@^18.2.0
  - react-router-dom@^6.21.2
  - "@tanstack/react-query@^5.17.0"
  - axios@^1.6.5
  - tailwindcss@^3.4.1
  - typescript@^5.3.3
  - vite@^5.0.12
  - lucide-react@^0.312.0
  - react-dropzone@^14.2.3
```

---

## @SECTION: IMPLEMENTATION_CHECKLIST

### Phase 1: 專案初始化 ✅

- [x] 建立專案目錄結構
- [x] 設定 Docker Compose
- [x] 建立 FastAPI 應用框架
- [x] 設定 Pydantic Settings
- [x] 建立 React 前端框架
- [x] 設定 Tailwind CSS

### Phase 2: 資料庫與模型 ✅

- [x] 設計資料庫 Schema
- [x] 建立 SQLAlchemy ORM 模型
  - [x] User
  - [x] File
  - [x] RuleProfile / AnonymizationRule
  - [x] Task / TaskFile
  - [x] AuditLog
- [x] 設定 Alembic 遷移
- [x] 建立 Repository 層
- [x] 建立預設規則種子資料

### Phase 3: PII 偵測引擎 ✅

- [x] 整合 Microsoft Presidio
- [x] 設定 spaCy NLP 引擎
- [x] 實作自訂識別器
  - [x] TaiwanIDRecognizer (身分證)
  - [x] TaiwanPhoneRecognizer (電話)
  - [x] TaiwanUnifiedBusinessNoRecognizer (統編)
  - [x] APIKeyRecognizer (API 金鑰)
  - [x] AccessTokenRecognizer
  - [x] PrivateKeyRecognizer
  - [x] DatabaseCredentialRecognizer
- [x] 實作語言自動偵測

### Phase 4: 脫敏策略實作 ✅

- [x] 策略基類設計
- [x] 實作 6 種策略
  - [x] MaskStrategy
  - [x] PartialMaskStrategy
  - [x] PseudonymizeStrategy
  - [x] GeneralizeStrategy
  - [x] KeepLabeledStrategy
  - [x] EncryptStrategy
- [x] 實作 RelationKeeper
- [x] 整合 AnonymizationEngine

### Phase 5: 文件解析模組 ✅

- [x] 設計 Parser 介面
- [x] 實作 ParserFactory
- [x] 實作各格式解析器
  - [x] PDFParser
  - [x] DocxParser
  - [x] ExcelParser
  - [x] TextParser
- [x] 實作語言偵測器

### Phase 6: API 端點 ✅

- [x] 檔案上傳 API
- [x] 清洗任務 API
- [x] 任務狀態 API
- [x] 規則管理 API
- [x] 下載 API
- [x] WebSocket 即時更新

### Phase 7: 前端介面 (進行中)

- [x] 基本路由設定
- [x] API 客戶端
- [x] TypeScript 型別定義
- [ ] FileUploader 元件完善
- [ ] RuleEditor 元件完善
- [ ] TaskManager 元件完善
- [ ] ResultPreview 元件完善
- [ ] DiffView 元件完善

### Phase 8: 進階功能 (待實作)

- [ ] Celery 任務處理整合
- [ ] MinIO 檔案儲存整合
- [ ] 使用者認證 (JWT)
- [ ] 批次處理優化
- [ ] 稽核日誌完善
- [ ] 效能監控

---

## @SECTION: VALIDATION

### 驗證方法

```bash
# 1. 啟動服務
docker-compose up -d

# 2. 等待服務就緒
sleep 30

# 3. 驗證健康檢查
curl http://localhost:8000/health
# Expected: {"status": "healthy", "version": "0.1.0"}

# 4. 驗證 API 文件
open http://localhost:8000/docs

# 5. 驗證前端
open http://localhost:3000
```

### 功能驗證清單

- [ ] 檔案上傳成功
- [ ] PII 偵測正確識別台灣身分證
- [ ] 脫敏策略正確套用
- [ ] 任務狀態即時更新
- [ ] 規則 CRUD 正常運作
- [ ] 結果下載成功

---

## 文件版本歷程

| 版本 | 日期 | 說明 |
|------|------|------|
| 1.0.0 | 2026-01-25 | 初始版本 |

---

> **注意**：本文件為 AI 可解析格式，使用 `@SECTION`、`@MODULE`、`@CLASS`、`@INTERFACE`、`@TABLE`、`@ENDPOINT` 等標記提供語義標註，便於 AI 精準定位並複刻系統。
