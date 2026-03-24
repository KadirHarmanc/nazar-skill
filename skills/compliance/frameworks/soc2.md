# SOC 2 Type II Code-Level Compliance Checks

Reference framework for code-level SOC 2 Type II Trust Services Criteria auditing.
Covers Security, Availability, Processing Integrity, Confidentiality, and Privacy.

---

## 1. Access Control (RBAC / Role Checks)

**What to Check:** Logical access controls must restrict access based on user roles and the principle of least privilege.

**Grep Patterns:**
```
# Role-based access
pattern: "(role|permission|isAdmin|is_admin|hasRole|hasPermission|authorize|can\(|ability|policy)"
context: "Role-based access control implementation"

# RBAC middleware
pattern: "(requireRole|checkRole|roleGuard|RoleMiddleware|PermissionGuard|authorize\()"
context: "Authorization middleware"

# RBAC configuration
pattern: "(roles|permissions|policies|abilities)\s*[:=]\s*(\{|\[)"
context: "Role/permission configuration"

# Service account access
pattern: "(service.*account|service.*key|machine.*token|api.*key.*role)"
context: "Service account - check least privilege"

# Admin access
pattern: "(\/admin|admin.*panel|admin.*dashboard|superuser|super_admin)"
context: "Admin access - check authorization"

# Privilege separation
pattern: "(read|write|delete|create|update|manage|view|edit|execute)\s*.*?(permission|role|access)"
context: "Granular permission levels"
```

**Pass Criteria:**
- RBAC system is implemented with defined roles and permissions
- Authorization middleware is applied to all protected routes
- Principle of least privilege: users have minimum necessary permissions
- Admin access requires additional verification
- Service accounts have scoped permissions
- Role assignments are audited

**Fail Indicators:**
- No RBAC implementation
- Authorization checks missing on routes
- All users have same access level
- Admin access without additional verification
- Overly permissive service account roles

**Fix Guide:**
- Implement RBAC with granular permissions: `{ role: 'editor', permissions: ['read', 'write', 'publish'] }`.
- Apply authorization middleware to all route groups.
- Implement principle of least privilege: start with no access, grant specific permissions.
- Require MFA for admin access.
- Use scoped API keys for service accounts.
- Log all role changes and access decisions.

---

## 2. Audit Logging (Who Did What When)

**What to Check:** All significant events must be logged with sufficient detail for forensic analysis: who, what, when, where, outcome.

**Grep Patterns:**
```
# Audit log implementation
pattern: "(audit.*log|auditLog|audit_log|activity.*log|event.*log|auditTrail|audit_trail)"
context: "Audit logging implementation"

# Structured logging
pattern: "(winston|bunyan|pino|log4j|loguru|structlog|morgan|logging\.getLogger)"
context: "Logging library - check structured format"

# Log fields
pattern: "(userId|user_id|action|resource|timestamp|ip|userAgent|result|status)"
context: "Audit log fields"

# Authentication events
pattern: "(login|logout|register|password.*change|mfa|token.*refresh)"
context: "Auth event - check if logged"

# Data access events
pattern: "(read|access|view|download|export)\s*.*?(log|audit|track|record)"
context: "Data access logging"

# Data modification events
pattern: "(create|update|delete|modify|change)\s*.*?(log|audit|track|record)"
context: "Data modification logging"

# Log storage
pattern: "(log.*transport|log.*destination|CloudWatch|Splunk|ELK|Datadog|logstash|fluentd)"
context: "Centralized log storage"

# Log immutability
pattern: "(append.*only|immutable|write.*once|log.*integrity|tamper)"
context: "Log integrity protection"
```

**Pass Criteria:**
- Audit logging is implemented for all security-relevant events
- Logs include: timestamp, userId, action, resource, IP, outcome
- Authentication events (login, logout, MFA, password change) are logged
- Data access and modification events are logged
- Logs are sent to centralized, immutable storage
- Log retention meets policy (minimum 1 year for SOC 2)

**Fail Indicators:**
- No audit logging implementation
- Logs missing critical fields (userId, action)
- Authentication events not logged
- Data access/modification not logged
- Logs stored only locally
- No log retention policy

**Fix Guide:**
- Implement structured audit logging: `auditLog({ userId, action, resource, ip, timestamp, result })`.
- Log all authentication events (success and failure).
- Log all data access and modifications (CRUD operations).
- Send logs to centralized immutable storage (CloudWatch, Splunk, ELK).
- Implement log retention: minimum 1 year for SOC 2 compliance.
- Protect log integrity: append-only storage, separate log service account.
- Include request metadata: IP, user agent, session ID.

