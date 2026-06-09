# X2X 實作細節說明（Informatica → Pentaho）

## 概覽

X2X 解決方案可分為三個核心模組：

1. **X2XAnalyzer**：分析 Informatica 現有工作負載、元資料、依賴、複雜度與風險。
2. **X2XConverter**：將 Informatica artifact 轉換為 Pentaho 可執行 artifact。
3. **X2XValidator**：驗證轉換後資料與流程結果是否與原系統一致。

整體目標唔係單純做 XML-to-XML 轉換，而係做一個 **保留 ETL 語意（semantic-preserving）** 嘅 migration framework，盡量確保行為、資料結果、控制流同埋營運特性可以對齊。

---

# 1. X2XAnalyzer

## 1.1 目標

X2XAnalyzer 主要負責喺 migration 前，對 Informatica 環境做完整理解，建立資產清單、依賴圖、複雜度評分、風險識別，同埋輸出 migration planning 所需資料。

---

## 1.2 Input

### 主要輸入

- Informatica repository export / XML export
- mappings
- mapplets
- workflows
- worklets
- sessions
- tasks
- source definitions
- target definitions
- transformations metadata
- connectors / links
- parameter files / variables
- pre-SQL / post-SQL
- session properties
- connection metadata
- folder / project structure

### 補充輸入

- Informatica 版本資料
- execution logs（如有）
- job run history（如有）
- source/target database metadata
- 自定義 transformation / external script 說明
- business rule 文件（如有）

---

## 1.3 Process

### Step 1：Metadata Ingestion

- 讀取 Informatica export XML / repository metadata
- 根據版本識別 schema 差異
- 做 XML parsing、schema validation、reference resolution
- 抽取所有 object 同 object relationship

### Step 2：Canonical Model Normalization

將 Informatica 原生表示方式轉成內部 canonical model，例如：

- Workflow
- Session
- Mapping
- Transformation
- Field mapping
- Parameter / Variable
- Source / Target
- Runtime property
- Dependency edge

目的係令後續分析同轉換唔需要直接依賴 vendor-specific XML 結構。

### Step 3：Dependency Analysis

建立以下依賴：

- workflow → session
- session → mapping
- mapping → transformation chain
- transformation → source / target
- reusable object → dependent mappings
- parameter / variable usage
- database / file / external system dependency

輸出 dependency graph，方便判斷 migration 範圍同批次。

### Step 4：Complexity Analysis

為每個 object 計分，例如：

- transformation 數量
- expression 複雜度
- lookup 數量與類型
- join 邏輯複雜度
- aggregator / router / update strategy 使用情況
- custom code / script 使用情況
- external dependency 數量
- workflow orchestration 深度
- parameterization 程度

### Step 5：Risk Analysis

識別高風險項目，例如：

- custom transformation
- unsupported component
- complex reusable mapplet
- dynamic lookup / cache semantics
- transaction / commit behavior
- error handling / reject logic
- restart / recovery requirement
- OS command / shell / script integration
- hard-coded environment setting
- embedded SQL
- undocumented business rule

### Step 6：Migration Readiness Classification

分類每個 asset：

- Auto-convertable
- Partially auto-convertable
- Manual redesign required
- Validation-intensive
- Unsupported / blocked

---

## 1.4 Output

### 主要輸出

- 資產清單（Inventory Report）
- 依賴關係圖（Dependency Graph）
- 複雜度評分報告（Complexity Report）
- 風險報告（Risk Assessment）
- Migration readiness 報告
- 版本與兼容性分析
- Conversion scope 建議
- 優先次序與 migration wave 建議

### 輸出格式建議

- JSON（供系統後續處理）
- HTML / PDF（供管理層與顧問閱讀）
- CSV / Excel（供 PM tracking）
- Graph format（例如 dependency visualization）

---

## 1.5 Implementation Approach

### A. Parser Layer

實作一個 Informatica metadata parser：

- XML parser
- schema-aware parser
- version adapter
- reference resolver

建議設計成 plug-in architecture，方便之後支援不同 Informatica 版本。

### B. Canonical Metadata Model

建立中間層資料模型，例如：

- `Project`
- `Workflow`
- `Job`
- `TransformationNode`
- `FieldExpression`
- `ConnectionDefinition`
- `RuntimeConfig`
- `DependencyEdge`

