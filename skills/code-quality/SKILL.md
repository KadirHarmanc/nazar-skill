---
description: "Analyze code quality - complexity, dead code, error handling, naming conventions. Use when user says 'code quality', 'kod kalitesi', 'complexity check', 'clean code', 'code smell', 'kalite analizi'."
---

# Nazar Code Quality Analysis Skill

You are performing a comprehensive code quality analysis. Follow these instructions systematically.

## Step 1: Detect Project Stack

Use Glob to identify the project's technology stack:

```
# Check for project files
package.json        -> Node.js / JavaScript / TypeScript
requirements.txt    -> Python
Gemfile            -> Ruby
pom.xml            -> Java
go.mod             -> Go
Cargo.toml         -> Rust
composer.json      -> PHP
pubspec.yaml       -> Dart / Flutter
*.csproj           -> .NET / C#
```

Note the primary language and framework for language-specific quality checks.

## Step 2: Systematic Quality Scan

Scan the project for the following quality issue categories using Grep, Glob, and Read tools.

### Category 1: Dead Code

**Grep Patterns:**
```
# Unused imports (JavaScript/TypeScript)
pattern: "^import\s+.*\s+from\s+"
context: "Check if imported symbol is used elsewhere in the file"

# Unused imports (Python)
pattern: "^(from\s+\S+\s+)?import\s+"
context: "Check if imported module/symbol is used"

# Commented-out code blocks
pattern: "^\s*(//|#|/\*)\s*(function|class|def|const|let|var|if|for|while|return|import)"
context: "Commented-out code - should be removed"

# Unreachable code after return/throw
pattern: "(return|throw|break|continue)\s*[;\n].*\S"
context: "Code after return/throw/break/continue may be unreachable"

# Unused variables
pattern: "(const|let|var|int|string|float)\s+(\w+)\s*="
context: "Check if variable is used after declaration"

# Empty functions/methods
pattern: "(function|def|fn)\s+\w+\s*\([^)]*\)\s*\{?\s*\}?"
context: "Check if function body is empty"

# TODO/FIXME that indicate dead code
pattern: "(TODO|FIXME|HACK|XXX|DEPRECATED).*?(remove|delete|clean|unused)"
context: "Developer note about dead code"
```

**Analysis:** For each import found, use Grep to check if the imported symbol is used in the same file. Report truly unused imports.

### Category 2: Duplicate Code

**Grep Patterns:**
```
# Similar function signatures
pattern: "(function|def|fn|func)\s+\w+\s*\([^)]*\)"
context: "Compare function signatures across files for duplicates"

# Repeated error handling blocks
pattern: "catch\s*\([^)]*\)\s*\{[^}]*\}"
context: "Compare catch blocks for duplication"

# Repeated validation logic
pattern: "if\s*\(\s*!\s*\w+\s*\)\s*\{?\s*(throw|return|res\.status)"
context: "Repeated validation pattern"

# Repeated API call patterns
pattern: "(fetch|axios|request)\s*\([^)]*\)"
context: "Repeated API call patterns - consider abstraction"

# Config/constant duplication
pattern: "(http://|https://|localhost:\d+|127\.0\.0\.1:\d+)"
context: "Hardcoded URLs that should be in config"
```

**Analysis:** Compare similar patterns across files. Report blocks that appear 3+ times and could be extracted into shared utilities.

### Category 3: Complexity Issues

**Grep Patterns:**
```
# Deep nesting (4+ levels)
pattern: "^\s{16,}(if|for|while|switch|try|else)"
context: "Deep nesting (4+ levels) - extract to functions"

# Long functions (check line count between function boundaries)
pattern: "(function|def|fn|func)\s+\w+"
context: "Check function length - flag if 100+ lines"

# Complex conditionals
pattern: "if\s*\(.*?&&.*?&&.*?\)|if\s*\(.*?\|\|.*?\|\|.*?\)"
context: "Complex conditional with 3+ operators"

# Switch with many cases
pattern: "case\s+['\"]?\w+['\"]?\s*:"
context: "Count cases in switch - flag if 10+"

# Callback hell / deep Promise chains
pattern: "\.then\s*\([^)]*\)\s*\.then\s*\([^)]*\)\s*\.then"
context: "Deep Promise chain - use async/await"

# God objects (files with many methods)
pattern: "(export\s+(function|const|class)|def\s+\w+|public\s+(void|static|async))"
context: "Count exports/methods per file - flag if 20+"

# Cyclomatic complexity indicators
pattern: "(if|else if|elif|case|catch|for|while|&&|\|\||\\?)\s"
context: "Count per function for cyclomatic complexity"
```

