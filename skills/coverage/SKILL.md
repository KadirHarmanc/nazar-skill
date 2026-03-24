---
description: "Show scan coverage metrics and what was checked. Use when user says 'coverage', 'kapsam', 'scan coverage', 'what was checked', 'ne kontrol edildi', 'scan summary'."
---

# Nazar Coverage Report

Show what was checked and what might have been missed.

## How to run

1. Check if scan results exist in conversation context. If not, suggest /nazar:scan first.
2. Calculate and present:
   - **Files scanned:** total files, by type (js/ts/py/dart/swift/kt/go/rs)
   - **Rules applied:** total active rules, by category (security/quality/platform)
   - **Issues found:** by severity (critical/high/medium/low)
   - **Framework coverage:** which frameworks detected, which rules applied
   - **Scan duration:** time taken
   - **Estimated false positive rate:** based on confidence distribution
3. Identify gaps:
   - Files/directories skipped (node_modules, .venv, etc.)
   - Rule categories not applicable (e.g., no mobile rules for a pure API project)
   - Potential areas not covered by current rules
4. Suggest next steps:
   - "/nazar:security-audit for deep OWASP analysis"
   - "/nazar:compliance for regulatory checks"
   - "/nazar:preflight for store submission checks"
