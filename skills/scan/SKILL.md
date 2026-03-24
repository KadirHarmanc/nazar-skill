---
description: "Scan a project for security vulnerabilities and code quality issues. Zero config - auto-detects your tech stack. Use when user says 'scan', 'tara', 'guvenlik kontrol', 'check my code', 'audit', 'kodumu incele', 'sorun var mi', 'review code', 'security check'."
---

# Nazar Scan

Autonomous security and quality scanner. Detects your tech stack, runs 280+ checks, reports findings with fix suggestions.

## How to run

1. Determine the project directory:
   - If user specified a path, use that
   - Otherwise use current working directory

2. Dispatch the `nazar-skill:scanner` agent with the project path. The scanner agent will:
   - Auto-detect frameworks (React Native, Flutter, Django, FastAPI, Express, etc.)
   - Launch security and quality agents in parallel
   - Merge findings, filter false positives, sort by severity
   - Present results

3. After scan completes, suggest next steps:
   - `/nazar:fix` -- auto-fix found issues
   - `/nazar:report` -- generate full report (Markdown/HTML/JSON)
   - `/nazar:security-audit` -- deep OWASP analysis
   - `/nazar:preflight` -- App Store / Play Store checks (for mobile projects)

## Supported frameworks
React Native, Flutter, Expo, React, Vue, Angular, Django, FastAPI, Express, Swift/ObjC, Kotlin/Java, Go, Rust (13 frameworks)

## What it checks
- Hardcoded secrets and credentials
- Injection vulnerabilities (SQL, XSS, NoSQL)
- Authentication and authorization issues
- Cryptographic weaknesses
- Dependency vulnerabilities
- Code quality (dead code, complexity, error handling)
- Framework-specific misconfigurations
- And 280+ more patterns