**Analysis:**
- Use Glob to find all source files; note files over 500 lines.
- For functions over 100 lines, report with suggestion to split.
- For nesting over 4 levels, report with early return/guard clause suggestion.

### Category 4: Error Handling Issues

**Grep Patterns:**
```
# Empty catch/except blocks
pattern: "catch\s*\([^)]*\)\s*\{\s*\}|except\s*(\s*\w+\s*)?:\s*pass"
context: "Empty catch/except - error swallowed"

# Generic catch-all
pattern: "catch\s*\(\s*(e|err|error|ex|exception)\s*\)\s*\{|except\s+Exception|except\s*:"
context: "Generic catch-all - should catch specific errors"

# Console.log in catch (not proper error handling)
pattern: "catch\s*\([^)]*\)\s*\{\s*console\.(log|error)\s*\("
context: "Console.log in catch - use proper error logging"

# Missing error handling on async
pattern: "(await\s+\w+)\s*(?!\s*\.catch|try)"
context: "Await without try/catch"

# Unhandled Promise
pattern: "new\s+Promise\s*\(|\.then\s*\([^)]*\)\s*(?!\.catch)"
context: "Promise without .catch() handler"

# Error thrown as string
pattern: "throw\s+['\"]|raise\s+['\"]"
context: "Throwing string instead of Error object"

# Process.exit in library code
pattern: "process\.exit|sys\.exit|os\.Exit"
context: "Process exit in non-entry-point code"
```

### Category 5: Async/Await Misuse

**Grep Patterns:**
```
# Missing await
pattern: "(async\s+function|async\s*\()[^}]*(fetch|axios|query|find|save|create|update|delete)\s*\([^)]*\)\s*(?!.*await)"
context: "Async function calling async operation without await"

# Unnecessary async
pattern: "async\s+(function\s+\w+|(\w+)\s*=)\s*[^}]*(?!await)[^}]*\}"
context: "Async function that never uses await"

# Sequential awaits that could be parallel
pattern: "await\s+\w+.*\n\s*await\s+\w+"
context: "Sequential awaits - could use Promise.all if independent"

# Async in forEach (does not await)
pattern: "\.(forEach|map)\s*\(\s*async"
context: "Async in forEach does not wait for completion - use for...of or Promise.all"

# Return await (unnecessary)
pattern: "return\s+await\s+"
context: "Return await is unnecessary in most cases"
```

### Category 6: Memory Leak Patterns

**Grep Patterns:**
```
# Event listeners not removed
pattern: "(addEventListener|on\(|addListener)\s*\([^)]*\)"
context: "Check if corresponding removeEventListener exists"

# Intervals not cleared
pattern: "(setInterval|setTimeout)\s*\([^)]*\)"
context: "Check if corresponding clearInterval/clearTimeout exists"

# Global state accumulation
pattern: "(global\.\w+\s*=|window\.\w+\s*=|globalThis\.\w+\s*=)"
context: "Global variable assignment - potential memory leak"

# Unsubscribed observables
pattern: "\.subscribe\s*\("
context: "Check if subscription is unsubscribed on cleanup"

# Missing cleanup in useEffect
pattern: "useEffect\s*\(\s*\(\s*\)\s*=>\s*\{"
context: "useEffect - check if cleanup function returns for listeners/intervals"

# Large arrays/maps growing
pattern: "(\.push\(|\.set\(|\.add\()\s*.*?(cache|store|buffer|queue|log)"
context: "Growing collection - check if bounded or cleaned"
```

### Category 7: Performance Anti-Patterns

