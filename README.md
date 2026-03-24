<p align="center">
  <br>
  <span style="font-size: 80px">🧿</span>
  <br>
</p>

<h1 align="center">Nazar Skill</h1>
<h3 align="center">Claude Code Plugin for Autonomous Security & Quality Scanning</h3>

<p align="center">
  <em>12 skills. 5 agents. 280+ checks. 13 frameworks. Zero config.</em>
</p>

<p align="center">
  <a href="https://github.com/KadirHarmanc/nazar-skill"><img src="https://img.shields.io/badge/Claude_Code-Plugin-blueviolet" alt="Claude Code Plugin"></a>
  <a href="https://opensource.org/licenses/MIT"><img src="https://img.shields.io/badge/License-MIT-blue.svg" alt="License"></a>
  <a href="https://github.com/KadirHarmanc/nazar"><img src="https://img.shields.io/badge/Powered_by-Nazar-orange" alt="Nazar"></a>
</p>

---

Nazar Skill turns Claude Code into a security scanner. Type `/nazar:scan` and it detects your tech stack, runs 280+ security & quality checks, and reports findings with fix suggestions. No setup, no dependencies, no config files.

```
/nazar:scan
```

That's it. It figures out the rest.

---

## Install

Add to your Claude Code plugins:

```bash
# Clone into your plugins directory
git clone https://github.com/KadirHarmanc/nazar-skill.git ~/.claude/plugins/nazar-skill

# Restart Claude Code
```

Done. All `/nazar:*` commands are now available.

---

## Skills

| Skill | What it does |
|-------|-------------|
| `/nazar:scan` | Scan project for security & quality issues. Auto-detects framework, runs parallel agents. |
| `/nazar:fix` | Auto-fix found issues. Moves secrets to .env, patches injections, removes debug code. |
| `/nazar:preflight` | App Store (22 rules) + Play Store (12 rules) pre-submission checks. |
| `/nazar:security-audit` | Deep OWASP analysis: Web Top 10, Mobile Top 10 (2024), API Top 10. |
| `/nazar:compliance` | GDPR/KVKK, SOC 2, PCI-DSS v4.0, HIPAA regulatory checks. |
| `/nazar:code-quality` | Dead code, complexity, error handling, naming conventions, duplicates. |
| `/nazar:health` | System dependency check: Python, simulator, adb, ffmpeg, ports. |
| `/nazar:report` | Generate reports in Markdown, HTML (dark theme), or JSON (CI/CD). |
| `/nazar:live-test` | Run YAML UI tests on iOS simulator / Android emulator. |
| `/nazar:yaml-gen` | Auto-generate YAML test files from project structure. |
| `/nazar:batch` | Scan multiple projects in parallel, compare results. |
| `/nazar:coverage` | Show scan coverage metrics and gaps. |

---

## Natural Language

You don't need slash commands. Just talk:

- "projemi tara" / "scan this project"
- "guvenlik kontrol" / "check for security issues"
- "sorunlari duzelt" / "fix the issues"
- "app store kontrol" / "check store compliance"

Claude understands and triggers the right skill automatically.

---

## What It Checks

### Security (all projects)
Hardcoded secrets, credentials, debug mode, weak crypto, insecure HTTP, eval/exec, SQL injection, path traversal, sensitive data in logs, insecure random.

### Mobile
**React Native:** AsyncStorage secrets, deep link validation, bridge security, Hermes exposure, WebView injection.
**Flutter:** shared_preferences secrets, DevTools in production, platform channel validation, obfuscation.
**Expo:** secure-store vs AsyncStorage, OTA security, environment variables.

### Web
**React:** XSS via innerHTML, localStorage secrets, CORS, CSP missing.
**Vue:** v-html injection, template injection, Vuex exposure.
**Angular:** security trust bypass, template injection, RXJS error handling.

### API
**Django:** DEBUG=True, ALLOWED_HOSTS, SECRET_KEY, raw SQL, csrf_exempt, CORS.
**FastAPI:** CORS wildcard, Pydantic Any, missing auth, rate limiting.
**Express:** helmet, rate-limit, prototype pollution, NoSQL injection.

### Native
**iOS (Swift/ObjC):** NSUserDefaults secrets, Keychain, ATS, UIWebView, jailbreak detection.
**Android (Kotlin/Java):** SharedPreferences, WebView JS, Intent validation, exported components.

---

## Agents

| Agent | Model | Role |
|-------|-------|------|
| **scanner** | Sonnet | Orchestrator -- detects framework, dispatches agents, merges results |
| **security** | Sonnet | Security scanning -- secrets, injection, crypto, auth, dependencies |
| **quality** | Haiku | Quality scanning -- dead code, complexity, error handling, naming |
| **fixer** | Sonnet | Auto-fix -- applies patches via Edit tool with context awareness |
| **preflight** | Haiku | Store compliance -- iOS App Store + Google Play rule checking |

