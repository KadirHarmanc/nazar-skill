# OWASP Top 10 (2021) - Web Application Security

Reference framework for deep security audit of web applications.

---

## A01 - Broken Access Control

**CWE References:** CWE-200, CWE-201, CWE-352, CWE-284, CWE-285, CWE-639, CWE-862, CWE-863, CWE-913

**Grep Patterns:**
```
# Missing authorization checks
pattern: "app\.(get|post|put|delete|patch)\s*\([^)]*\)\s*=>\s*\{"
context: "Route handler without auth middleware"

# Direct object reference
pattern: "params\.(id|userId|user_id|accountId)"
context: "Check if authorization validates ownership"

# CORS misconfiguration
pattern: "Access-Control-Allow-Origin.*\*"
context: "Wildcard CORS allows any origin"

# Missing CSRF protection
pattern: "app\.(post|put|delete|patch)"
context: "State-changing endpoint without CSRF token"

# Privilege escalation
pattern: "(role|isAdmin|is_admin|permission|isAuthorized)\s*[=!]"
context: "Client-side role check that can be bypassed"

# Insecure direct object reference
pattern: "findById|findOne|get.*ById"
context: "Database query using user-supplied ID without ownership check"

# JWT without verification
pattern: "jwt\.decode\s*\("
context: "JWT decoded without signature verification"

# Directory traversal
pattern: "\.\./|\.\.\\\\|path\.join.*req\.(params|query|body)"
context: "Path constructed from user input"
```

**Context Analysis Instructions:**
1. For every route handler, verify an authentication middleware is applied.
2. For every database query using a user-supplied ID, verify the query also checks ownership.
3. Check that CORS is configured with a specific origin whitelist, not wildcard.
4. Verify CSRF tokens are validated on all state-changing endpoints.
5. Ensure admin routes have role-based middleware, not just authentication.
6. Check that JWT tokens are verified with jwt.verify(), not just jwt.decode().

**Example Exploit Scenario:**
An authenticated user with role "user" can access /api/admin/users because the route only checks authentication but not authorization. The attacker accesses another user's data because the backend fetches by ID without verifying ownership.

**Fix Guide:**
- Implement middleware-based authorization on every route.
- Always validate resource ownership in database queries.
- Set CORS to specific origins.
- Use CSRF libraries (csurf for Express, built-in CSRF in Django/Rails).
- Default deny: deny access unless explicitly granted.

---

## A02 - Cryptographic Failures

**CWE References:** CWE-259, CWE-327, CWE-328, CWE-330, CWE-331, CWE-312, CWE-319, CWE-320, CWE-326

**Grep Patterns:**
```
# Weak hashing algorithms
pattern: "(md5|sha1|SHA1|MD5)\s*\(|createHash\s*\(\s*['\"]md5|createHash\s*\(\s*['\"]sha1"
context: "Weak hash algorithm for sensitive data"

# Hardcoded secrets
pattern: "(password|secret|api_key|apiKey|token|private_key)\s*[:=]\s*['\"][^'\"]{8,}"
context: "Hardcoded secret in source code"

# HTTP instead of HTTPS
pattern: "http://(?!localhost|127\.0\.0\.1|0\.0\.0\.0)"
context: "Unencrypted HTTP connection to external service"

# Weak encryption
pattern: "(DES|RC4|Blowfish|ECB)"
context: "Weak or deprecated encryption algorithm"

# Plaintext sensitive data in logs
pattern: "console\.(log|info|debug|warn).*?(password|token|secret|credit|ssn|card)"
context: "Sensitive data logged in plaintext"

# Insecure random
pattern: "Math\.random\(\)|random\.random\(\)"
context: "Cryptographically insecure random number generator"
```

**Context Analysis Instructions:**
1. Verify all password storage uses bcrypt, scrypt, or Argon2 with adequate cost factor.
2. Check that sensitive data in transit uses TLS 1.2+.
3. Ensure no sensitive data is logged.
4. Verify encryption uses AES-256-GCM or ChaCha20-Poly1305.
5. Check that cryptographic random is used for tokens/keys.

**Example Exploit Scenario:**
The application stores user passwords with MD5 hashing without salt. An attacker who gains database access uses rainbow tables to crack most passwords. API keys are hardcoded in the frontend bundle.

**Fix Guide:**
- Use bcrypt/scrypt/Argon2 for password hashing with cost factor >= 12.
- Use AES-256-GCM for encryption at rest.
- Move secrets to environment variables or a secrets manager.
- Use crypto.randomBytes() for security-sensitive random values.
- Add Strict-Transport-Security header.