---

## 3. Encryption (AES-256, TLS 1.2+)

**What to Check:** Data must be encrypted at rest and in transit using industry-standard algorithms.

**Grep Patterns:**
```
# TLS/HTTPS enforcement
pattern: "(https|TLS|ssl|HSTS|Strict-Transport|SECURE_SSL_REDIRECT|force.*https)"
context: "Transport encryption"

# TLS version
pattern: "(TLSv1\.[012]|SSLv[23]|minVersion|secureProtocol|tls\.DEFAULT_MIN_VERSION)"
context: "TLS version configuration"

# Encryption at rest
pattern: "(AES|aes-256|encrypt.*column|field.*encrypt|at.rest|pgcrypto|KMS|encryption.*key)"
context: "Encryption at rest"

# Weak encryption
pattern: "(DES|RC4|MD5|SHA1|ECB|Blowfish|3DES)"
context: "Weak encryption algorithm"

# Key management
pattern: "(KMS|key.*management|key.*rotation|key.*vault|HSM|master.*key)"
context: "Key management system"

# Database encryption
pattern: "(encrypt.*database|database.*encrypt|RDS.*encrypt|transparent.*data.*encryption|TDE)"
context: "Database-level encryption"

# Backup encryption
pattern: "(backup.*encrypt|encrypt.*backup|encrypted.*snapshot)"
context: "Backup encryption"
```

**Pass Criteria:**
- All connections use TLS 1.2 or higher
- HSTS header is configured
- AES-256 or equivalent used for data at rest
- Key management system is in place
- Database encryption is enabled
- Backups are encrypted
- No weak/deprecated algorithms used

**Fail Indicators:**
- TLS 1.0 or 1.1 allowed
- No HSTS header
- Data at rest not encrypted
- Weak algorithms (DES, RC4, MD5 for security purposes)
- No key management
- Unencrypted backups

**Fix Guide:**
- Enforce TLS 1.2 minimum: disable TLS 1.0, 1.1, SSLv3.
- Set HSTS header: `Strict-Transport-Security: max-age=31536000; includeSubDomains; preload`.
- Use AES-256-GCM for encryption at rest.
- Implement key management with AWS KMS, Azure Key Vault, or HashiCorp Vault.
- Enable database encryption (RDS encryption, PostgreSQL pgcrypto).
- Encrypt all backups and snapshots.
- Schedule key rotation (annually at minimum).

---

## 4. Secret Management (No Hardcoded Secrets)

**What to Check:** Secrets, credentials, and API keys must not be hardcoded. They must be managed through a secure secrets management system.

**Grep Patterns:**
```
# Hardcoded secrets
pattern: "(password|secret|api_key|apiKey|token|private_key|auth_token)\s*[:=]\s*['\"][^'\"]{8,}"
context: "Hardcoded secret"

# Environment variables
pattern: "(process\.env|os\.environ|ENV\[|getenv|dotenv|config\(\))"
context: "Environment variable usage - check if secrets come from env"

# Secret managers
pattern: "(AWS.*Secrets|SecretManager|Vault|KeyVault|SSM.*Parameter|doppler|infisical)"
context: "Secret management system"

# .env files
pattern: "(\.env|\.env\.local|\.env\.production|\.env\.development)"
context: "Environment file - check if in .gitignore"

# Git-committed secrets
pattern: "(\.env|credentials|secret|private.*key)\s"
context: "Check .gitignore for secret files"

# Docker secrets
pattern: "(docker.*secret|DOCKER.*PASSWORD|DOCKER.*TOKEN)"
context: "Docker secret management"

# CI/CD secrets
pattern: "(GITHUB_TOKEN|CI_TOKEN|DEPLOY_KEY|NPM_TOKEN|PYPI_TOKEN)"
context: "CI/CD secret - check if in environment, not code"

# Known API key patterns
pattern: "(sk-[a-zA-Z0-9]{32,}|AIza[0-9A-Za-z_-]{35}|AKIA[0-9A-Z]{16}|ghp_[a-zA-Z0-9]{36})"
context: "Known API key format detected in code"
```

**Pass Criteria:**
- No hardcoded secrets in source code
- Secrets loaded from environment variables or secret manager
- .env files are in .gitignore
- Secret management system is used (AWS Secrets Manager, Vault, etc.)
- Secrets are rotated regularly
- CI/CD uses encrypted secrets