Scan runs **security** and **quality** agents in parallel. Results are merged, deduplicated, filtered by confidence score (threshold: 60), and sorted by severity.

---

## Hooks

Three automatic hooks, active from install:

| Hook | Event | What it does |
|------|-------|-------------|
| Secret Guard | PreToolUse (Write/Edit) | Blocks writing hardcoded secrets to files |
| Scan Reminder | Stop | Reminds to scan if code was changed but no scan was run |
| Preflight Nudge | UserPromptSubmit | Suggests preflight check when deploying |

---

## Confidence Scoring

Every finding goes through AI filtering to minimize false positives:

```
Score = 0.25 * pattern_match
      + 0.30 * context_risk
      + 0.30 * data_flow
      + 0.15 * (1 - framework_protection)

80+  : Critical -- report, suggest fix
60-80: Warning  -- report, lower priority
40-60: Info     -- verbose mode only
<40  : Suppress -- skip
```

Target false positive rate: **< 10%**

---

## Supported Frameworks

| Category | Frameworks |
|----------|-----------|
| **Mobile** | React Native, Flutter, Expo |
| **Web** | React, Vue, Angular |
| **Backend** | Django, FastAPI, Express |
| **Native** | Swift/Objective-C, Kotlin/Java |
| **Systems** | Go, Rust |

---

## Compliance Frameworks

| Framework | What it checks |
|-----------|---------------|
| **GDPR/KVKK** | Consent, data deletion, export, PII masking, encryption, retention |
| **SOC 2** | RBAC, audit logging, encryption, secret management, rate limiting |
| **PCI-DSS v4.0** | CSP, SRI, PAN masking, MFA, TLS, key management |
| **HIPAA** | PHI detection, encryption, access logging, BAA, audit trail |

---

## OWASP Coverage

| Framework | Version | Items |
|-----------|---------|-------|
| OWASP Top 10 | 2021 | A01-A10 (Web) |
| OWASP Mobile Top 10 | 2024 | M01-M10 (Mobile) |
| OWASP API Security | 2023 | API1-API10 (API) |

Each item includes CWE references, detection patterns, exploit scenarios, and fix guides.

---

## App Store & Play Store

### iOS App Store (22 rules)
Info.plist permissions, PrivacyInfo.xcprivacy (iOS 17+), Required Reason API, UIWebView deprecated, ATS, icon sizes, launch screen, bundle ID, minimum iOS, push entitlements, IAP, IPv6, 64-bit, IDFA/ATT, SDK manifests, debug mode, placeholder content, crash patterns, background modes, extensions, keychain, associated domains.

### Google Play (12 rules)
targetSdkVersion, Data Safety Form, QUERY_ALL_PACKAGES, granular permissions (Android 13+), POST_NOTIFICATIONS, foreground service type, debuggable, allowBackup, exported components, ProGuard/R8, signing config, content rating.

---

## Project Structure

```
nazar-skill/
  .claude-plugin/
    plugin.json              # Plugin manifest

  skills/
    scan/SKILL.md            # Core scanner
    scan/rules/              # 6 rule files (108 patterns)
    fix/SKILL.md             # Auto-fixer
    preflight/SKILL.md       # Store checks
    preflight/rules/         # 4 rule files (34 rules)
    security-audit/SKILL.md  # OWASP deep audit
    security-audit/frameworks/ # 3 OWASP frameworks
    compliance/SKILL.md      # Regulatory checks
    compliance/frameworks/   # 4 compliance frameworks
    code-quality/SKILL.md    # Quality analysis
    health/SKILL.md          # System check
    report/SKILL.md          # Report generator
    live-test/SKILL.md       # Simulator testing
    yaml-gen/SKILL.md        # Test generator
    batch/SKILL.md           # Multi-project scan
    coverage/SKILL.md        # Coverage metrics

  agents/
    scanner/AGENT.md         # Orchestrator (Sonnet)
    security/AGENT.md        # Security agent (Sonnet)
    quality/AGENT.md         # Quality agent (Haiku)
    fixer/AGENT.md           # Fix agent (Sonnet)
    preflight/AGENT.md       # Store agent (Haiku)

  hooks/
    hooks.json               # 3 automatic hooks
```

---

## How It Works

1. **Framework Detection** -- Glob for package.json, pubspec.yaml, requirements.txt, etc. Read to confirm stack.
2. **Parallel Scan** -- Security agent + Quality agent run simultaneously with relevant rule files.
3. **AI Filtering** -- Merge results, remove duplicates, calculate confidence scores, drop below threshold.
4. **Report** -- Present findings sorted by severity with fix suggestions.

No external tools installed. No API keys. No configuration. Just Claude Code's built-in tools: Grep, Glob, Read, Edit, Bash, Agent.

---

## Related

- [Nazar CLI](https://github.com/KadirHarmanc/nazar) -- Standalone security scanner (pip install nazar)

---

## License

MIT
