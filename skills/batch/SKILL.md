---
description: "Scan multiple projects in parallel. Use when user says 'batch scan', 'toplu tarama', 'scan all projects', 'tum projeleri tara', 'scan directory'."
---

# Nazar Batch Scan

Scan multiple projects at once and compare results.

## How to run

1. Accept a parent directory path (e.g. ~/projects/)
2. Use Glob to find project indicators in subdirectories:
   - package.json, pubspec.yaml, requirements.txt, Cargo.toml, go.mod, build.gradle, *.xcodeproj
3. List detected projects and confirm with user
4. For each project (max 4 parallel via Agent tool):
   - Dispatch nazar-skill:scanner agent with project path
   - Collect results
5. Present comparison table:

| Project | Framework | Grade | Critical | High | Medium | Low | Total |
|---------|-----------|-------|----------|------|--------|-----|-------|

6. Highlight the project with most critical issues
7. Suggest: "Run /nazar:scan on [worst project] for detailed analysis"
