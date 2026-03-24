# API Backend Security Rules

Security patterns for Django, FastAPI, and Express backend applications.

---

# Django (10 Patterns)

## 1. Django DEBUG Mode

- **Name:** django-debug-mode
- **Regex:** `(DEBUG\s*=\s*True(?!\s*if|\s*#.*production|\s*#.*prod))`
- **Severity:** critical
- **Description:** Detects Django DEBUG=True which exposes detailed error pages with stack traces, SQL queries, settings values, and environment variables to end users.
- **FP Indicators:**
  - Located in settings.dev.py or settings.local.py
  - Guarded by environment variable check: `DEBUG = os.environ.get("DEBUG") == "True"`
  - Comment indicates development-only usage
- **Fix:** Set `DEBUG = os.environ.get("DEBUG", "False") == "True"`. Ensure production deployments set the env var to False. Use separate settings files for development and production.

---

## 2. Django ALLOWED_HOSTS Empty

- **Name:** django-allowed-hosts
- **Regex:** `(ALLOWED_HOSTS\s*=\s*\[\s*\]|ALLOWED_HOSTS\s*=\s*\[\s*["\']\*["\']\s*\])`
- **Severity:** high
- **Description:** Detects empty or wildcard ALLOWED_HOSTS which disables Django's host header validation, enabling HTTP host header attacks and cache poisoning.
- **FP Indicators:**
  - Development-only settings file
  - Overridden by environment variable in production
  - Test settings
- **Fix:** Set `ALLOWED_HOSTS` to explicit list of valid domains: `ALLOWED_HOSTS = os.environ.get("ALLOWED_HOSTS", "").split(",")`. Never use `["*"]` in production.

---

## 3. Django SECRET_KEY Hardcoded

- **Name:** django-secret-key-hardcoded
- **Regex:** `(SECRET_KEY\s*=\s*["\'][a-zA-Z0-9!@#$%^&*()_+\-=]{20,}["\'])`
- **Severity:** critical
- **Description:** Detects hardcoded Django SECRET_KEY in settings files. This key is used for cryptographic signing of sessions, tokens, and password reset links.
- **FP Indicators:**
  - Value is clearly a placeholder (e.g., `"change-me-in-production"`)
  - Located in example/template settings file
  - Loaded from environment variable despite string format
- **Fix:** Load from environment: `SECRET_KEY = os.environ["SECRET_KEY"]`. Generate a strong key: `python -c "from django.core.management.utils import get_random_secret_key; print(get_random_secret_key())"`.

---

## 4. Django Raw SQL

- **Name:** django-raw-sql
- **Regex:** `(\.raw\s*\(\s*["\']|\.extra\s*\(\s*.*where\s*=|connection\.cursor\s*\(\s*\).*execute\s*\(\s*["\'].*%|RawSQL\s*\()`
- **Severity:** critical
- **Description:** Detects raw SQL queries in Django that bypass the ORM's automatic parameterization, creating SQL injection risk.
- **FP Indicators:**
  - Using parameterized queries: `.raw("SELECT * FROM t WHERE id=%s", [id])`
  - Located in data migration files with no user input
  - Performance-critical read-only queries with hardcoded parameters
- **Fix:** Use Django ORM queries instead of raw SQL. If raw SQL is needed, always use parameterized queries: `Model.objects.raw("SELECT * FROM t WHERE id=%s", [user_id])`. Never use string formatting.

---

## 5. Django csrf_exempt

- **Name:** django-csrf-exempt
- **Regex:** `(@csrf_exempt|csrf_exempt\s*\(|CsrfViewMiddleware.*REMOVE|CSRF_COOKIE_SECURE\s*=\s*False)`
- **Severity:** high
- **Description:** Detects views with CSRF protection disabled, making them vulnerable to cross-site request forgery attacks.
- **FP Indicators:**
  - API endpoint using token-based auth (DRF with JWT)
  - Webhook endpoint that verifies signatures
  - Health check or public read-only endpoint
