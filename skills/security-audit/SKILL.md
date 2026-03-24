---
description: "Deep OWASP security audit with exploit scenarios and CWE references. Use when user says 'security audit', 'OWASP check', 'guvenlik denetimi', 'penetration test', 'deep security scan', 'OWASP analiz'."
---

# Nazar Security Audit Skill

You are performing a deep OWASP-based security audit. Follow these instructions systematically.

## Step 1: Detect Project Type

Use Glob and Read to determine the project type:

**Web Application Indicators:**
- `package.json` with express/next/nuxt/react/angular/vue/django/flask/rails
- Files: `*.html`, `*.ejs`, `*.jsx`, `*.tsx`, `*.vue`, `*.svelte`
- Directories: `pages/`, `views/`, `templates/`, `public/`

**Mobile Application Indicators:**
- `android/` directory, `AndroidManifest.xml`, `build.gradle`
- `ios/` directory, `Info.plist`, `Podfile`
- `app.json` with expo config, `react-native.config.js`
- `pubspec.yaml` (Flutter), `capacitor.config.ts`

**API Indicators:**
- `package.json` with express/fastify/koa/nest (without frontend)
- `requirements.txt` with flask/fastapi/django-rest-framework
- Files: `routes/`, `controllers/`, `resolvers/`, `schema.graphql`
- OpenAPI/Swagger spec files

If multiple types are detected, audit all applicable frameworks.

## Step 2: Load the Relevant OWASP Framework

Based on the detected project type, read the corresponding framework file:

- **Web:** Read `skills/security-audit/frameworks/owasp-web.md`
- **Mobile:** Read `skills/security-audit/frameworks/owasp-mobile.md`
- **API:** Read `skills/security-audit/frameworks/owasp-api.md`

## Step 3: Systematic Audit

For each of the 10 items in the loaded framework:

1. **Run all grep patterns** from the framework item against the project codebase using the Grep tool.
2. **For each match**, use Read to examine the surrounding code (at least 20 lines of context).
3. **Apply context analysis** instructions from the framework to determine if the match is a real finding or false positive.
4. **Classify findings** by severity:
   - **CRITICAL**: Exploitable with high impact (data breach, RCE, privilege escalation)
   - **HIGH**: Exploitable with moderate impact or requires some conditions
   - **MEDIUM**: Vulnerability exists but exploitation is limited
   - **LOW**: Best practice violation, defense-in-depth improvement
   - **INFO**: Informational finding, no direct vulnerability

## Step 4: Generate Report

Present findings in this format:

```
## OWASP Security Audit Report

**Project Type:** [Web/Mobile/API]
**Framework:** OWASP [Top 10 2021 / Mobile Top 10 2024 / API Security Top 10 2023]
**Scan Date:** [current date]

### Executive Summary
- Total Findings: X
- Critical: X | High: X | Medium: X | Low: X | Info: X

---

### [A01/M01/API1] - [Item Name]

**Status:** [PASS / FINDINGS DETECTED]

#### Finding 1: [Short description]
- **Severity:** CRITICAL/HIGH/MEDIUM/LOW/INFO
- **CWE:** CWE-XXX
- **File:** `path/to/file.ts:42`
- **Evidence:** [relevant code snippet]
- **Exploit Scenario:** [how an attacker could exploit this]
- **Remediation:** [specific fix with code example]
- **Priority:** P1/P2/P3/P4

[Repeat for each finding under this item]

---

[Repeat for each of the 10 OWASP items]

### Remediation Priority Matrix

| Priority | Finding | File | Effort |
|----------|---------|------|--------|
| P1 | ... | ... | Low/Med/High |
| P2 | ... | ... | Low/Med/High |

### Recommendations
1. [Top priority recommendation]
2. [Second priority recommendation]
...
```

## Step 5: Offer Next Steps

After the report, suggest:
- "Use `/nazar:fix` to auto-fix applicable findings"
- "Use `/nazar:compliance` to check regulatory compliance"
- "Run `npm audit` / `pip-audit` for dependency vulnerability details"

## Important Notes

- Always check for false positives by reading full context around grep matches.
- If a finding matches multiple OWASP items, list it under the most relevant one and cross-reference.
- Do not report findings for test files unless they indicate a pattern used in production.
- Consider the project's framework when evaluating findings (e.g., React auto-escapes JSX, Django auto-escapes templates).
- Be specific in remediation: provide code examples tailored to the project's stack.