**Fail Indicators:**
- Hardcoded secrets in source code
- .env files committed to git
- No secret management system
- Known API key patterns in code
- Secrets in Docker Compose files committed to git

**Fix Guide:**
- Move all secrets to environment variables or a secret manager.
- Add `.env*` to .gitignore immediately.
- Use a secret manager: AWS Secrets Manager, HashiCorp Vault, or Doppler.
- Implement secret rotation: at least annually, immediately on compromise.
- Use CI/CD encrypted secrets (GitHub Actions secrets, GitLab CI variables).
- Run secret scanning in CI/CD (git-secrets, truffleHog, detect-secrets).
- Rotate any secret that was ever committed to version control.

---

## 5. Health Check Endpoints

**What to Check:** System must have monitoring endpoints to verify availability and health status, but they must not expose sensitive information.

**Grep Patterns:**
```
# Health endpoints
pattern: "(\/health|\/status|\/ping|\/ready|\/alive|\/healthz|\/readyz|\/livez)"
context: "Health check endpoint"

# Health check implementation
pattern: "(healthCheck|health_check|statusCheck|readinessProbe|livenessProbe)"
context: "Health check logic"

# Dependency checks
pattern: "(database.*health|redis.*health|queue.*health|cache.*health|dependency.*check)"
context: "Dependency health verification"

# Information disclosure
pattern: "(version|uptime|hostname|memory|cpu|disk|env|config)\s*.*?(health|status)"
context: "Information in health endpoint - check for over-exposure"

# Monitoring integration
pattern: "(prometheus|grafana|datadog|newrelic|pingdom|uptimerobot|healthchecks\.io)"
context: "Monitoring system integration"

# Alerting
pattern: "(alert|notify|pagerduty|opsgenie|slack.*webhook|email.*alert)"
context: "Health alerting configuration"
```

**Pass Criteria:**
- Health check endpoints exist (/health, /ready)
- Health checks verify critical dependencies (database, cache, queue)
- Health endpoints do not expose sensitive information
- Monitoring system is configured to check health endpoints
- Alerting is configured for health check failures

**Fail Indicators:**
- No health check endpoints
- Health endpoints expose system details (version, hostname, config)
- No dependency health checks
- No monitoring or alerting

**Fix Guide:**
- Implement `/health` (basic), `/ready` (with dependency checks) endpoints.
- Check critical dependencies: database connectivity, cache availability, queue connection.
- Return minimal information: `{ "status": "ok" }` or `{ "status": "degraded", "checks": { "db": "ok", "cache": "failed" } }`.
- Never expose version, hostname, environment variables, or memory/CPU in public health endpoints.
- Configure monitoring to poll health endpoints every 30-60 seconds.
- Set up alerting for consecutive failures (PagerDuty, OpsGenie, Slack).

---

## 6. Rate Limiting

**What to Check:** API endpoints must be rate-limited to prevent abuse, DoS attacks, and brute-force attempts.

**Grep Patterns:**
```
# Rate limiting libraries
pattern: "(express-rate-limit|rate-limit|rateLimit|throttle|slowDown|RateLimiter|django-ratelimit|rack-throttle)"
context: "Rate limiting implementation"

# Rate limit configuration
pattern: "(windowMs|max|limit|interval|points|duration|burst|rpm|rps)"
context: "Rate limit configuration values"

# Rate limit headers
pattern: "(X-RateLimit|RateLimit-|Retry-After|X-Retry-After)"
context: "Rate limit response headers"

# Per-endpoint rate limiting
pattern: "(login|register|forgot|reset|otp|verify|upload|search).*?(rate|limit|throttle)"
context: "Endpoint-specific rate limiting"

# IP-based limiting
pattern: "(ip|IP|clientIp|client_ip|remoteAddress).*?(rate|limit|ban|block)"
context: "IP-based rate limiting"

# DDoS protection
pattern: "(cloudflare|akamai|fastly|AWS.*WAF|shield|ddos)"
context: "DDoS protection service"
```

**Pass Criteria:**
- Global rate limiting is configured
- Sensitive endpoints have stricter rate limits (login, registration, OTP)
- Rate limit headers are returned in responses
- IP-based and user-based rate limiting exist
- DDoS protection is in place

**Fail Indicators:**
- No rate limiting implementation
- Sensitive endpoints without rate limiting
- No DDoS protection