- **Fix:** Remove csrf_exempt decorator. For API views, use Django REST Framework with proper authentication. For webhooks, verify request signatures instead of disabling CSRF.

---

## 6. Django Insecure Cookie Settings

- **Name:** django-insecure-cookies
- **Regex:** `(SESSION_COOKIE_SECURE\s*=\s*False|CSRF_COOKIE_SECURE\s*=\s*False|SESSION_COOKIE_HTTPONLY\s*=\s*False|SECURE_SSL_REDIRECT\s*=\s*False)`
- **Severity:** high
- **Description:** Detects insecure cookie and SSL settings that allow session hijacking, CSRF token theft, or downgrade attacks.
- **FP Indicators:**
  - Development-only settings
  - Behind a reverse proxy that handles SSL
  - Test configuration
- **Fix:** Set all security flags to True in production: `SESSION_COOKIE_SECURE=True`, `CSRF_COOKIE_SECURE=True`, `SESSION_COOKIE_HTTPONLY=True`, `SECURE_SSL_REDIRECT=True`.

---

## 7. Django Mass Assignment

- **Name:** django-mass-assignment
- **Regex:** `(\.create\s*\(\s*\*\*request\.(POST|data|body)|form\.save\s*\(\s*\).*(?!.*fields)|serializer\.save\s*\(\s*\*\*request\.data\b|ModelForm.*class\s+Meta.*fields\s*=\s*["\']__all__["\'])`
- **Severity:** high
- **Description:** Detects potential mass assignment vulnerabilities where user input is passed directly to model creation or update without field filtering.
- **FP Indicators:**
  - Serializer has explicit fields list
  - Form has explicit fields in Meta class
  - Request data is validated/filtered before assignment
- **Fix:** Always specify explicit fields in serializers and forms. Never use `fields = "__all__"`. Validate and whitelist fields before passing to create/update.

---

## 8. Django Insecure File Upload

- **Name:** django-insecure-upload
- **Regex:** `(FileField\s*\((?!.*upload_to)|ImageField\s*\((?!.*validators)|request\.FILES.*\.save\s*\(|\.write\s*\(\s*request\.FILES|MEDIA_ROOT.*MEDIA_URL(?!.*SECURE))`
- **Severity:** high
- **Description:** Detects file upload handling that may lack proper validation, allowing arbitrary file upload attacks.
- **FP Indicators:**
  - File validators are applied at the form/serializer level
  - Upload path is sanitized and outside web root
  - File type validation is done in a custom handler
- **Fix:** Validate file types with `FileExtensionValidator` and content-type checks. Set `upload_to` to a non-web-accessible directory. Limit file size in settings. Scan uploaded files for malware.

---

## 9. Django Verbose Error Pages

- **Name:** django-verbose-errors
- **Regex:** `(handler500\s*=.*(?!custom)|ADMINS\s*=\s*\[\s*\]|raise\s+Http404\s*\(.*\{|traceback\.format_exc\s*\(\s*\).*response)`
- **Severity:** medium
- **Description:** Detects configurations that may expose verbose error information to users, including stack traces and internal paths.
- **FP Indicators:**
  - Custom error handlers are defined
  - Error tracking service (Sentry) is configured
  - DEBUG is False (Django hides details automatically)
- **Fix:** Set DEBUG=False in production. Define custom error handlers: `handler404`, `handler500`. Use Sentry or similar for error tracking. Never expose tracebacks in API responses.

---

## 10. Django Insecure Password Validation

- **Name:** django-password-validation
- **Regex:** `(AUTH_PASSWORD_VALIDATORS\s*=\s*\[\s*\]|AUTH_PASSWORD_VALIDATORS\s*=\s*\[\s*\n?\s*\]|make_password\s*\(.*(?!.*hasher))`
- **Severity:** high
- **Description:** Detects empty or weak password validation configuration, allowing users to set trivially guessable passwords.
- **FP Indicators:**
  - Validators are configured in a separate settings file
  - Application does not use Django auth (external auth provider)
  - Test settings intentionally disable validation
