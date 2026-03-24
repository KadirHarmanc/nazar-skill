# General Security & Quality Rules

Cross-framework security and code quality patterns applicable to any language or stack.

---

## 1. Hardcoded Secrets

- **Name:** hardcoded-secrets
- **Regex:** `(sk-[a-zA-Z0-9]{20,}|AKIA[0-9A-Z]{16}|ghp_[a-zA-Z0-9]{36}|-----BEGIN\s+(RSA|EC|DSA|OPENSSH)\s+PRIVATE\s+KEY|password\s*=\s*["\'][^"\']{4,}|api_key\s*=\s*["\'][^"\']{4,})`
- **Severity:** critical
- **Description:** Detects hardcoded API keys, AWS access keys, GitHub tokens, private keys, and plaintext passwords or API keys assigned directly in source code.
- **FP Indicators:**
  - Value is a placeholder like `"your-api-key-here"`, `"changeme"`, `"xxx"`, or `"TODO"`
  - Located in test fixtures, mock data, or example/template files
  - The value references an environment variable or config lookup (e.g., `password=os.environ[...]`)
  - Comment lines explaining the format (e.g., `# password=<value>`)
- **Fix:** Move all secrets to environment variables, a secrets manager (AWS Secrets Manager, HashiCorp Vault), or a `.env` file excluded from version control via `.gitignore`.

---

## 2. Hardcoded Credentials

- **Name:** hardcoded-credentials
- **Regex:** `(username\s*=\s*["\'][a-zA-Z0-9_]+["\'].*password\s*=\s*["\'][^"\']+["\']|credentials\s*=\s*\{[^}]*password)`
- **Severity:** critical
- **Description:** Detects username/password pairs written directly in source code, including credential objects with embedded passwords.
- **FP Indicators:**
  - Located inside test/mock/fixture files with clearly fake values
  - Values are empty strings or documented placeholders
  - The block is a schema definition or type annotation, not runtime code
- **Fix:** Use a credentials provider, environment variables, or a vault service. Never commit real credentials to source control.

---

## 3. Debug Mode Enabled

- **Name:** debug-mode-enabled
- **Regex:** `(DEBUG\s*=\s*True|debug\s*[:=]\s*true|debugMode\s*[:=]\s*true|EnableDebugging\s*=\s*true)`
- **Severity:** high
- **Description:** Detects debug mode flags left enabled, which can expose stack traces, internal state, and verbose error messages in production.
- **FP Indicators:**
  - Located in development-only config files (e.g., `settings.dev.py`, `.env.development`)
  - Gated behind an environment check (`if env == "development"`)
  - Inside test configuration
- **Fix:** Set debug flags from environment variables and ensure they default to `false`/`False` in production. Use `DEBUG=os.environ.get("DEBUG", "False")` or equivalent.

---

## 4. Weak Cryptography

- **Name:** weak-crypto
- **Regex:** `(md5\s*\(|MD5\.Create|SHA1\s*\(|sha1\s*\(|hashlib\.md5|hashlib\.sha1|DES\.|DESede|RC4|Cipher\.getInstance\(\s*["\']DES|createHash\(\s*["\']md5|createHash\(\s*["\']sha1)`
- **Severity:** high
- **Description:** Detects use of cryptographically broken or weak algorithms (MD5, SHA1, DES, RC4) for security-sensitive operations like password hashing, signature verification, or encryption.
- **FP Indicators:**
  - Used for non-security purposes: checksums, cache keys, content fingerprinting
  - Explicitly commented as non-cryptographic usage
  - Used in test/mock code to generate deterministic values
- **Fix:** Replace MD5/SHA1 with SHA-256 or SHA-3 for hashing. Replace DES/RC4 with AES-256-GCM for encryption. For passwords, use bcrypt, scrypt, or Argon2.

---

## 5. Insecure HTTP

- **Name:** insecure-http
- **Regex:** `http://[a-zA-Z0-9][a-zA-Z0-9\-\.]+\.[a-zA-Z]{2,}`
- **Severity:** medium
- **Description:** Detects plaintext HTTP URLs that should use HTTPS, risking man-in-the-middle attacks and data interception.
- **FP Indicators:**
  - URL points to localhost or 127.0.0.1
  - Located in test files, comments, or documentation
  - Used in a development-only configuration
  - References XML namespaces or schema URIs (e.g., `http://www.w3.org/`)
