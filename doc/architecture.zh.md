# code-reviewer - 架構

> 返回 [README](./README.zh.md)

## Overview

```mermaid
graph TB
    User[使用者] -->|/code-reviewer PROJECT_PATH OUTPUT_FILE| SKILL[SKILL.md<br/>Orchestration]
    SKILL --> Entry[analyze_code.py<br/>語言偵測 + 分派]
    Entry -->|go| Go[analyze_go.py]
    Entry -->|python| Py[analyze_python.py]
    Entry -->|js/ts| JS[analyze_js_ts.py]
    Go --> AST[go_ast.go<br/>go run 執行]
    Go --> Common[common.py<br/>共用偵測器]
    Py --> Common
    JS --> Common
    JS --> ESLint[專案本地 eslint<br/>可選]
    AST --> Result[ProjectAnalysis JSON]
    Common --> Result
    ESLint --> Result
    Result --> Gate[No-Op 閘門]
    Gate -->|命中| Notice[輸出「無需處理」訊息]
    Gate -->|未命中| Report[.doc/code-reviewer/<br/>優化建議報告]
```

## Module: SKILL.md（Orchestration）

定義 Detect → Analyze → Evaluate → Gate → Generate → Save 六階段流程與驗證清單，不含可執行程式碼，透過 prompt 指令約束 Claude 的行為。

```mermaid
graph TB
    subgraph SKILL["SKILL.md"]
        Detect[1. Detect<br/>偵測主要語言] --> Analyze[2. Analyze<br/>呼叫對應分析器]
        Analyze --> Evaluate[3. Evaluate<br/>計算指標並排序問題]
        Evaluate --> Gate[4. Gate<br/>檢查 No-Op 條件]
        Gate -->|命中| Skip[跳過 Generate/Save<br/>輸出無需處理訊息]
        Gate -->|未命中| Generate[5. Generate<br/>套用 Recommendation Principles]
        Generate --> Save[6. Save<br/>mkdir -p + 寫入報告]
    end
    SlashCmd[/code-reviewer/] --> SKILL
```

## Module: analyze_code.py（進入點 + 分派）

偵測專案主要語言後，分派給對應分析器；統一組裝 `issue_counts`、排序 `issues`，輸出單一 JSON。

```mermaid
graph TB
    subgraph Entry["analyze_code.py"]
        Main[main] --> Detect[detect_language<br/>go.mod / tsconfig.json /<br/>package.json / pyproject.toml]
        Detect --> Dispatch[_dispatch]
        Dispatch -->|go| GoAnalyzer[analyze_go.analyze]
        Dispatch -->|python| PyAnalyzer[analyze_python.analyze]
        Dispatch -->|js/ts| JSAnalyzer[analyze_js_ts.analyze]
        Dispatch -->|其他| Unsupported[不支援語言<br/>Issue: low]
        GoAnalyzer --> Build[_build_output<br/>排序 + 計數]
        PyAnalyzer --> Build
        JSAnalyzer --> Build
        Unsupported --> Build
        Build --> Stdout[stdout JSON]
    end
```

## Module: analyze_go.py + go_ast.go（Go 分析器）

`analyze_go.py` 負責 `gofmt` 前處理、`go.mod` 解析與字串模式掃描（憑證／SQL／指令注入／連續註解），並呼叫 `go_ast.go`（獨立 Go 程式，透過 `go run` 執行）取得函式簽章、未使用 import、`interface{}` 偵測、丟棄回傳值等需要真正 AST 的結果，再合併為單一 `ProjectAnalysis`。

```mermaid
graph TB
    subgraph GoPy["analyze_go.py"]
        Analyze[analyze] --> GoMod[_apply_go_mod<br/>解析 module/require]
        Analyze --> ScanSrc[_scan_sources<br/>逐檔 gofmt + 字串掃描]
        Analyze --> RunAST[_run_ast_helper]
        RunAST --> Merge[_merge_ast_output]
        Merge --> Metrics[_finalize_function_metrics]
    end
    subgraph GoAST["go_ast.go（go run）"]
        WalkFS[filepath.Walk *.go] --> AnalyzeFile[analyzeFile]
        AnalyzeFile --> UnusedImport[checkUnusedImport]
        AnalyzeFile --> EmptyInterface[interface{} 偵測]
        AnalyzeFile --> FuncInfo[analyzeFunction<br/>簽章 / 行數 / 巢狀深度]
        AnalyzeFile --> Discarded[checkDiscardedReturn<br/>_ = f() 模式]
        FuncInfo --> JSONOut[JSON stdout]
        UnusedImport --> JSONOut
        EmptyInterface --> JSONOut
        Discarded --> JSONOut
    end
    RunAST -->|go run go_ast.go root| WalkFS
    JSONOut -->|subprocess stdout| RunAST
    ScanSrc --> Common[common.py<br/>detect_hardcoded_credentials<br/>detect_sql_injection<br/>detect_command_injection<br/>detect_commented_code]
```

## Module: analyze_python.py（Python 分析器）

使用內建 `ast` 模組解析語法樹：`_NestingVisitor` 走訪 `If/For/While/Try/With` 節點計算巢狀深度，額外檢查未使用 import（比對 `ast.Name`／`ast.Attribute` 引用）與裸 `except:`。