- **Fix:** Configure all four built-in validators: UserAttributeSimilarityValidator, MinimumLengthValidator, CommonPasswordValidator, NumericPasswordValidator. Add custom validators as needed.

---

# FastAPI (8 Patterns)

## 11. FastAPI CORS Wildcard

- **Name:** fastapi-cors-wildcard
- **Regex:** `(CORSMiddleware.*allow_origins\s*=\s*\[\s*["\']\*["\']\s*\]|allow_origins\s*=\s*\[\s*["\']\*["\']\s*\]|allow_methods\s*=\s*\[\s*["\']\*["\']\s*\]|allow_credentials\s*=\s*True.*allow_origins\s*=\s*\[\s*["\']\*["\'])`
- **Severity:** high
- **Description:** Detects CORS middleware with wildcard origins, especially when combined with allow_credentials, which effectively disables the same-origin policy.
- **FP Indicators:**
  - Development-only configuration
  - Behind an API gateway that handles CORS
  - Internal-only API not exposed to browsers
- **Fix:** Set explicit allowed origins: `allow_origins=["https://myapp.com"]`. Never combine `allow_credentials=True` with wildcard origins. Use environment-based origin lists.

---

## 12. FastAPI Missing Rate Limiting

- **Name:** fastapi-rate-limit-missing
- **Regex:** `(slowapi|RateLimiter|rate_limit|throttle|Limiter)`
- **Severity:** medium
- **Description:** Checks for rate limiting implementation in FastAPI. Absence of rate limiting allows brute-force attacks and API abuse.
- **FP Indicators:**
  - This is a presence check - finding it is good
  - Rate limiting handled by API gateway/reverse proxy (nginx, CloudFlare)
  - Internal-only API with network-level restrictions
- **Fix:** Add `slowapi` for rate limiting: `limiter = Limiter(key_func=get_remote_address)`. Apply to sensitive endpoints: `@limiter.limit("5/minute")`. Consider stricter limits for auth endpoints.

---

## 13. FastAPI SQL Injection

- **Name:** fastapi-sql-injection
- **Regex:** `(\.execute\s*\(\s*f["\']|\.execute\s*\(\s*["\'].*\.format\s*\(|\.execute\s*\(\s*["\'].*%s.*["\']\s*%\s*[^,\)]|text\s*\(\s*f["\'])`
- **Severity:** critical
- **Description:** Detects SQL queries built with f-strings, format(), or % formatting, creating SQL injection vulnerabilities.
- **FP Indicators:**
  - Using SQLAlchemy ORM with model queries
  - Parameterized queries with `text()` and bound parameters
  - Table/column names (not user input) in f-strings
- **Fix:** Use SQLAlchemy ORM or parameterized queries: `db.execute(text("SELECT * FROM users WHERE id=:id"), {"id": user_id})`. Never interpolate user input into SQL strings.

---

## 14. FastAPI Unvalidated Path Parameters

- **Name:** fastapi-unvalidated-params
- **Regex:** `(@app\.(get|post|put|delete|patch)\s*\(\s*["\'].*\{.*\}.*["\'](?![\s\S]*Path\s*\())`
- **Severity:** medium
- **Description:** Detects route handlers with path parameters that may lack Pydantic/Path validation, allowing unexpected input types or values.
- **FP Indicators:**
  - Parameters are typed in function signature (FastAPI auto-validates types)
  - Custom dependency injection validates parameters
  - Simple integer IDs that FastAPI coerces automatically
- **Fix:** Use `Path()` with validation: `user_id: int = Path(..., gt=0, le=999999)`. Use Pydantic models for complex input validation. Add regex patterns for string parameters.

---

## 15. FastAPI Insecure JWT

