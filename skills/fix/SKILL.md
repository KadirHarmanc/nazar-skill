---
description: "Auto-fix security and quality issues found by scan. Use when user says 'fix', 'duzelt', 'patch', 'sorunlari coz', 'otomatik duzelt', 'fix issues'."
---

# Nazar Fix Skill

This skill auto-fixes security and quality issues found by a previous /nazar:scan.

## Workflow

### Step 1: Check for Scan Results

Check if a scan was run in this conversation. Look for scan findings in the conversation context.

If no scan results exist, tell the user:

> Once /nazar:scan ile tarama yapin, sonra /nazar:fix ile duzeltme yapabilirsiniz.

Do not proceed further without scan results.

### Step 2: Analyze Findings

From the scan results, categorize each finding:

- **Auto-fixable**: Issues with clear, safe, mechanical fixes (hardcoded secrets, SQL injection, XSS, console.log/print, empty catch, weak crypto)
- **Needs manual review**: Issues where the fix might break functionality, requires domain knowledge, or has ambiguous resolution

### Step 3: Show Summary

Present to the user:

```
Tarama Sonuclari:
- Toplam: X sorun bulundu
- Otomatik duzeltilebilir: Y sorun
- Manuel inceleme gerekli: Z sorun

Dagilim:
- Kritik: N
- Yuksek: N
- Orta: N
- Dusuk: N
```

### Step 4: Ask User Preference

Ask the user how they want to proceed:

- **Fix all**: Fix all auto-fixable issues at once
- **Select specific**: Let user choose which issues to fix
- **Review first**: Show each proposed fix before applying

If the user does not specify, default to "review first" mode for critical/high severity and "fix all" for medium/low.

### Step 5: Dispatch Fixer Agent

Send the selected findings to the `nazar-skill:fixer` agent. The agent will:
- Process findings in severity order (critical first)
- Apply fixes one by one
- Report each change made

### Step 6: Show Results Table

After fixing, show a summary table:

```
| Dosya | Sorun | Ciddiyet | Durum |
|-------|-------|----------|-------|
| src/db.js:15 | SQL Injection | Kritik | Duzeltildi |
| src/auth.js:8 | Hardcoded Secret | Kritik | Duzeltildi |
| src/utils.js:42 | console.log | Dusuk | Duzeltildi |
| src/api.js:99 | Deprecated API | Orta | Manuel Inceleme |
```

Status values:
- **Duzeltildi**: Successfully fixed
- **Atlandi**: Skipped (not selected by user)
- **Manuel Inceleme**: Needs manual review (too risky to auto-fix)
- **Hata**: Fix attempted but failed

### Step 7: Suggest Verification

After all fixes are applied, suggest:

> Duzeltmeleri dogrulamak icin tekrar /nazar:scan calistirmanizi oneririm.

## Notes

- Never fix issues without user awareness. Always show what will change.
- If a fix requires installing a new dependency (e.g., DOMPurify, bcrypt), inform the user and list the required packages.
- Group related fixes when possible (e.g., if multiple files have the same hardcoded secret, fix them all together with one .env entry).
- Track which files were modified so the user can review changes with git diff.