**Fix Guide:**
- Implement global rate limiting: 100 requests per minute per IP.
- Stricter limits for sensitive endpoints: login (5/min), registration (3/min), OTP (3/min).
- Return rate limit headers: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`.
- Implement both IP-based and user-based rate limiting.
- Use DDoS protection service (Cloudflare, AWS Shield, Akamai).
- Implement exponential backoff for authentication failures.
- Use distributed rate limiting for multi-instance deployments (Redis-based).

---

## 7. Input Validation

**What to Check:** All user input must be validated, sanitized, and constrained before processing.

**Grep Patterns:**
```
# Validation libraries
pattern: "(joi|yup|zod|class-validator|express-validator|cerberus|marshmallow|pydantic|dry-validation)"
context: "Input validation library"

# Schema validation
pattern: "(schema|validate|sanitize|validator|ValidationPipe|IsString|IsEmail|IsInt)"
context: "Input validation schema"

# Missing validation
pattern: "(req\.body\.|req\.query\.|req\.params\.|request\.data|request\.POST|request\.GET)"
context: "User input usage - check if validated"

# SQL injection prevention
pattern: "(parameterized|prepared.*statement|placeholder|\$[0-9]|sanitize)"
context: "SQL injection prevention"

# XSS prevention
pattern: "(escape|encode|sanitize|DOMPurify|xss|helmet|csp)"
context: "XSS prevention"

# File upload validation
pattern: "(mimetype|fileType|extension|size|fileFilter|upload.*limit)"
context: "File upload validation"
```

**Pass Criteria:**
- Input validation library is used
- All API endpoints validate input against schemas
- SQL injection prevention (parameterized queries)
- XSS prevention (output encoding, CSP)
- File upload validation (type, size, content)
- Error messages do not reveal validation logic

**Fail Indicators:**
- No input validation library
- Direct use of user input without validation
- String concatenation in SQL queries
- No file upload validation

**Fix Guide:**
- Use a validation library (Zod, Joi, Pydantic) on every endpoint.
- Define schemas for all request bodies, query parameters, and path parameters.
- Use parameterized queries for all database operations.
- Implement CSP header and output encoding for XSS prevention.
- Validate file uploads: check MIME type, extension, file size, and content.
- Return generic validation errors; do not reveal internal validation rules.

---

## 8. Error Handling (No Stack Traces to Users)

**What to Check:** Errors must be handled gracefully without exposing internal details to users.

**Grep Patterns:**
```
# Stack trace exposure
pattern: "(err\.stack|error\.stack|stackTrace|traceback|stack_trace)"
context: "Stack trace - check if exposed to users"

# Error handler
pattern: "(errorHandler|error.*middleware|exception.*handler|@ExceptionHandler|rescue_from)"
context: "Global error handler"

# Detailed error response
pattern: "(res\.(json|send)\s*\(.*?(err|error)\.(message|stack)|raise.*Exception|throw.*Error)"
context: "Error details in response"

# Debug mode
pattern: "(DEBUG\s*[:=]\s*[Tt]rue|debug.*mode|NODE_ENV.*development|FLASK_DEBUG)"
context: "Debug mode - check if disabled in production"

# Unhandled rejections
pattern: "(unhandledRejection|uncaughtException|process\.on\s*\(\s*['\"]uncaught)"
context: "Unhandled error catching"

# Error logging
pattern: "(catch|except|rescue).*?(console|logger|log)\.(error|warn)"
context: "Error logging in catch blocks"

# Empty catch blocks
pattern: "catch\s*\([^)]*\)\s*\{\s*\}"
context: "Empty catch - error swallowed"
```

**Pass Criteria:**
- Global error handler is implemented
- Production errors return generic messages (no stack traces)
- Errors are logged server-side with full details
- Unhandled rejections/exceptions are caught
- Debug mode is disabled in production
- No empty catch blocks

**Fail Indicators:**
- Stack traces returned to users
- No global error handler
- Debug mode enabled in production
- Empty catch blocks (swallowed errors)
- Unhandled rejection/exception crashes

**Fix Guide:**
- Implement global error handler that catches all errors.
- Return generic error messages in production: `{ "error": "An unexpected error occurred", "requestId": "abc123" }`.
- Log full error details server-side (stack trace, request context, user ID).
- Set `NODE_ENV=production` or equivalent in production.
- Handle unhandled rejections: `process.on('unhandledRejection', handler)`.
- Never have empty catch blocks; at minimum, log the error.
- Use error monitoring service (Sentry, Bugsnag, Datadog) for production error tracking.

---

## 9. Session Management

**What to Check:** Sessions must be managed securely with proper configuration, timeouts, and protection mechanisms.

**Grep Patterns:**
```
# Session configuration
pattern: "(session|cookie)\s*[:=]\s*\{|express-session|cookie-session|django.*session|rack.*session"
context: "Session configuration"