- **Name:** fastapi-insecure-jwt
- **Regex:** `(jwt\.decode\s*\(.*algorithms\s*=\s*\[\s*["\'](none|HS256)["\']|jwt\.decode\s*\(.*verify\s*=\s*False|jwt\.decode\s*\(.*options\s*=\s*\{[^}]*verify_signature[^}]*False)`
- **Severity:** critical
- **Description:** Detects insecure JWT configurations: accepting "none" algorithm, disabled verification, or weak algorithms without proper key management.
- **FP Indicators:**
  - HS256 with a strong secret (context-dependent)
  - Verification disabled in test code only
  - Token decoded for claims inspection only (non-auth use)
- **Fix:** Use RS256 or ES256 with proper key pairs. Always verify signatures: `jwt.decode(token, key, algorithms=["RS256"])`. Never accept "none" algorithm. Use `python-jose` or `PyJWT` with secure defaults.

---

## 16. FastAPI Debug/Docs in Production

- **Name:** fastapi-debug-docs
- **Regex:** `(FastAPI\s*\(\s*(?!.*docs_url\s*=\s*None)(?!.*redoc_url\s*=\s*None)|app\s*=\s*FastAPI\s*\(\s*\)(?!\s*.*docs_url))`
- **Severity:** medium
- **Description:** Detects FastAPI instances with Swagger/ReDoc documentation endpoints enabled, which may expose API structure in production.
- **FP Indicators:**
  - Docs are protected by authentication middleware
  - Internal-only API where docs are needed
  - Docs URL is conditionally set based on environment
- **Fix:** Disable docs in production: `FastAPI(docs_url=None, redoc_url=None)` or conditionally: `docs_url="/docs" if settings.DEBUG else None`.

---

## 17. FastAPI Missing HTTPS Redirect

- **Name:** fastapi-https-redirect
- **Regex:** `(HTTPSRedirectMiddleware|TrustedHostMiddleware|X-Forwarded-Proto)`
- **Severity:** medium
- **Description:** Checks for HTTPS redirect and trusted host middleware. Absence may allow HTTP access or host header attacks.
- **FP Indicators:**
  - This is a presence check - finding it is good
  - HTTPS handled by reverse proxy or load balancer
  - Internal API not exposed to internet
- **Fix:** Add middleware: `app.add_middleware(HTTPSRedirectMiddleware)` and `app.add_middleware(TrustedHostMiddleware, allowed_hosts=["myapp.com"])`. Or configure at the reverse proxy level.

---

## 18. FastAPI Dependency Injection Bypass

- **Name:** fastapi-dependency-bypass
- **Regex:** `(@app\.(post|put|delete|patch)\s*\((?!.*dependencies|.*Depends).*\n\s*(?:async\s+)?def\s+\w+\s*\((?!.*Depends\s*\(\s*\w*(auth|permission|security|verify|validate)))`
- **Severity:** high
- **Description:** Detects mutation endpoints that may not use dependency injection for authentication or authorization checks.
- **FP Indicators:**
  - Auth dependency is applied at router level
  - Global middleware handles authentication
  - Public endpoint intentionally without auth
- **Fix:** Use dependency injection for auth: `@app.post("/", dependencies=[Depends(require_auth)])` or `current_user: User = Depends(get_current_user)`. Apply auth at router level for consistent protection.

---

# Express (10 Patterns)

## 19. Express Helmet Missing

- **Name:** express-helmet-missing
- **Regex:** `(helmet\s*\(|app\.use\s*\(\s*helmet)`
- **Severity:** high
- **Description:** Checks for Helmet.js middleware which sets critical security headers (CSP, X-Frame-Options, HSTS, etc.). Absence leaves the app without HTTP security headers.
- **FP Indicators:**
  - This is a presence check - finding it is good
  - Security headers set manually or by reverse proxy
  - Headers configured at CDN/load balancer level
