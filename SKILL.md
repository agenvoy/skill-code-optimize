---
name: code-optimize
description: Analyze project source code and generate optimization suggestions. Use when user wants code review, performance optimization advice, security hardening recommendations, or architecture improvement suggestions.
---

# Code Optimization Analyzer

分析專案原始碼並產生優化建議報告。

## Command Syntax

```
/code-optimize [PROJECT_PATH] [OUTPUT_FILE]
```

### Parameters (All Optional)

| Parameter | Default | Description |
|-----------|---------|-------------|
| `PROJECT_PATH` | Current directory | 專案根目錄路徑 |
| `OUTPUT_FILE` | `suggest.md` | 輸出檔案名稱 |

### Examples

```bash
/code-optimize                           # 分析當前目錄，輸出到 suggest.md
/code-optimize ./my-project              # 分析指定專案
/code-optimize . optimization.md         # 自訂輸出檔名
```

---

## Workflow

```
1. Analyze    →  執行 analyze_code.py 取得專案結構與程式碼資訊
2. Scan       →  掃描各類問題（重複、硬編碼、安全性等）
3. Evaluate   →  評估架構設計與效能瓶頸
4. Generate   →  產生優化建議報告（繁體中文）
5. Save       →  寫入 OUTPUT_FILE
```

---

## Step 1: Analyze Project

```bash
python3 ~/.claude/skills/code-optimize/scripts/analyze_code.py /path/to/project
```

Output: JSON 包含：
- `language`: 主要語言
- `files`: 檔案列表
- `functions`: 函式資訊
- `types`: 型別定義
- `issues`: 偵測到的問題
- `metrics`: 程式碼指標

---

## Analysis Categories

### 1. Code Quality Issues

| Issue Type | Detection | Severity |
|------------|-----------|----------|
| 重複程式碼 | 相似函式/邏輯區塊 | Medium |
| 硬編碼值 | 路徑、URL、密碼、Port | High |
| 未使用 Import | 引入但未使用的套件 | Low |
| 過長函式 | 超過 50 行的函式 | Medium |
| 過深巢狀 | 超過 3 層的巢狀結構 | Medium |
| 註解掉的程式碼 | 大量被註解的 code | Low |

### 2. Performance Issues

| Issue Type | Detection | Severity |
|------------|-----------|----------|
| N+1 Query | 迴圈內資料庫查詢 | High |
| 無效迴圈 | 可用 map/filter 取代 | Low |
| 記憶體洩漏風險 | 未關閉的資源 | High |
| 同步阻塞 | 可改為非同步的操作 | Medium |

### 3. Security Issues

| Issue Type | Detection | Severity |
|------------|-----------|----------|
| SQL Injection | 字串拼接 SQL | Critical |
| Command Injection | 未驗證的 shell 指令 | Critical |
| 硬編碼密鑰 | API Key、Password | Critical |
| 不安全的隨機數 | 使用 Math.random 等 | Medium |
| 缺少輸入驗證 | 未驗證外部輸入 | High |

### 4. Architecture Issues

| Issue Type | Detection | Severity |
|------------|-----------|----------|
| 循環依賴 | Package 互相引用 | High |
| God Object | 過大的 struct/class | Medium |
| 缺少錯誤處理 | 忽略的 error return | High |
| 緊耦合 | 直接依賴具體實作 | Medium |

---

## Output Format

### Report Structure (suggest.md)

```markdown
# {project_name} 優化建議報告

## 摘要

- 語言：{language}
- 檔案數：{file_count}
- 函式數：{function_count}
- 問題總數：{issue_count}（Critical: X, High: X, Medium: X, Low: X）

---

## Critical Issues

### 1. {issue_title}

**檔案**：`{file_path}:{line_number}`

**問題**：{description}

**目前程式碼**：
```{lang}
{current_code}
```

**建議修改**：
```{lang}
{suggested_code}
```

**原因**：{reason}

---

## High Priority Issues

...

## Medium Priority Issues

...

## Low Priority Issues

...

---

## 架構建議

### {suggestion_title}

**現況**：{current_state}

**建議**：{recommendation}

**優點**：
- {benefit_1}
- {benefit_2}

---

## 效能優化建議

...

---

## 安全性強化建議

...

---

## 待處理項目清單

- [ ] {task_1}
- [ ] {task_2}
- [ ] {task_3}
```

---

## Severity Levels

| Level | Description | Action |
|-------|-------------|--------|
| Critical | 安全漏洞或資料風險 | 立即修復 |
| High | 影響穩定性或效能 | 優先處理 |
| Medium | 影響維護性或可讀性 | 計劃修復 |
| Low | 風格或最佳實踐 | 有空處理 |

---

## Language-Specific Rules

### Go

- 檢查 `context.Context` 傳遞
- 檢查 `defer` 資源釋放
- 檢查 error wrapping (`%w`)
- 檢查 exported function 文件
- 檢查 race condition 風險

### Python

- 檢查 type hints
- 檢查 exception handling
- 檢查 resource cleanup (with statement)
- 檢查 f-string vs format

### JavaScript/TypeScript

- 檢查 async/await 使用
- 檢查 null/undefined 處理
- 檢查 memory leak (event listeners)
- 檢查 TypeScript strict mode

---

## Output Guidelines

1. **繁體中文** — 報告使用繁體中文（ZH-TW）
2. **技術術語** — 保留英文（如 Race Condition、Memory Leak）
3. **具體建議** — 提供可執行的修改建議，非泛泛之談
4. **優先排序** — 按嚴重程度排序問題
5. **程式碼範例** — 提供修正前後的對比

---

## Validation Checklist

- [ ] 專案成功分析
- [ ] 所有 Critical issues 列出
- [ ] 每個問題包含檔案位置
- [ ] 每個問題包含修改建議
- [ ] 報告已儲存至指定路徑