---

## A03 - Injection

**CWE References:** CWE-20, CWE-74, CWE-75, CWE-77, CWE-78, CWE-79, CWE-89, CWE-90, CWE-94, CWE-643

**Grep Patterns:**
```
# SQL injection
pattern: "(query|execute|exec)\s*\(\s*[`'\"].*?\$\{|(\+\s*req\.(body|query|params))"
context: "String concatenation in SQL query"

# SQL injection (Python)
pattern: "cursor\.(execute|executemany)\s*\(\s*f['\"]"
context: "F-string formatting in SQL"

# NoSQL injection
pattern: "\$where|\$regex|\$gt|\$lt|\$ne.*req\.(body|query|params)"
context: "MongoDB operator injection"

# XSS via innerHTML
pattern: "\.innerHTML\s*="
context: "Setting innerHTML can lead to XSS if user content is not sanitized"

# XSS via framework unsafe APIs
pattern: "v-html|ng-bind-html|\|safe\b|\|raw\b"
context: "Framework API that renders unescaped HTML"

# Command injection
pattern: "exec\s*\(|spawn\s*\(|system\s*\(|popen\s*\(|child_process"
context: "OS command execution with possible user input"

# Template injection (SSTI)
pattern: "render_template_string|Template\s*\(.*req"
context: "Server-side template injection"

# Dynamic code construction
pattern: "Function\s*\(.*?(req|user|input|param|data)"
context: "Dynamic code construction from user input - injection risk"

# LDAP injection
pattern: "ldap\.(search|bind).*[\+\$]"
context: "LDAP query with user input"

# Log injection
pattern: "logger?\.(info|warn|error|debug)\s*\(.*req\.(body|query|params)"
context: "User input directly in log message"
```

**Context Analysis Instructions:**
1. For every database query, verify parameterized queries or ORM is used.
2. For every HTML rendering, verify output encoding or a safe templating engine.
3. For every OS command execution, verify user input is not passed.
4. Check all user input flows from entry to database/template/command.
5. Verify Content-Security-Policy header is set.
6. Check for dynamic code construction with user-supplied data.

**Example Exploit Scenario:**
A search endpoint constructs SQL with string concatenation. An attacker sends a UNION SELECT payload to extract the user table. The application also renders search results by setting innerHTML with user content, allowing stored XSS.

**Fix Guide:**
- Use parameterized queries everywhere.
- Use ORM methods that auto-parameterize.
- For XSS: use framework auto-escaping. Avoid innerHTML with user data.
- Set CSP header.
- For commands: avoid exec/system; use specific APIs or allowlist approach.

---

## A04 - Insecure Design

**CWE References:** CWE-73, CWE-183, CWE-209, CWE-256, CWE-501, CWE-522, CWE-602, CWE-656

**Grep Patterns:**
```
# No rate limiting on sensitive endpoints
pattern: "app\.(post)\s*\(\s*['\"]/(login|register|forgot|reset|verify|otp)"
context: "Sensitive endpoint - check for rate limiting middleware"

# Business logic bypass
pattern: "(price|amount|quantity|discount|total)\s*=\s*req\.(body|query|params)"
context: "User-controllable business logic value"

# Enumeration vectors
pattern: "(User not found|Invalid username|Email not registered|No account)"
context: "Different error messages reveal valid accounts"

# Missing multi-step validation
pattern: "(step|wizard|checkout)\s*=\s*req\."
context: "Multi-step process controlled by client"
```

**Context Analysis Instructions:**
1. Review authentication flows for rate limiting, lockout, and brute-force protection.
2. Check business logic for client-side trust.
3. Verify multi-step processes validate each step server-side.
4. Look for enumeration vectors in error messages.
5. Check that financial operations are idempotent and atomic.

**Example Exploit Scenario:**
An e-commerce checkout allows the client to send the final price in the request body. An attacker changes the total to 0.01. The login endpoint returns different messages for invalid username vs wrong password.

**Fix Guide:**
- Never trust client-side values for business logic; always recalculate server-side.
- Implement rate limiting on authentication endpoints.
- Use generic error messages: "Invalid credentials".
- Implement account lockout with exponential backoff.

---

## A05 - Security Misconfiguration

**CWE References:** CWE-2, CWE-11, CWE-13, CWE-15, CWE-16, CWE-388, CWE-489, CWE-497, CWE-611

**Grep Patterns:**
```
# Debug mode in production
pattern: "(DEBUG|debug)\s*[:=]\s*(true|True|1|'true')"
context: "Debug mode enabled"