**Grep Patterns:**
```
# N+1 query pattern
pattern: "(for|forEach|map)\s*\(.*\{[^}]*?(find|query|fetch|select|get)\s*\("
context: "Database query inside loop - N+1 problem"

# Unnecessary re-renders (React)
pattern: "(useState|useContext|useSelector)\s*\("
context: "Check if state updates cause unnecessary re-renders"

# Missing useMemo/useCallback
pattern: "(const\s+\w+\s*=\s*\([^)]*\)\s*=>|function\s+\w+)\s*.*?return\s*<"
context: "Inline function in render - consider useCallback"

# Large bundle imports
pattern: "import\s+.*\s+from\s+['\"]lodash['\"]|import\s+.*\s+from\s+['\"]moment['\"]"
context: "Full library import - use specific imports"

# Synchronous file operations
pattern: "(readFileSync|writeFileSync|readdirSync|existsSync|statSync)"
context: "Synchronous file I/O - blocks event loop"

# Missing pagination
pattern: "(findAll|find\(\s*\{\s*\}|SELECT\s+\*|\.list\(\s*\))"
context: "Unbounded query without pagination"

# Missing indexes
pattern: "(findOne|find)\s*\(\s*\{[^}]*?\b(where|query)\b"
context: "Database query - verify indexes exist for query fields"
```

### Category 8: Accessibility Issues

**Grep Patterns:**
```
# Missing alt text
pattern: "<img[^>]*(?!.*alt=)[^>]*/?\s*>"
context: "Image without alt attribute"

# Missing aria labels
pattern: "<(button|a|input|div|span)[^>]*(onClick|click|press)[^>]*(?!.*aria-)[^>]*>"
context: "Interactive element without ARIA label"

# Missing form labels
pattern: "<input[^>]*(?!.*aria-label|.*id=.*<label)[^>]*/?\s*>"
context: "Input without associated label"

# Color-only indicators
pattern: "(color|background).*?(red|green|#f00|#0f0|#ff0000|#00ff00)"
context: "Color-only indicator - needs alternative"

# Missing focus management
pattern: "(tabIndex|tabindex)\s*=\s*['\"]?-1"
context: "Negative tabIndex removes from tab order"

# Missing heading hierarchy
pattern: "<h[1-6]"
context: "Check heading hierarchy - should be sequential"

# Missing language attribute
pattern: "<html[^>]*(?!.*lang=)[^>]*>"
context: "HTML without lang attribute"
```

### Category 9: Hardcoded i18n Strings

**Grep Patterns:**
```
# Hardcoded user-facing strings
pattern: "(placeholder|title|label|aria-label|alt)\s*=\s*['\"][A-Z][a-z]+"
context: "Hardcoded string in attribute - should use i18n"

# Hardcoded JSX text
pattern: ">\s*[A-Z][a-zA-Z\s]{5,}\s*</"
context: "Hardcoded text in JSX - should use i18n"

# Error messages
pattern: "(message|msg|text|error)\s*[:=]\s*['\"][A-Z][a-zA-Z\s]{10,}['\"]"
context: "Hardcoded error/message string"

# i18n library usage
pattern: "(i18n|intl|t\(|formatMessage|useTranslation|gettext|ngettext|__|_\()"
context: "i18n library in use - check coverage"

# Missing i18n configuration
pattern: "(locale|language|i18n.*config|intl.*config)"
context: "i18n configuration"
```

### Category 10: Naming Convention Violations

**Grep Patterns:**
```
# Single-letter variables (except loops)
pattern: "(const|let|var|int|string)\s+[a-z]\s*[=;]"
context: "Single-letter variable name"

# Inconsistent casing
pattern: "(const|let|var)\s+[a-z]+_[a-z]+\s*=|def\s+[a-z]+[A-Z]"
context: "Inconsistent naming convention (mixing camelCase and snake_case)"

# Boolean naming
pattern: "(const|let|var)\s+(?!is|has|can|should|will|did)[a-z]+\s*=\s*(true|false)"
context: "Boolean without is/has/can prefix"

# Generic names
pattern: "(const|let|var)\s+(data|result|response|temp|tmp|val|value|item|thing|stuff)\s*="
context: "Generic variable name - be more descriptive"

# Abbreviated names
pattern: "(const|let|var)\s+(btn|msg|err|req|res|cb|fn|usr|mgr|util)\s*="
context: "Abbreviated name - prefer full words for readability"

# Class/component naming
pattern: "class\s+[a-z]|function\s+[a-z]\w*Component|const\s+[a-z]\w*\s*=\s*\(\s*\)\s*=>\s*<"
context: "Class/component should start with uppercase"
```