- **Fix:** Install and use Helmet: `app.use(helmet())`. Customize CSP as needed: `app.use(helmet({ contentSecurityPolicy: { directives: { ... } } }))`.

---

## 20. Express CORS Wildcard

- **Name:** express-cors-wildcard
- **Regex:** `(cors\s*\(\s*\)|cors\s*\(\s*\{[^}]*origin\s*:\s*(true|["\']\*["\'])|res\.setHeader\s*\(\s*["\']Access-Control-Allow-Origin["\'],\s*["\']\*["\'])`
- **Severity:** high
- **Description:** Detects CORS middleware with wildcard or overly permissive origin configuration, potentially allowing any website to make authenticated requests.
- **FP Indicators:**
  - Development-only configuration
  - Public API intended for any origin (read-only, no credentials)
  - CORS handled by reverse proxy
- **Fix:** Set explicit origins: `cors({ origin: ["https://myapp.com"], credentials: true })`. Use a validation function for dynamic origins: `origin: (origin, callback) => { ... }`.

---

## 21. Express Rate Limit Missing

- **Name:** express-rate-limit-missing
- **Regex:** `(express-rate-limit|rateLimit|RateLimiterMemory|RateLimiterRedis|rate-limiter-flexible)`
- **Severity:** high
- **Description:** Checks for rate limiting implementation. Absence allows brute-force attacks, credential stuffing, and API abuse.
- **FP Indicators:**
  - This is a presence check - finding it is good
  - Rate limiting at API gateway/reverse proxy level
  - Internal service not exposed to internet
- **Fix:** Add express-rate-limit: `const limiter = rateLimit({ windowMs: 15 * 60 * 1000, max: 100 })`. Apply stricter limits on auth endpoints. Use Redis for distributed rate limiting.

---

## 22. Express Prototype Pollution

- **Name:** express-prototype-pollution
- **Regex:** `(Object\.assign\s*\(\s*\{\},\s*req\.(body|query|params)|merge\s*\(\s*.*req\.(body|query)|deepMerge\s*\(.*req\.(body|query)|\{.*\.\.\.req\.(body|query|params)\s*\})`
- **Severity:** critical
- **Description:** Detects merging/spreading of request data into objects, which can enable prototype pollution attacks via `__proto__`, `constructor`, or `prototype` keys.
- **FP Indicators:**
  - Input is validated/sanitized to strip dangerous keys
  - Using a safe merge library that blocks prototype keys
  - Pydantic or Joi schema validation before merge
- **Fix:** Validate input with Joi/Zod schemas that strip unknown keys. Use `Object.create(null)` for dictionaries. Sanitize input: `delete req.body.__proto__; delete req.body.constructor`. Use `helmet` and `hpp` middleware.

---

## 23. Express NoSQL Injection

- **Name:** express-nosql-injection
- **Regex:** `(\.find\s*\(\s*req\.(body|query|params)|\.findOne\s*\(\s*req\.(body|query)|\.updateOne\s*\(\s*req\.(body|query)|\.deleteOne\s*\(\s*\{.*req\.(body|query)|\.aggregate\s*\(\s*req\.(body|query))`
- **Severity:** critical
- **Description:** Detects MongoDB/NoSQL queries using request data directly, enabling NoSQL injection via operators like `$gt`, `$ne`, `$regex`.
- **FP Indicators:**
  - Input is validated/sanitized before query
  - Using mongoose with schema validation
  - Specific fields extracted from request (not the whole object)
- **Fix:** Never pass req.body directly to queries. Extract and validate specific fields. Use `mongo-sanitize` to strip `$` operators: `sanitize(req.body)`. Validate with Joi/Zod schemas.

---

## 24. Express Session Misconfiguration

