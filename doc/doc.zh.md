# code-reviewer - 技術文件

> 返回 [README](./README.zh.md)

## 前置需求

- [Claude Code](https://claude.ai/claude-code) CLI 已安裝並設定
- Python 3.10 或以上（使用內建 `ast` 模組與現代型別語法）
- Go 1.21 或以上（選用，僅 Go 專案 AST 分析需要；不可用時自動降級為字串掃描）
- 專案本地 `eslint`（選用，僅 JS/TS 專案 lint 規則整合需要）

## 安裝

### 從 GitHub Clone

```bash
git clone https://github.com/pardnchiu/skill-code-reviewer.git \
    ~/.claude/skills/code-reviewer
```

### 手動安裝

將下列檔案放置於 `~/.claude/skills/code-reviewer/`：

```
code-reviewer/
├── scripts/
│   ├── analyze_code.py              # 進入點：語言偵測 + 分派
│   ├── analyze_go.py                # Go 分析器
│   ├── analyze_python.py            # Python 分析器
│   ├── analyze_js_ts.py             # JavaScript/TypeScript 分析器
│   ├── common.py                    # 共用型別與偵測工具
│   ├── go_ast.go                    # Go AST helper（go run 執行）
│   ├── analysis_categories.md       # 偵測類別與嚴重度對照
│   ├── recommendation_principles.md # 建議產出硬性規則
│   └── output_format.md             # 報告結構範本
├── SKILL.md
├── LICENSE
├── README.md
└── doc/
    ├── README.zh.md
    ├── doc.md
    ├── doc.zh.md
    ├── architecture.md
    └── architecture.zh.md
```

安裝完成後，於 Claude Code 中呼叫 `/code-reviewer` 即可使用。

## 設定

此 skill 無需任何設定檔或環境變數；所有行為由指令參數與 `scripts/` 下的參考文件控制，無需初始化步驟。

## 使用方式

### 基本用法

```bash
/code-reviewer
```

分析當前工作目錄，報告寫入 `.doc/code-reviewer/{yyyy-MM-dd_HH-mm}.md`（24 小時制本地時間戳，例：`2026-04-25_14-30.md`）。

### 指定專案路徑

```bash
/code-reviewer ./my-project
```

輸出至 `my-project/.doc/code-reviewer/{yyyy-MM-dd_HH-mm}.md`。

### 指定輸出檔案（顯式覆寫）

```bash
/code-reviewer . custom.md
```

直接寫入 `./custom.md`，略過預設的 `.doc/code-reviewer/` 路徑規則。顯式指定輸出檔案時視為強制產檔請求，即使命中 No-Op 條件仍會寫入（內容為「未觀察到需處理事項」的最小報告）。

### 手動執行分析腳本

```bash
python3 ~/.claude/skills/code-reviewer/scripts/analyze_code.py /path/to/project
```

輸出 JSON，包含 `language`、`name`、`file_count`、`function_count`、`files`、`functions`、`issues`、`issue_counts`、`metrics`、`dependencies`，可用於除錯或串接其他工具。

## 命令列參考

### 指令參數

| 參數 | 預設 | 說明 |
|-----------|---------|-------------|
| `PROJECT_PATH` | 當前目錄 | 專案根目錄路徑 |
| `OUTPUT_FILE` | `.doc/code-reviewer/{yyyy-MM-dd_HH-mm}.md` | 輸出檔案路徑（相對於 `PROJECT_PATH`） |

兩個參數皆為選填。

### 輸出路徑規則

| 情境 | 行為 |
|------|------|
| 未指定 `OUTPUT_FILE` | 寫入 `{PROJECT_PATH}/.doc/code-reviewer/{yyyy-MM-dd_HH-mm}.md`，目錄不存在時自動建立 |
| 顯式指定 `OUTPUT_FILE` | 直接使用該路徑；含目錄需自行確保存在 |
| 任何情況 | 永遠不在專案根目錄落檔，所有產出集中於 `.doc/code-reviewer/` |

### No-Op 條件

同時滿足下列三項時，skip 建立目錄與寫檔，僅輸出一行「無需處理」訊息：

| 項目 | 條件 |
|------|------|
| 問題計數 | `issue_counts` 的 critical / high / medium / low 皆為 0 |
| 建議產出 | 套用 Recommendation Principles 後，架構／效能／安全三段皆無有效建議 |
| Metric 超標 | 未觀察到超標 metric（見 `scripts/recommendation_principles.md` 例外欄位定義） |

顯式指定 `OUTPUT_FILE` 時視為強制產檔請求，即使命中 No-Op 條件仍會寫入最小報告。

### 支援語言與分析器

| 語言 | 分析器 | 相依 |
|----------|----------|--------------|
| Go | `go/ast`（透過 `go run go_ast.go` helper）+ 字串掃描 | `go` ≥ 1.21 |
| Python | 內建 `ast` 模組 | Python ≥ 3.10 |
| JavaScript / TypeScript | 內建 brace-based 結構掃描（函式邊界／巢狀深度）+ 專案本地 `eslint`（可選）+ 字串掃描 | `node_modules/.bin/eslint`（可選） |

對應工具鏈不可用時，自動降級為字串掃描並在報告中標示。

### 偵測類別

| 類別 | 偵測方式 | 嚴重度 |
|------|-----------|----------|
| 過長函式 | 函式 > 50 行 | Medium |
| 過深巢狀 | 巢狀深度 > 3 層 | Medium |
| 未使用 import | AST 名稱引用分析 | Low |
| 大量連續註解 | ≥ 10 行連續單行註解 | Low |
| Go: `interface{}` | AST 偵測空介面 | Low |
| Go: 丟棄回傳值 | `_ = f()` 模式 | Medium |
| Python: bare except | `except:` 無類型 | Medium |
| JS/TS: eslint 規則 | 呼叫專案 eslint | High / Medium |
| 硬編碼密鑰（關鍵字） | `password=`／`secret=`／`api_key=` 等 | Critical |
| 可疑高熵字串 | Shannon entropy ≥ 4.0，長度 ≥ 32，排除 UUID／MD5／SHA1／SHA256／MIME type | High |
| SQL Injection | 字串拼接／f-string／% 格式化 SQL | High |
| Command Injection | 拼接系統指令 | High |

完整對照見 [`scripts/analysis_categories.md`](../scripts/analysis_categories.md)。

### 建議產出禁止事項

報告中的「架構建議／效能優化建議／安全性強化建議」段落套用下列硬性規則，違反者直接移除該建議：

| 反模式 | 說明 |
|---|---|
| 包裝既有抽象 | 既有 `dataclass`／`NamedTuple`／`TypedDict`／`Enum` 已是 factory 或常數集合，禁止再建議 helper 包裝 |
| 補文件為目的 | 禁止規模性建議「為所有函式補 docstring」 |
| 預測性優化 | 禁止「未來規模增大時」「若支援更多語言時」這類建議 |
| 裝飾性重構 | 禁止無具體指標佐證的「拆成更多小函式」建議 |
| 測試基礎建設擴張 | 禁止泛泛建議「補測試」 |
| 嚴重度膨脹 | Heuristic 偵測一律標註「需人工確認」，不得升級為 Critical |

完整規則與自我檢查清單見 [`scripts/recommendation_principles.md`](../scripts/recommendation_principles.md)。

### 報告結構

```markdown
# {project_name} 優化建議報告

## 摘要
## Critical Issues
## High Priority Issues
## Medium Priority Issues
## Low Priority Issues
## 架構建議
## 效能優化建議
## 安全性強化建議
## 待處理項目清單
```

完整範本見 [`scripts/output_format.md`](../scripts/output_format.md)。

### analyze_code.py 參數

| 參數 | 說明 |
|----------|-------------|
| `<project_path>` | 待分析專案根目錄的絕對或相對路徑 |

語言偵測順序：`go.mod` → `tsconfig.json` → `package.json` → `pyproject.toml`/`setup.py`/`requirements.txt`/`Pipfile`；皆不存在時依副檔名數量最多者判定。

***

©️ 2026