呢層係成個平台最重要基礎，Analyzer、Converter、Validator 都應共用。

### C. Rule Engine

實作分析規則：

- complexity scoring rules
- unsupported pattern rules
- risk scoring rules
- readiness classification rules

### D. Report Generator

將 canonical model + rule result 輸出成報告。

### E. 技術建議

- Backend：Java / Kotlin / Python 都可
- XML parsing：Jackson XML / JAXB / lxml / SAX + DOM hybrid
- Graph analysis：NetworkX / JGraphT
- Rules：Drools / 自定義 rule engine
- Output：JSON + Markdown + PDF generator

---

# 2. X2XConverter

## 2.1 目標

X2XConverter 負責將 Informatica mappings / workflows / sessions 轉換成 Pentaho 可執行 job / transformation artifacts，並保留盡可能多嘅 ETL 語意、控制流同 runtime 設定。

---

## 2.2 Input

### 主要輸入

- X2XAnalyzer 輸出嘅 canonical model
- Informatica 原始 XML / metadata
- Informatica 版本資訊
- Pentaho 目標版本資訊
- transformation mapping catalog
- compatibility matrix
- manual override configuration
- environment mapping configuration

### 必要知識輸入

#### Informatica schema / metadata model
包括：
- mappings
- transformations
- links
- workflow/session/task metadata
- parameter / variable model
- runtime behavior metadata

#### Pentaho schema / metadata model
包括：
- job definition
- transformation definition
- step config
- hop config
- database connection config
- parameter / variable config
- logging / error handling metadata

#### Mapping specification
例如：
- Source Qualifier → Table Input / equivalent extraction pattern
- Expression → Calculator / Formula / JS / UDJC pattern
- Lookup → Stream Lookup / DB Lookup
- Joiner → Merge Join / DB Join strategy
- Aggregator → Group By
- Filter → Filter Rows
- Router → multiple Filter Rows paths
- Update Strategy → insert/update/delete control pattern

---

## 2.3 Process

### Step 1：Load Canonical Model

讀取 Analyzer 輸出嘅 normalized metadata，避免直接對 vendor XML 做硬編碼轉換。

### Step 2：Transformation Mapping

將每個 Informatica transformation 對應到：

- 1:1 Pentaho step
- 1:n Pentaho step pattern
- n:1 consolidation pattern
- unsupported/manual remediation item

### Step 3：Workflow / Control Flow Conversion

轉換：

- workflow → Pentaho job
- session / task → job entry / transformation invocation
- dependencies → hop / execution order
- conditional logic → success/failure branch
- parameter passing → variable / parameter injection

### Step 4：Field-Level Conversion

對每個欄位做 mapping：

- datatype mapping
- expression translation
- null handling semantics
- default value behavior
- trimming / casting / formatting behavior
- lookup return field behavior
- aggregate field derivation

### Step 5：Runtime Property Conversion

轉換執行相關設定：

- commit size
- logging level
- error threshold
- reject handling
- cache option
- sorting requirement
- partitioning / parallelism（如適用）
- retry / restart semantics（若 Pentaho 可模擬）

### Step 6：Generate Pentaho Artifacts

輸出：

- `.ktr` transformation files
- `.kjb` job files
- shared database connections
- parameter files
- environment-specific config templates
- conversion manifest

### Step 7：Exception Packaging

對無法完全自動轉換部分輸出 remediation report：

- unsupported transformation
- custom code migration required
- manual SQL review required
- orchestration redesign required
- runtime semantic mismatch warning

---

## 2.4 Output

### 主要輸出

- Pentaho transformation files (`.ktr`)
- Pentaho job files (`.kjb`)
- database connection definitions
- parameter / variable template
- conversion manifest
- unsupported items report
- warning / remediation report
- conversion traceability report

### Traceability 內容

每一個 Pentaho object 最好可追蹤返：

- 來源 Informatica object
- conversion rule ID
- conversion timestamp
- warning / fallback decision
- manual override record

---

## 2.5 Implementation Approach

### A. Canonical-to-Canonical Strategy

唔建議直接寫死 Informatica XML → Pentaho XML。建議用：

**Informatica XML → Canonical Model → Pentaho Model → Pentaho XML**

好處：
- 可維護性高
- 易於支援多版本
- 易於加其他 target platform
- 易於測試