# Stack trace exposure
pattern: "err\.stack|stackTrace"
context: "Error handler may expose stack traces"

# Missing security headers
pattern: "helmet|X-Content-Type-Options|X-Frame-Options|Strict-Transport"
context: "Check if security headers are configured"

# XML External Entity
pattern: "XMLParser|xml2js|parseXML|etree\.parse|lxml|DOMParser"
context: "XML parsing - check if external entities are disabled"

# CORS permissive
pattern: "credentials:\s*true.*origin:\s*true"
context: "CORS reflects any origin with credentials"

# Open redirect
pattern: "redirect\s*\(\s*req\.(query|body|params)\."
context: "Redirect URL from user input"
```

**Context Analysis Instructions:**
1. Check for debug mode in production configuration files.
2. Verify security headers are set.
3. Ensure error responses do not include stack traces.
4. Check XML parsers disable external entities.
5. Verify default accounts are disabled.

**Example Exploit Scenario:**
A production application with DEBUG=true exposes database connection strings in error messages. XML file upload does not disable external entities, allowing XXE.

**Fix Guide:**
- Never enable debug in production.
- Configure helmet.js or equivalent.
- Implement a generic error handler for production.
- Disable XML external entities.

---

## A06 - Vulnerable and Outdated Components

**CWE References:** CWE-1035, CWE-1104

**Grep Patterns:**
```
# Package files
pattern: "package\.json|requirements\.txt|Gemfile|pom\.xml|go\.mod"
context: "Dependency manifest file"

# Unpinned versions
pattern: "\"[^\"]+\":\s*\"[\^~>]"
context: "Unpinned dependency version"

# CDN scripts without SRI
pattern: "<script.*src=['\"]https?://cdn\."
context: "External resource without Subresource Integrity"

# Deprecated APIs
pattern: "(Buffer\s*\(\s*['\"]|crypto\.createCipher\b|url\.parse\()"
context: "Deprecated API with known issues"
```

**Context Analysis Instructions:**
1. Check for outdated major versions in dependency manifests.
2. Verify lock files exist and are committed.
3. Check if dependency scanning is in CI/CD pipeline.
4. Look for CDN scripts without SRI hashes.

**Example Exploit Scenario:**
The application uses a version of lodash with prototype pollution vulnerability. Dependencies are not regularly updated and no scanning is in place.

**Fix Guide:**
- Run dependency audit tools regularly.
- Configure Dependabot or Renovate.
- Pin dependencies to exact versions.
- Add SRI hashes to CDN-loaded scripts.

---

## A07 - Identification and Authentication Failures

**CWE References:** CWE-255, CWE-259, CWE-287, CWE-307, CWE-384, CWE-521, CWE-613

**Grep Patterns:**
```
# Weak password policy
pattern: "(minLength|min_length|password.*length)\s*[:=<>]\s*[1-7][^0-9]"
context: "Password minimum length less than 8"

# Insecure session config
pattern: "(cookie|session).*secure\s*:\s*false|httpOnly\s*:\s*false"
context: "Insecure cookie/session configuration"

# JWT misconfiguration
pattern: "algorithm.*none|verify\s*:\s*false"
context: "JWT accepts 'none' algorithm or skips verification"

# Password in URL
pattern: "password.*[?&]|[?&].*password"
context: "Password transmitted in URL query string"
```

**Context Analysis Instructions:**
1. Verify password policy enforces minimum 8 characters.
2. Check that sessions are regenerated after authentication.
3. Verify session cookies have secure, httpOnly, and sameSite flags.
4. Check for MFA implementation.
5. Verify JWT uses strong algorithms.
6. Check for rate limiting on login endpoints.

**Example Exploit Scenario:**
Unlimited login attempts allow credential stuffing. JWT accepts the "none" algorithm, enabling token forgery.

**Fix Guide:**
- Enforce strong password policy.
- Implement rate limiting on auth endpoints.
- Implement MFA (TOTP) for all users.
- Set cookie flags: secure, httpOnly, sameSite: strict.
- Configure JWT with explicit algorithm allowlist.

---

## A08 - Software and Data Integrity Failures

**CWE References:** CWE-345, CWE-353, CWE-426, CWE-494, CWE-502, CWE-565, CWE-784, CWE-829, CWE-830, CWE-915

**Grep Patterns:**
```
# Insecure deserialization (Python)
pattern: "yaml\.load\s*\((?!.*SafeLoader|.*safe_load)"
context: "YAML load without SafeLoader - arbitrary code execution risk"