```mermaid
graph TB
    subgraph PyAnalyzer["analyze_python.py"]
        Analyze[analyze] --> ParseFile[_analyze_file<br/>ast.parse]
        ParseFile --> UnusedImp[_check_unused_imports]
        ParseFile --> BareExcept[_check_bare_except]
        ParseFile --> WalkFn[ast.walk FunctionDef]
        WalkFn --> AnalyzeFn[_analyze_function]
        AnalyzeFn --> LengthCheck[_check_function_length<br/>> 50 行]
        AnalyzeFn --> NestVisitor[_NestingVisitor<br/>計算巢狀深度]
        NestVisitor --> NestCheck[_check_function_nesting<br/>> 3 層]
        ParseFile --> Common[common.py<br/>字串模式偵測]
    end
```

## Module: analyze_js_ts.py（JavaScript/TypeScript 分析器）

先以 `_strip_noise` 遮蔽註解／字串／模板字面值（保留行數），再以 brace-matching 找出函式邊界與巢狀深度；`_find_eslint` 偵測專案本地 eslint，存在時執行並映射訊息為 `Issue`。

```mermaid
graph TB
    subgraph JSAnalyzer["analyze_js_ts.py"]
        Analyze[analyze] --> Strip[_strip_noise<br/>遮蔽註解/字串/模板]
        Strip --> Extract[_extract_functions<br/>function / arrow / method pattern]
        Extract --> MatchBrace[_match_braces<br/>brace pairs]
        MatchBrace --> ScanNest[_scan_nesting<br/>_brace_kind 分類]
        Analyze --> FindEslint[_find_eslint<br/>node_modules/.bin/eslint]
        FindEslint -->|存在| RunEslint[_run_eslint --format json]
        RunEslint --> MapMsg[_map_eslint_messages]
        FindEslint -->|不存在| NoEslintIssue[Issue: eslint 不可用]
        Analyze --> Common[common.py<br/>字串模式偵測]
    end
```

## Module: common.py（共用型別與偵測器）

提供 `Issue`／`FunctionInfo`／`CodeMetrics`／`ProjectAnalysis` dataclass，以及三個語言共用的字串模式偵測函式，供 Go／Python／JS-TS 分析器直接呼叫。

```mermaid
classDiagram
    class ProjectAnalysis {
        +str language
        +str name
        +list~str~ files
        +list~FunctionInfo~ functions
        +list~Issue~ issues
        +CodeMetrics metrics
        +list~str~ dependencies
    }
    class Issue {
        +str severity
        +str category
        +str title
        +str description
        +str file
        +int line
        +str code_snippet
        +str suggestion
    }
    class FunctionInfo {
        +str name
        +str signature
        +str file
        +int line
        +int line_count
        +bool has_doc
    }
    class CodeMetrics {
        +int total_lines
        +int code_lines
        +float avg_function_length
        +int max_function_length
        +int max_nesting_depth
    }
    ProjectAnalysis --> FunctionInfo
    ProjectAnalysis --> Issue
    ProjectAnalysis --> CodeMetrics
```

**共用偵測函式**：`detect_hardcoded_credentials`（關鍵字 + Shannon entropy）、`detect_sql_injection`、`detect_command_injection`、`detect_commented_code`。

## Data Flow

完整一次 `/code-reviewer` 呼叫的資料流：

```mermaid
sequenceDiagram
    participant User
    participant Claude as Claude Code
    participant Skill as SKILL.md
    participant Entry as analyze_code.py
    participant Lang as 語言分析器
    participant FS as Filesystem

    User->>Claude: /code-reviewer [PROJECT_PATH] [OUTPUT_FILE]
    Claude->>Skill: 載入 skill 定義
    Skill->>Entry: analyze_code.py <project_path>
    Entry->>Entry: detect_language
    Entry->>Lang: _dispatch(lang, root)
    alt Go
        Lang->>FS: gofmt -s -w *.go
        Lang->>Lang: go run go_ast.go <root>
    else Python
        Lang->>Lang: ast.parse 逐檔分析
    else JS/TS
        Lang->>FS: 偵測 node_modules/.bin/eslint
        opt eslint 存在
            Lang->>Lang: eslint . --format json
        end
    end
    Lang-->>Entry: ProjectAnalysis
    Entry-->>Skill: stdout JSON（issues / issue_counts / metrics）
    Skill->>Skill: 套用 recommendation_principles.md 過濾建議
    Skill->>Skill: 檢查 No-Op 條件
    alt No-Op 命中且未指定 OUTPUT_FILE
        Skill-->>User: 「無需處理」單行訊息
    else 需要產檔
        Skill->>FS: mkdir -p .doc/code-reviewer/
        Skill->>FS: 寫入 {yyyy-MM-dd_HH-mm}.md
        Skill-->>User: 報告路徑
    end
```

## No-Op 閘門狀態機

```mermaid
stateDiagram-v2
    [*] --> CheckIssues: 分析完成
    CheckIssues --> CheckMetrics: issue_counts 全為 0
    CheckIssues --> WriteReport: issue_counts 有非 0 項
    CheckMetrics --> CheckSuggestions: 無超標 metric
    CheckMetrics --> WriteReport: 有超標 metric
    CheckSuggestions --> NoOp: 架構/效能/安全三段皆無有效建議
    CheckSuggestions --> WriteReport: 至少一段有有效建議
    NoOp --> ForcedWrite: 使用者顯式指定 OUTPUT_FILE
    NoOp --> SkipWrite: 未指定 OUTPUT_FILE
    ForcedWrite --> [*]: 寫入最小報告
    SkipWrite --> [*]: 僅輸出無需處理訊息
    WriteReport --> [*]: 寫入完整報告
```

***

©️ 2026