### B. Mapping Catalog

建立 conversion catalog，例如：

| Informatica | Pentaho | 類型 | 備註 |
|---|---|---|---|
| Source Qualifier | Table Input | 1:1 / pattern | 視 SQL pushdown 情況 |
| Expression | Calculator / Formula / JS | 1:n | 視 expression 複雜度 |
| Lookup | DB Lookup / Stream Lookup | 1:n | 視 cache / source 類型 |
| Router | Filter Rows + hops | 1:n | 需建立多分支 |
| Aggregator | Group By | 1:1 | 注意 null/group semantics |
| Update Strategy | custom pattern | pattern | 可能需手動處理 |

### C. Template-based Generator

用 template 或 object serializer 產生 Pentaho XML：

- step template
- job entry template
- connection template
- logging / error handling template

### D. Translation Engine

對 expression / formula / SQL 做 translation：

- tokenizer
- parser
- AST builder
- target expression renderer

特別係 expression conversion，唔好只靠 string replace。

### E. Manual Override Framework

加入人工覆蓋配置：

- object-level override
- transformation-level override
- field-level override
- environment mapping override

### F. 技術建議

- Domain model：Java / Kotlin 較適合大型 rule-based converter
- XML generation：JAXB / Jackson XML / templating engine
- Expression parsing：ANTLR / 自定義 parser
- Config：YAML / JSON
- Traceability：SQLite / JSON manifest

---

# 3. X2XValidator

## 3.1 目標

X2XValidator 負責驗證 migration 後 Pentaho 流程喺資料結果、欄位邏輯、業務規則、控制流與執行品質上，是否與原 Informatica 達到可接受一致性。

---

## 3.2 Input

### 主要輸入

- 原 Informatica job / mapping metadata
- 已轉換 Pentaho job / transformation metadata
- source dataset
- target dataset
- baseline run result（Informatica）
- migrated run result（Pentaho）
- validation rulebook
- schema mapping
- key field definition
- business rule definition
- tolerance thresholds

### 可選輸入

- 歷史 execution logs
- sample golden datasets
- SLA / performance benchmark
- exception whitelist

---

## 3.3 Process

### Step 1：Validation Scope Definition

定義驗證層次：

- schema-level
- row-count-level
- field-value-level
- aggregate-level
- business-rule-level
- process-level
- performance-level

### Step 2：Schema Validation

檢查：

- table / file structure
- field names
- datatype mapping
- field length / precision / scale
- nullable / default setting

### Step 3：Data Reconciliation

比較：

- row count
- distinct count
- null count
- min / max
- sum / avg
- checksum / hash
- primary key match rate
- unmatched rows
- duplicate rows

### Step 4：Field-Level Comparison

逐欄比較：

- exact match
- tolerance match（例如 decimal / timestamp）
- transformed value correctness
- code mapping correctness
- derived field correctness

### Step 5：Business Rule Validation

驗證重要商業邏輯：

- 條件分流結果
- lookup enrichment 結果
- aggregation 邏輯
- update/insert decision
- reject handling
- exception classification

### Step 6：Process Validation

檢查：

- job flow order
- dependency execution
- success/failure branch behavior
- parameter passing correctness
- error logging
- restart / rerun behavior

### Step 7：Performance Validation

比較：

- total runtime
- throughput
- step bottleneck
- DB read/write time
- memory / CPU pattern（如有監控）

### Step 8：Exception Classification

將不一致分類：

- expected difference
- configuration issue
- conversion issue
- source data timing issue
- environment issue
- business rule mismatch

---

## 3.4 Output

### 主要輸出

- validation summary report
- schema validation report
- data reconciliation report
- field-level mismatch report
- business rule validation report
- process validation checklist
- performance comparison report
- final migration acceptance score

### 結果分類

每個 object / job 可標示為：

- Passed
- Passed with Warning
- Failed
- Manual Review Required

---

## 3.5 Implementation Approach

### A. Validation Rulebook

建立規則庫：

- schema rules
- datatype tolerance rules
- key-based compare rules
- aggregate compare rules
- business rule assertions
- performance threshold rules

### B. Comparison Engine

實作多層比較引擎：

1. metadata compare
2. dataset profiling
3. key-based row compare
4. aggregate compare
5. exception summarization

### C. Data Access Abstraction

提供統一 connector：