# Cookie flags
pattern: "(secure|httpOnly|sameSite|domain|path|maxAge|expires)\s*[:=]"
context: "Cookie security flags"

# Session storage
pattern: "(MemoryStore|connect-redis|connect-mongo|session.*store|SESSION_ENGINE)"
context: "Session storage backend"

# Session regeneration
pattern: "(regenerate|regenerateId|session_regenerate|cycle.*session)"
context: "Session regeneration after auth"

# Session invalidation
pattern: "(destroy|invalidate|logout|revoke|terminate)\s*.*?session"
context: "Session invalidation on logout"

# Concurrent sessions
pattern: "(concurrent.*session|active.*session|max.*session|session.*limit)"
context: "Concurrent session management"

# CSRF protection
pattern: "(csrf|CSRF|xsrf|XSRF|csrfToken|_token|authenticity_token)"
context: "CSRF protection"
```

**Pass Criteria:**
- Session cookies have `secure`, `httpOnly`, `sameSite` flags
- Session storage uses a proper backend (Redis, database), not memory
- Sessions are regenerated after authentication
- Sessions are invalidated on logout
- Session timeout is configured (idle and absolute)
- CSRF protection is implemented
- Concurrent session limits exist

**Fail Indicators:**
- Missing cookie security flags
- In-memory session storage in production
- Session not regenerated after login
- No session invalidation on logout
- No session timeout
- No CSRF protection

**Fix Guide:**
- Configure cookie flags: `{ secure: true, httpOnly: true, sameSite: 'strict', maxAge: 3600000 }`.
- Use Redis or database for session storage in production.
- Regenerate session after login: `req.session.regenerate()`.
- Destroy session on logout: `req.session.destroy()`.
- Set timeouts: idle (30 min), absolute (8 hours).
- Implement CSRF protection (csurf, Django CSRF, Rails authenticity token).
- Limit concurrent sessions per user (optional: notify on new login).

---

## 10. Change Management (Git Hooks, CI/CD)

**What to Check:** Code changes must go through a controlled process with review, testing, and automated checks before deployment.

**Grep Patterns:**
```
# Git hooks
pattern: "(pre-commit|pre-push|commit-msg|husky|lint-staged|lefthook)"
context: "Git hooks for pre-commit checks"

# CI/CD pipeline
pattern: "(\.github/workflows|\.gitlab-ci|Jenkinsfile|\.circleci|bitbucket-pipelines|azure-pipelines)"
context: "CI/CD pipeline configuration"

# Automated testing
pattern: "(test|spec|__tests__|jest|mocha|pytest|rspec|junit|vitest)"
context: "Automated test suite"

# Code review
pattern: "(CODEOWNERS|pull_request|merge_request|review.*required|branch.*protection)"
context: "Code review enforcement"

# Branch protection
pattern: "(protect.*branch|required.*review|required.*status|main.*protect)"
context: "Branch protection rules"

# Deployment approval
pattern: "(deploy.*approval|manual.*deploy|deploy.*gate|environment.*protect)"
context: "Deployment approval process"

# Security scanning in CI
pattern: "(snyk|sonarqube|semgrep|codeql|trivy|npm.*audit|safety|bandit|brakeman)"
context: "Security scanning in CI/CD"

# Infrastructure as Code
pattern: "(terraform|cloudformation|pulumi|ansible|chef|puppet)"
context: "Infrastructure as Code for controlled changes"
```

**Pass Criteria:**
- Git hooks enforce pre-commit checks (linting, formatting)
- CI/CD pipeline runs automated tests
- Code review is required before merge (CODEOWNERS, branch protection)
- Security scanning is included in CI/CD
- Branch protection is enabled on main/production branches
- Deployment requires approval for production
- Infrastructure changes are managed as code

**Fail Indicators:**
- No CI/CD pipeline
- No automated tests
- No code review requirement
- No branch protection
- Direct push to production allowed
- No security scanning in CI

**Fix Guide:**
- Set up CI/CD pipeline with automated testing on every PR.
- Configure branch protection: require reviews, require status checks, no force push.
- Add pre-commit hooks: linting, formatting, secret scanning (husky + lint-staged).
- Include security scanning in CI: `npm audit`, Snyk, SonarQube, or Semgrep.
- Require at least 1 code review approval before merge.
- Implement deployment gates: manual approval for production deployments.
- Use Infrastructure as Code (Terraform, CloudFormation) for environment changes.
- Maintain deployment history and rollback capability.