- **Fix:** Replace `http://` with `https://` for all production endpoints. Enforce HTTPS at the infrastructure level with HSTS headers.

---

## 6. Console/Print Statements

- **Name:** console-print-statements
- **Regex:** `(console\.(log|debug|info|warn|error)\s*\(|print\s*\(|System\.out\.println|fmt\.Println|puts\s+["\']|NSLog\s*\()`
- **Severity:** low
- **Description:** Detects leftover console/print statements that can leak information and clutter production logs.
- **FP Indicators:**
  - Located in a dedicated logger or logging utility module
  - Inside test or debug-only files
  - Used in CLI tools where stdout output is expected behavior
  - Wrapped in a debug/development environment check
- **Fix:** Replace with a structured logging library (e.g., Winston, Pino, Python `logging`, Log4j). Remove debug prints before merging to production branches.

---

## 7. TODO/FIXME/HACK Markers

- **Name:** todo-fixme-hack
- **Regex:** `(TODO|FIXME|HACK|XXX|WORKAROUND)\s*[:;]?\s*\S`
- **Severity:** low
- **Description:** Detects code markers indicating incomplete work, known issues, or temporary workarounds that should be resolved before production release.
- **FP Indicators:**
  - Part of a tracked issue reference (e.g., `TODO(#1234)`)
  - Located in documentation or changelog files
  - In a backlog/roadmap file
- **Fix:** Resolve the underlying issue, create a tracked ticket if needed, and remove the marker. Use issue trackers instead of code comments for long-term work items.

---

## 8. Commented-Out Code Blocks

- **Name:** commented-out-code
- **Regex:** `(//\s*(if|for|while|return|const|let|var|function)\s|#\s*(def|class|if|for|return|import)\s|/\*[\s\S]*?(function|return|var)[\s\S]*?\*/)`
- **Severity:** low
- **Description:** Detects blocks of commented-out code that add noise, confuse maintainers, and may contain stale logic.
- **FP Indicators:**
  - Single-line explanatory comments that happen to start with a keyword
  - Documentation comments with code examples
  - License header blocks
- **Fix:** Remove commented-out code. Use version control history to recover old code if needed. If the code is for reference, move it to documentation.

---

## 9. Empty Catch/Except Blocks

- **Name:** empty-catch-except
- **Regex:** `(catch\s*\([^)]*\)\s*\{\s*\}|except\s*.*:\s*pass|except\s*.*:\s*\.\.\.|\bcatch\b\s*\{\s*\})`
- **Severity:** medium
- **Description:** Detects empty exception handlers that silently swallow errors, making debugging extremely difficult and hiding potential failures.
- **FP Indicators:**
  - Contains an explicit comment explaining why the exception is intentionally ignored
  - Used in cleanup/finally logic where failure is acceptable
  - Test code that verifies an exception is thrown
- **Fix:** At minimum, log the exception. Prefer specific exception types over bare catches. Re-raise or handle meaningfully: `catch (e) { logger.error("context", e); }`.

---

## 10. Hardcoded IP Addresses

- **Name:** hardcoded-ip-addresses
- **Regex:** `(["'\s=:](10\.\d{1,3}\.\d{1,3}\.\d{1,3}|192\.168\.\d{1,3}\.\d{1,3}|172\.(1[6-9]|2[0-9]|3[01])\.\d{1,3}\.\d{1,3})["\s,;:])`
- **Severity:** medium
- **Description:** Detects hardcoded private/internal IP addresses that indicate environment-specific configuration leaking into source code.
- **FP Indicators:**
  - Located in network configuration files, Docker Compose, or infrastructure-as-code
  - Used in test fixtures for network testing
  - Documentation or comments explaining network topology
- **Fix:** Use DNS names, service discovery, or environment variables instead of hardcoded IPs. Define IPs in environment-specific config files excluded from source control.

---

## 11. Sensitive Data in Logs