- Oracle
- SQL Server
- DB2
- MySQL
- PostgreSQL
- flat file / CSV
- parquet（如需要）

### D. Sampling + Full Compare Strategy

實務上可支援：

- sampling compare（快）
- full reconciliation（準）
- key range compare
- date partition compare

### E. Test Pack Generation

可由 Analyzer / Converter 自動生成 validation test case，例如：

- source-target row count test
- field checksum test
- business key match test
- aggregate parity test

### F. 技術建議

- Validation engine：Python / Java 都可
- Data compare：Spark / Pandas / SQL-based compare framework
- Reporting：HTML / Markdown / Excel / dashboard
- Scheduling：可整合 CI/CD 或 migration pipeline

---

# 4. 三個模組之間嘅關係

## 4.1 End-to-End Flow

1. **X2XAnalyzer** 讀取 Informatica metadata，做 inventory、風險與複雜度分析。
2. **X2XConverter** 根據 Analyzer 輸出嘅 canonical model 同 mapping catalog 產生 Pentaho artifacts。
3. **X2XValidator** 根據來源與目標結果，驗證 migration 是否成功。

---

## 4.2 共用核心能力

三個模組應共用以下基礎組件：

### 共用 1：Canonical Model
- 減少重覆解析
- 統一 object representation
- 提升可測試性

### 共用 2：Version Compatibility Layer
- 處理 Informatica / Pentaho 不同版本差異

### 共用 3：Rule Repository
- Analyzer risk rules
- Converter mapping rules
- Validator assertion rules

### 共用 4：Traceability Repository
記錄：
- source object → target object mapping
- rule applied
- warning generated
- manual override
- validation result

---

# 5. 實作建議架構

## 5.1 推薦模組化架構

### Layer 1：Ingestion Layer
- XML loader
- schema validator
- parser
- repository importer

### Layer 2：Canonical Modeling Layer
- normalized ETL object model
- dependency graph model
- runtime metadata model

### Layer 3：Rule & Mapping Layer
- complexity rules
- risk rules
- transformation mapping rules
- validation assertion rules

### Layer 4：Execution Layer
- Analyzer engine
- Converter engine
- Validator engine

### Layer 5：Artifact & Reporting Layer
- Pentaho generator
- report generator
- manifest generator
- dashboard exporter

---

## 5.2 建議資料流

- Informatica XML → Parser → Canonical Model
- Canonical Model → Analyzer Rules → Assessment Reports
- Canonical Model → Converter Rules → Pentaho Model → Pentaho XML
- Source + Target + Rulebook → Validator Engine → Validation Reports

---

# 6. MVP 建議範圍

如果要先做 MVP，建議優先支援最常見 transformation：

- Source Qualifier
- Expression
- Filter
- Lookup
- Joiner
- Aggregator
- Router
- Sorter
- Sequence
- Target load

Workflow 方面先支援：
- basic session flow
- success/failure path
- parameter passing
- simple scheduling metadata extraction

Validator 方面先支援：
- schema compare
- row count compare
- checksum compare
- key-based sample compare
- aggregate compare

---

# 7. 關鍵成功因素

## 必須具備

- 完整 Informatica schema / metadata understanding
- 完整 Pentaho artifact understanding
- 詳細 transformation mapping matrix
- canonical model 設計
- exception handling 機制
- traceability 機制
- validation framework

## 常見失敗原因

- 只做 XML 字串轉換，冇 semantic layer
- 忽略 runtime behavior 差異
- 冇 version compatibility strategy
- 冇 manual remediation flow
- 冇 validation evidence

---

# 8. 總結

X2XAnalyzer、X2XConverter 同 X2XValidator 應該被設計成一個完整 migration platform，而唔係三個獨立 script。

- **X2XAnalyzer** 重點係「理解現況、識別風險、計劃 migration」。
- **X2XConverter** 重點係「保留 ETL 語意，生成可維護 Pentaho artifacts」。
- **X2XValidator** 重點係「證明 migration 結果正確、可接受、可上線」。

如果設計得好，呢三個模組可以形成一條完整流水線：

**Analyze → Convert → Validate**

並且支援：
- 自動化 migration
- 半自動 remediation
- 可審計 traceability
- 可量化 validation

呢個先係 enterprise-grade Informatica → Pentaho migration solution 應有嘅實作方向。