- **Name:** express-session-misconfig
- **Regex:** `(session\s*\(\s*\{[^}]*secret\s*:\s*["\'][^"\']{1,20}["\']|session\s*\(\s*\{(?![\s\S]*secure[\s\S]*true)|cookie\s*:\s*\{(?![\s\S]*httpOnly[\s\S]*true)|resave\s*:\s*true)`
- **Severity:** high
- **Description:** Detects express-session with weak secrets, missing secure/httpOnly flags, or resave enabled, making sessions vulnerable to theft or fixation.
- **FP Indicators:**
  - Development-only configuration
  - Running behind HTTPS proxy that adds secure flag
  - Cookie settings defined separately
- **Fix:** Use a strong random secret from environment variable. Set `cookie: { secure: true, httpOnly: true, sameSite: "strict", maxAge: ... }`. Set `resave: false` and `saveUninitialized: false`. Use Redis/DB session store.

---

## 25. Express SQL Injection

- **Name:** express-sql-injection
- **Regex:** `(\.query\s*\(\s*["\'].*\$\{|\.query\s*\(\s*["\'].*\+\s*req\.(body|query|params)|\.query\s*\(\s*`[^`]*\$\{.*req\.|connection\.execute\s*\(\s*`[^`]*\$\{)`
- **Severity:** critical
- **Description:** Detects SQL queries constructed with template literals or concatenation using request data, creating SQL injection vulnerabilities.
- **FP Indicators:**
  - Using parameterized queries with placeholder markers
  - Table/column names from trusted config (not user input)
  - ORM (Sequelize, Knex, Prisma) query builder
- **Fix:** Use parameterized queries: `db.query("SELECT * FROM users WHERE id = ?", [req.params.id])`. Use an ORM like Sequelize, Knex, or Prisma. Never concatenate user input into SQL.

---

## 26. Express File Upload Unrestricted

- **Name:** express-file-upload
- **Regex:** `(multer\s*\(\s*\{(?![\s\S]*fileFilter|[\s\S]*limits)|upload\.(single|array|fields)\s*\((?![\s\S]*fileFilter)|express-fileupload(?![\s\S]*limits))`
- **Severity:** high
- **Description:** Detects file upload configuration without file type filtering or size limits, allowing arbitrary file upload attacks.
- **FP Indicators:**
  - File filter is defined separately and applied
  - Validation happens in route handler after upload
  - Internal tool with trusted users only
- **Fix:** Configure multer with fileFilter and limits: `multer({ limits: { fileSize: 5 * 1024 * 1024 }, fileFilter: (req, file, cb) => { /* validate type */ } })`. Validate MIME types and file extensions.

---

## 27. Express Error Leaking

- **Name:** express-error-leaking
- **Regex:** `(res\.(send|json)\s*\(\s*(err|error)\s*\)|res\.(send|json)\s*\(\s*\{\s*error\s*:\s*(err|error)\.(message|stack)\s*\}|app\.use\s*\(.*err.*res\.status\s*\(\s*500\s*\)\.send\s*\(\s*err)`
- **Severity:** medium
- **Description:** Detects error handlers that send raw error objects or stack traces to clients, potentially exposing internal paths and implementation details.
- **FP Indicators:**
  - Error message is generic/custom (not the raw error)
  - Stack trace only sent in development mode
  - Error is sanitized before sending
- **Fix:** Send generic error messages in production: `res.status(500).json({ error: "Internal server error" })`. Log full errors server-side. Use error handling middleware that differentiates environments.

---

## 28. Express Missing Input Validation

- **Name:** express-input-validation
- **Regex:** `(express-validator|celebrate|Joi\.object|zod\.object|yup\.object|ajv\.compile|class-validator)`
- **Severity:** medium
- **Description:** Checks for input validation library usage. Absence means request data is likely used without validation, enabling various injection attacks.
- **FP Indicators:**
  - This is a presence check - finding it is good
  - Validation is done with custom middleware
  - TypeScript types provide compile-time validation (but not runtime)
- **Fix:** Add input validation with express-validator, Joi, Zod, or Yup. Validate all request body, query params, and path params. Use schemas that whitelist allowed fields and types.