## Step 3: File-Level Analysis

After pattern scanning, perform file-level analysis:

1. **Use Glob** to list all source files.
2. **Check file sizes**: flag files over 500 lines.
3. **Check function count per file**: flag files with 20+ exports/functions.
4. **Check test coverage indicators**: look for corresponding test files.

## Step 4: Generate Report

Present findings in this format:

```
## Code Quality Analysis Report

**Project:** [project name]
**Language/Framework:** [detected stack]
**Analysis Date:** [current date]
**Files Analyzed:** [count]

### Quality Summary

| Category | Issues Found | Critical | Warning | Info |
|----------|-------------|----------|---------|------|
| Dead Code | X | X | X | X |
| Duplicate Code | X | X | X | X |
| Complexity | X | X | X | X |
| Error Handling | X | X | X | X |
| Async/Await | X | X | X | X |
| Memory Leaks | X | X | X | X |
| Performance | X | X | X | X |
| Accessibility | X | X | X | X |
| i18n | X | X | X | X |
| Naming | X | X | X | X |

### Maintainability Score: [A-F]

**Scoring Criteria:**
- A (90-100): Excellent - minimal issues, clean codebase
- B (75-89): Good - minor issues, well-maintained
- C (60-74): Acceptable - some issues need attention
- D (40-59): Poor - significant issues, refactoring needed
- F (0-39): Critical - major issues, high technical debt

**Score Calculation:**
- Start at 100
- Critical issue: -5 points each
- Warning: -2 points each
- Info: -0.5 points each
- Bonus: +5 for test coverage indicators, +5 for linting config, +5 for CI/CD

---

### Detailed Findings

#### Category: [Category Name]

##### Finding 1: [Short description]
- **Severity:** CRITICAL / WARNING / INFO
- **File:** `path/to/file.ts:42`
- **Evidence:** [relevant code snippet, max 5 lines]
- **Issue:** [what is wrong]
- **Suggestion:** [how to fix, with code example]
- **Auto-fixable:** Yes/No

[Repeat for each finding]

---

### Top 10 Priority Fixes

| # | Issue | File | Severity | Auto-fix |
|---|-------|------|----------|----------|
| 1 | ... | ... | CRITICAL | Yes/No |
| 2 | ... | ... | CRITICAL | Yes/No |
| ... | ... | ... | ... | ... |

### Technical Debt Estimate

| Category | Issues | Estimated Effort |
|----------|--------|-----------------|
| Dead Code Cleanup | X items | X hours |
| Duplicate Code Extraction | X items | X hours |
| Complexity Reduction | X items | X hours |
| Error Handling Improvement | X items | X hours |
| **Total** | **X items** | **X hours** |
```

## Step 5: Offer Next Steps

After the report, suggest:
- "Use `/nazar:fix` to auto-fix [N] applicable issues"
- "Use `/nazar:security-audit` for security-focused analysis"
- "Use `/nazar:compliance` for regulatory compliance checks"

## Important Notes

- Adjust patterns based on the detected language/framework.
- Do not report issues in `node_modules/`, `vendor/`, `dist/`, `build/`, `.next/`, `__pycache__/` directories.
- Do not report issues in test files unless they indicate a pattern used in production code.
- Consider framework conventions: React components should be PascalCase, Python should be snake_case.
- Be specific in suggestions: provide code examples tailored to the project's stack.
- For auto-fixable issues, provide the exact code change needed.
- The maintainability score should reflect the overall health of the codebase, not just issue count.