# Insecure deserialization (PHP)
pattern: "unserialize\s*\("
context: "PHP unserialize with untrusted data"

# Insecure deserialization (Java)
pattern: "ObjectInputStream|readObject"
context: "Java deserialization of untrusted data"

# CDN without SRI
pattern: "<script[^>]*src=['\"]https?://[^'\"]*cdn"
context: "External script - check for SRI attribute"

# Mass assignment
pattern: "(Object\.assign|\.update\(req\.body|\.create\(req\.body|\*\*request\.(data|POST))"
context: "Mass assignment from user input"

# Unsigned cookies
pattern: "cookie.*unsigned|signed\s*:\s*false"
context: "Cookies without signature"
```

**Context Analysis Instructions:**
1. Check all deserialization calls use safe methods.
2. Verify external scripts have SRI hashes.
3. Look for mass assignment vulnerabilities.
4. Verify cookies and tokens are signed.
5. Check CI/CD pipeline integrity.

**Example Exploit Scenario:**
A Python application uses yaml.load without SafeLoader, allowing code execution via crafted YAML. Mass assignment through Object.assign(user, req.body) allows privilege escalation by setting isAdmin:true.

**Fix Guide:**
- Use yaml.safe_load() everywhere.
- Never pass entire request body to ORM; use allowlists.
- Add SRI hashes to all external scripts.
- Use signed cookies.

---

## A09 - Security Logging and Monitoring Failures

**CWE References:** CWE-117, CWE-223, CWE-532, CWE-778

**Grep Patterns:**
```
# Missing logging on security events
pattern: "(login|logout|register|password.*reset|role.*change|delete.*user)"
context: "Security-relevant event - check if logged"

# Sensitive data in logs
pattern: "(log|logger|console\.log|print)\s*\(.*?(password|token|secret|credit|ssn)"
context: "Sensitive data potentially in logs"

# Empty catch blocks
pattern: "catch\s*\([^)]*\)\s*\{[\s]*\}"
context: "Empty catch block - error swallowed"

# No alerting
pattern: "(alert|notify|alarm|pagerduty|opsgenie)"
context: "Check if security alerting is configured"
```

**Context Analysis Instructions:**
1. Verify all authentication events are logged.
2. Check that authorization failures are logged.
3. Verify sensitive data is not in log messages.
4. Check for centralized logging.
5. Check for real-time alerting on security events.

**Example Exploit Scenario:**
Brute-force attacks go undetected because failed logins are not logged. There is no forensic trail after a breach.

**Fix Guide:**
- Log all security events with structured format.
- Never log sensitive data; use PII redaction.
- Send logs to centralized, append-only storage.
- Set up alerts for suspicious patterns.

---

## A10 - Server-Side Request Forgery (SSRF)

**CWE References:** CWE-918

**Grep Patterns:**
```
# URL from user input
pattern: "(fetch|axios|request|http\.get|urllib|requests\.get|HttpClient)\s*\(\s*(req\.|request\.|params|user)"
context: "HTTP request with user-supplied URL"

# URL in request body
pattern: "(url|uri|link|href|endpoint|webhook|callback)\s*[:=]\s*req\.(body|query|params)"
context: "URL parameter from user input"

# Cloud metadata
pattern: "(169\.254\.169\.254|metadata\.google|metadata\.azure)"
context: "Cloud metadata endpoint access"

# Image/PDF from URL
pattern: "(imageUrl|image_url|pdfUrl|avatarUrl|profileImage)\s*[:=]\s*req\."
context: "Media fetch from user URL - SSRF vector"
```

**Context Analysis Instructions:**
1. Identify all places where the application makes HTTP requests using user-supplied URLs.
2. Check if URL validation is performed (allowlist of domains/protocols).
3. Verify internal network addresses are blocked.
4. Check cloud metadata endpoint protection.
5. Look for webhook/callback URL features.

**Example Exploit Scenario:**
A profile picture feature allows users to provide an image URL. An attacker sets their URL to the cloud metadata endpoint and receives AWS IAM credentials. With these credentials, the attacker accesses the S3 bucket.

**Fix Guide:**
- Implement URL allowlist.
- Block internal network ranges and cloud metadata endpoints.
- Only allow http and https protocols.
- Resolve DNS and validate IP before making the request.
- Use a dedicated egress proxy for outbound requests.
