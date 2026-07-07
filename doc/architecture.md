# code-reviewer - Architecture

> Back to [README](../README.md)

## Overview

```mermaid
graph TB
    User[User] -->|/code-reviewer PROJECT_PATH OUTPUT_FILE| SKILL[SKILL.md<br/>Orchestration]
    SKILL --> Entry[analyze_code.py<br/>Language Detect + Dispatch]
    Entry -->|go| Go[analyze_go.py]
    Entry -->|python| Py[analyze_python.py]
    Entry -->|js/ts| JS[analyze_js_ts.py]
    Go --> AST[go_ast.go<br/>run via go run]
    Go --> Common[common.py<br/>Shared Detectors]
    Py --> Common
    JS --> Common
    JS --> ESLint[Project-local eslint<br/>optional]
    AST --> Result[ProjectAnalysis JSON]
    Common --> Result
    ESLint --> Result
    Result --> Gate[No-Op Gate]
    Gate -->|hit| Notice[Emit "nothing to do" message]
    Gate -->|miss| Report[.doc/code-reviewer/<br/>Optimization Report]
```

## Module: SKILL.md (Orchestration)

Defines the six-stage Detect → Analyze → Evaluate → Gate → Generate → Save workflow and validation checklist. Contains no executable code — it constrains Claude's behavior through prompt instructions.

```mermaid
graph TB
    subgraph SKILL["SKILL.md"]
        Detect[1. Detect<br/>Detect primary language] --> Analyze[2. Analyze<br/>Invoke matching analyzer]
        Analyze --> Evaluate[3. Evaluate<br/>Compute metrics, sort issues]
        Evaluate --> Gate[4. Gate<br/>Check No-Op condition]
        Gate -->|hit| Skip[Skip Generate/Save<br/>Emit nothing-to-do message]
        Gate -->|miss| Generate[5. Generate<br/>Apply Recommendation Principles]
        Generate --> Save[6. Save<br/>mkdir -p + write report]
    end
    SlashCmd[/code-reviewer/] --> SKILL
```

## Module: analyze_code.py (Entry Point + Dispatch)

After detecting the project's primary language, dispatches to the corresponding analyzer; assembles `issue_counts`, sorts `issues`, and emits a single JSON blob.

```mermaid
graph TB
    subgraph Entry["analyze_code.py"]
        Main[main] --> Detect[detect_language<br/>go.mod / tsconfig.json /<br/>package.json / pyproject.toml]
        Detect --> Dispatch[_dispatch]
        Dispatch -->|go| GoAnalyzer[analyze_go.analyze]
        Dispatch -->|python| PyAnalyzer[analyze_python.analyze]
        Dispatch -->|js/ts| JSAnalyzer[analyze_js_ts.analyze]
        Dispatch -->|other| Unsupported[Unsupported language<br/>Issue: low]
        GoAnalyzer --> Build[_build_output<br/>sort + count]
        PyAnalyzer --> Build
        JSAnalyzer --> Build
        Unsupported --> Build
        Build --> Stdout[stdout JSON]
    end
```

## Module: analyze_go.py + go_ast.go (Go Analyzer)

`analyze_go.py` handles `gofmt` preprocessing, `go.mod` parsing, and string-pattern scans (credentials / SQL / command injection / comment blocks), then invokes `go_ast.go` (a standalone Go program run via `go run`) for the results that need real AST — function signatures, unused imports, `interface{}` detection, discarded return values — and merges them into a single `ProjectAnalysis`.

```mermaid
graph TB
    subgraph GoPy["analyze_go.py"]
        Analyze[analyze] --> GoMod[_apply_go_mod<br/>parse module/require]
        Analyze --> ScanSrc[_scan_sources<br/>per-file gofmt + string scan]
        Analyze --> RunAST[_run_ast_helper]
        RunAST --> Merge[_merge_ast_output]
        Merge --> Metrics[_finalize_function_metrics]
    end
    subgraph GoAST["go_ast.go (go run)"]
        WalkFS[filepath.Walk *.go] --> AnalyzeFile[analyzeFile]
        AnalyzeFile --> UnusedImport[checkUnusedImport]
        AnalyzeFile --> EmptyInterface[interface{} detection]
        AnalyzeFile --> FuncInfo[analyzeFunction<br/>signature / lines / nesting]
        AnalyzeFile --> Discarded[checkDiscardedReturn<br/>_ = f() pattern]
        FuncInfo --> JSONOut[JSON stdout]
        UnusedImport --> JSONOut
        EmptyInterface --> JSONOut
        Discarded --> JSONOut
    end
    RunAST -->|go run go_ast.go root| WalkFS
    JSONOut -->|subprocess stdout| RunAST
    ScanSrc --> Common[common.py<br/>detect_hardcoded_credentials<br/>detect_sql_injection<br/>detect_command_injection<br/>detect_commented_code]
```

## Module: analyze_python.py (Python Analyzer)