- **Name:** sensitive-data-logging
- **Regex:** `(log(ger)?\.(info|debug|warn|error|log)\s*\(.*\b(password|secret|token|apiKey|api_key|authorization|credit.?card|ssn|social.?security)\b|console\.(log|debug|info)\s*\(.*\b(password|token|secret|key)\b)`
- **Severity:** high
- **Description:** Detects logging statements that may include sensitive data such as passwords, tokens, API keys, or personal information.
- **FP Indicators:**
  - Logging a message string that mentions the word but not the value (e.g., `"password updated successfully"`)
  - Located in test code with fake/mock values
  - The sensitive field is explicitly masked or redacted before logging
- **Fix:** Never log sensitive values directly. Use a log redaction library or mask values: `logger.info("auth", { token: "***" })`. Implement structured logging with field-level filtering.

---

## 12. Insecure Random

- **Name:** insecure-random
- **Regex:** `(Math\.random\s*\(|random\.random\s*\(|random\.randint\s*\(|rand\s*\(\s*\)|srand\s*\(|java\.util\.Random\b)`
- **Severity:** high
- **Description:** Detects use of non-cryptographic random number generators in contexts that may require security (tokens, keys, session IDs, OTPs).
- **FP Indicators:**
  - Used for UI purposes (animations, shuffling display order, jitter)
  - Located in game logic or non-security-critical simulation
  - Test code generating random test data
- **Fix:** Use cryptographically secure alternatives: `crypto.randomBytes()` / `crypto.randomUUID()` (Node.js), `secrets.token_hex()` (Python), `SecureRandom` (Java/Kotlin), `arc4random` (Swift/C).

---

## 13. Path Traversal

- **Name:** path-traversal
- **Regex:** `(\.\./|\.\.\\|path\.(join|resolve)\s*\(.*req\.(params|query|body)|readFile\s*\(.*req\.|open\s*\(.*request\.(GET|POST|args))`
- **Severity:** high
- **Description:** Detects potential path traversal vulnerabilities where user input is used to construct file paths without proper sanitization.
- **FP Indicators:**
  - The path is validated/sanitized before use (e.g., `path.normalize`, checking for `..`)
  - Relative imports in module resolution (e.g., `from ../utils import`)
  - Static asset references in CSS or HTML
- **Fix:** Validate and sanitize all user-supplied path components. Use `path.resolve()` and verify the result stays within an allowed base directory. Reject paths containing `..` sequences.

---

## 14. Dynamic Code Execution

- **Name:** dynamic-code-execution
- **Regex:** `(\beval\s*\(|\bexec\s*\(|\bFunction\s*\(\s*["\']|new\s+Function\s*\(|setTimeout\s*\(\s*["\']|setInterval\s*\(\s*["\'])`
- **Severity:** critical
- **Description:** Detects dynamic code execution functions that can lead to arbitrary code execution if fed user-controlled input. Includes eval, exec, Function constructor, and string-based timer calls.
- **FP Indicators:**
  - Used in build tools or code generators (Webpack config, Babel)
  - Exec calling system commands with hardcoded arguments (no user input)
  - Located in REPL or developer tooling code
  - Python exec in migration scripts with trusted input
- **Fix:** Avoid dynamic code execution entirely. Use `JSON.parse()` instead of eval for JSON. Use template engines instead of string-based code generation. If dynamic execution is unavoidable, use a sandboxed environment.

---

## 15. SQL String Concatenation

- **Name:** sql-string-concatenation
- **Regex:** `(["']SELECT\s.*["']\s*\+|["']INSERT\s.*["']\s*\+|["']UPDATE\s.*["']\s*\+|["']DELETE\s.*["']\s*\+|f["\']SELECT\s|f["\']INSERT\s|f["\']UPDATE\s|f["\']DELETE\s|\.format\(.*SELECT|\.format\(.*INSERT|["']SELECT\s.*%s|\.query\(\s*["']SELECT\s.*\+)`
- **Severity:** critical
- **Description:** Detects SQL queries built via string concatenation or interpolation, which is the primary vector for SQL injection attacks.
- **FP Indicators:**
  - Using parameterized query placeholders (e.g., `?`, `$1`, `%s` with proper parameter binding)
  - Located in SQL migration or schema files with no user input
  - ORM-generated query logging
  - Test fixtures building queries with hardcoded values
- **Fix:** Always use parameterized queries or prepared statements. Use an ORM (SQLAlchemy, Prisma, Django ORM, Sequelize). Never concatenate user input into SQL strings.
