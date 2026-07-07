# code-reviewer - Documentation

> Back to [README](../README.md)

## Prerequisites

- [Claude Code](https://claude.ai/claude-code) CLI installed and configured
- Python 3.10 or higher (uses the built-in `ast` module and modern generic syntax)
- Go 1.21 or higher (optional, only needed for AST analysis of Go projects; falls back to string scanning otherwise)
- Project-local `eslint` (optional, only needed for JS/TS lint-rule integration)

## Installation

### Clone from GitHub

```bash
git clone https://github.com/pardnchiu/skill-code-reviewer.git \
    ~/.claude/skills/code-reviewer
```

### Manual Installation

Place the following files under `~/.claude/skills/code-reviewer/`:

```
code-reviewer/
├── scripts/
│   ├── analyze_code.py              # Entry point: language detection + dispatch
│   ├── analyze_go.py                # Go analyzer
│   ├── analyze_python.py            # Python analyzer
│   ├── analyze_js_ts.py             # JavaScript/TypeScript analyzer
│   ├── common.py                    # Shared types + detection utilities
│   ├── go_ast.go                    # Go AST helper (invoked via go run)
│   ├── analysis_categories.md       # Detection category / severity table
│   ├── recommendation_principles.md # Hard rules for recommendation output
│   └── output_format.md             # Report structure template
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

Once installed, invoke `/code-reviewer` from Claude Code.

## Configuration

This skill requires no config file or environment variables — all behavior is driven by command arguments and the reference documents under `scripts/`, with no initialization step.

## Usage

### Basic

```bash
/code-reviewer
```

Analyzes the current working directory and writes the report to `.doc/code-reviewer/{yyyy-MM-dd_HH-mm}.md` (24-hour local timestamp, e.g. `2026-04-25_14-30.md`).

### Specify Project Path

```bash
/code-reviewer ./my-project
```

Writes to `my-project/.doc/code-reviewer/{yyyy-MM-dd_HH-mm}.md`.

### Specify Output File (Explicit Override)

```bash
/code-reviewer . custom.md
```

Writes directly to `./custom.md`, bypassing the default `.doc/code-reviewer/` path rule. An explicit output file is treated as a forced-write request — the report is written even when the No-Op condition is met (as a minimal "nothing to address" report).

### Run the Analysis Script Manually

```bash
python3 ~/.claude/skills/code-reviewer/scripts/analyze_code.py /path/to/project
```

Outputs JSON containing `language`, `name`, `file_count`, `function_count`, `files`, `functions`, `issues`, `issue_counts`, `metrics`, and `dependencies` — useful for debugging or piping into other tooling.

## CLI Reference

### Command Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `PROJECT_PATH` | Current directory | Project root path |
| `OUTPUT_FILE` | `.doc/code-reviewer/{yyyy-MM-dd_HH-mm}.md` | Output file path (relative to `PROJECT_PATH`) |

Both parameters are optional.

### Output Path Rules

| Scenario | Behavior |
|------|------|
| `OUTPUT_FILE` not specified | Writes to `{PROJECT_PATH}/.doc/code-reviewer/{yyyy-MM-dd_HH-mm}.md`, auto-creating the directory if missing |
| `OUTPUT_FILE` explicitly specified | Uses that path directly; caller must ensure any containing directory exists |
| Always | Never writes to the project root — all output is confined to `.doc/code-reviewer/` |

### No-Op Condition

When all three of the following hold, the skill skips directory creation and file writes, emitting a single "nothing to do" line instead:

| Item | Condition |
|------|------|
| Issue counts | `issue_counts` critical / high / medium / low are all 0 |
| Recommendation output | After applying the Recommendation Principles, the architecture / performance / security sections all have no actionable suggestions |
| Metric thresholds | No metric exceeds its threshold (see the exception fields in `scripts/recommendation_principles.md`) |

An explicitly specified `OUTPUT_FILE` is treated as a forced-write request — a minimal report is written even when the No-Op condition is met.

### Supported Languages and Analyzers

| Language | Analyzer | Dependencies |
|----------|----------|--------------|
| Go | `go/ast` (via a `go run go_ast.go` helper) + string scanning | `go` ≥ 1.21 |
| Python | Built-in `ast` module | Python ≥ 3.10 |
| JavaScript / TypeScript | Built-in brace-based structural scan (function boundaries / nesting depth) + project-local `eslint` (optional) + string scanning | `node_modules/.bin/eslint` (optional) |

When the corresponding toolchain is unavailable, analysis automatically degrades to string scanning and is flagged in the report.

### Detection Categories

| Category | Detection | Severity |
|------|-----------|----------|
| Long function | Function > 50 lines | Medium |
| Deep nesting | Nesting depth > 3 levels | Medium |
| Unused import | AST name-reference analysis | Low |
| Large comment block | ≥ 10 consecutive single-line comments | Low |
| Go: `interface{}` | AST detection of empty interfaces | Low |
| Go: discarded return value | `_ = f()` pattern | Medium |
| Python: bare except | `except:` with no type | Medium |
| JS/TS: eslint rules | Invokes the project's eslint | High / Medium |
| Hardcoded credential (keyword) | `password=`/`secret=`/`api_key=`, etc. | Critical |
| Suspicious high-entropy string | Shannon entropy ≥ 4.0, length ≥ 32, excludes UUID/MD5/SHA1/SHA256/MIME type | High |
| SQL Injection | String concatenation / f-string / % formatting into SQL | High |
| Command Injection | Concatenating system commands | High |

Full table: [`scripts/analysis_categories.md`](../scripts/analysis_categories.md).

### Recommendation Output Restrictions

The report's "architecture / performance / security" recommendation sections apply the following hard rules — any violating suggestion is removed rather than kept:

| Anti-pattern | Description |
|---|---|
| Wrapping an existing abstraction | An existing `dataclass`/`NamedTuple`/`TypedDict`/`Enum` that is already a factory or constant set must not get a wrapper helper suggestion |
| Documentation for its own sake | No blanket "add docstrings to all functions" suggestions |
| Speculative optimization | No "when the project scales up" / "if more languages are supported" suggestions |
| Decorative refactor | No "split into more small functions" suggestion without a concrete supporting metric |
| Test-infrastructure expansion | No generic "add more tests" suggestion |
| Severity inflation | Heuristic detections (pattern/entropy) are always flagged "needs manual confirmation," never escalated to Critical |

Full rules and self-check list: [`scripts/recommendation_principles.md`](../scripts/recommendation_principles.md).

### Report Structure

```markdown
# {project_name} Optimization Report

## Summary
## Critical Issues
## High Priority Issues
## Medium Priority Issues
## Low Priority Issues
## Architecture Recommendations
## Performance Recommendations
## Security Recommendations
## Task Checklist
```

Full template: [`scripts/output_format.md`](../scripts/output_format.md).

### analyze_code.py Arguments

| Argument | Description |
|----------|-------------|
| `<project_path>` | Absolute or relative path to the project root to analyze |

Language detection order: `go.mod` → `tsconfig.json` → `package.json` → `pyproject.toml`/`setup.py`/`requirements.txt`/`Pipfile`; when none match, the extension with the most files wins.

***

©️ 2026