Parses the syntax tree with the built-in `ast` module: `_NestingVisitor` walks `If/For/While/Try/With` nodes to compute nesting depth, alongside unused-import checks (matching `ast.Name`/`ast.Attribute` references) and bare `except:` detection.

```mermaid
graph TB
    subgraph PyAnalyzer["analyze_python.py"]
        Analyze[analyze] --> ParseFile[_analyze_file<br/>ast.parse]
        ParseFile --> UnusedImp[_check_unused_imports]
        ParseFile --> BareExcept[_check_bare_except]
        ParseFile --> WalkFn[ast.walk FunctionDef]
        WalkFn --> AnalyzeFn[_analyze_function]
        AnalyzeFn --> LengthCheck[_check_function_length<br/>> 50 lines]
        AnalyzeFn --> NestVisitor[_NestingVisitor<br/>compute nesting depth]
        NestVisitor --> NestCheck[_check_function_nesting<br/>> 3 levels]
        ParseFile --> Common[common.py<br/>string-pattern detectors]
    end
```

## Module: analyze_js_ts.py (JavaScript/TypeScript Analyzer)

First blanks out comments/strings/template literals with `_strip_noise` (preserving line numbers), then uses brace-matching to locate function boundaries and nesting depth; `_find_eslint` detects a project-local eslint and, if present, runs it and maps messages into `Issue` objects.

```mermaid
graph TB
    subgraph JSAnalyzer["analyze_js_ts.py"]
        Analyze[analyze] --> Strip[_strip_noise<br/>blank comments/strings/templates]
        Strip --> Extract[_extract_functions<br/>function / arrow / method pattern]
        Extract --> MatchBrace[_match_braces<br/>brace pairs]
        MatchBrace --> ScanNest[_scan_nesting<br/>_brace_kind classification]
        Analyze --> FindEslint[_find_eslint<br/>node_modules/.bin/eslint]
        FindEslint -->|found| RunEslint[_run_eslint --format json]
        RunEslint --> MapMsg[_map_eslint_messages]
        FindEslint -->|not found| NoEslintIssue[Issue: eslint unavailable]
        Analyze --> Common[common.py<br/>string-pattern detectors]
    end
```

## Module: common.py (Shared Types + Detectors)

Provides the `Issue`/`FunctionInfo`/`CodeMetrics`/`ProjectAnalysis` dataclasses plus three string-pattern detection functions shared across the Go/Python/JS-TS analyzers.

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

**Shared detectors**: `detect_hardcoded_credentials` (keyword + Shannon entropy), `detect_sql_injection`, `detect_command_injection`, `detect_commented_code`.

## Data Flow

Complete flow of a single `/code-reviewer` invocation:

```mermaid
sequenceDiagram
    participant User
    participant Claude as Claude Code
    participant Skill as SKILL.md
    participant Entry as analyze_code.py
    participant Lang as Language Analyzer
    participant FS as Filesystem

    User->>Claude: /code-reviewer [PROJECT_PATH] [OUTPUT_FILE]
    Claude->>Skill: Load skill definition
    Skill->>Entry: analyze_code.py <project_path>
    Entry->>Entry: detect_language
    Entry->>Lang: _dispatch(lang, root)
    alt Go
        Lang->>FS: gofmt -s -w *.go
        Lang->>Lang: go run go_ast.go <root>
    else Python
        Lang->>Lang: ast.parse per file
    else JS/TS
        Lang->>FS: detect node_modules/.bin/eslint
        opt eslint present
            Lang->>Lang: eslint . --format json
        end
    end
    Lang-->>Entry: ProjectAnalysis
    Entry-->>Skill: stdout JSON (issues / issue_counts / metrics)
    Skill->>Skill: Filter recommendations via recommendation_principles.md
    Skill->>Skill: Check No-Op condition
    alt No-Op hit and OUTPUT_FILE not specified
        Skill-->>User: Single "nothing to do" line
    else Report needed
        Skill->>FS: mkdir -p .doc/code-reviewer/
        Skill->>FS: Write {yyyy-MM-dd_HH-mm}.md
        Skill-->>User: Report path
    end
```

## No-Op Gate State Machine

```mermaid
stateDiagram-v2
    [*] --> CheckIssues: Analysis complete
    CheckIssues --> CheckMetrics: issue_counts all 0
    CheckIssues --> WriteReport: any issue_counts nonzero
    CheckMetrics --> CheckSuggestions: no metric exceeds threshold
    CheckMetrics --> WriteReport: a metric exceeds threshold
    CheckSuggestions --> NoOp: architecture/perf/security sections all have no actionable suggestion
    CheckSuggestions --> WriteReport: at least one section has an actionable suggestion
    NoOp --> ForcedWrite: user explicitly specified OUTPUT_FILE
    NoOp --> SkipWrite: OUTPUT_FILE not specified
    ForcedWrite --> [*]: write minimal report
    SkipWrite --> [*]: emit nothing-to-do message only
    WriteReport --> [*]: write full report
```

***

©️ 2026
