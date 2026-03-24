# OWASP API Security Top 10 (2023) - API Security

Reference framework for deep security audit of APIs (REST, GraphQL, gRPC).

---

## API1 - Broken Object Level Authorization (BOLA)

**CWE References:** CWE-284, CWE-285, CWE-639, CWE-862, CWE-863

**Grep Patterns:**
```
# Direct object reference via URL params
pattern: "params\.(id|userId|orderId|accountId|documentId|resourceId)"
context: "Object accessed by ID - check if ownership is validated"

# Database query without ownership check
pattern: "(findById|findOne|findByPk|get_object_or_404|objects\.get)\s*\(\s*(req\.|request\.|params|args)"
context: "Object fetched by ID without ownership filter"

# GraphQL object access
pattern: "(Query|Mutation).*?(id|ID|Id)\s*:"
context: "GraphQL resolver accessing object by ID"

# REST endpoint with resource ID
pattern: "(get|post|put|patch|delete)\s*\(\s*['\"]\/[^'\"]*\/:id|\/[^'\"]*\/:.*Id"
context: "REST endpoint with resource ID parameter"

# Missing ownership filter
pattern: "(findOne|findById|findByPk|get)\s*\(\s*\{?\s*(?!.*userId|.*user_id|.*owner|.*createdBy)"
context: "Query without ownership check in WHERE clause"

# UUID vs sequential ID
pattern: "(autoIncrement|serial|SERIAL|IDENTITY|AUTO_INCREMENT)"
context: "Sequential IDs are enumerable - consider UUIDs"
```

**Context Analysis Instructions:**
1. For every endpoint that accesses a specific resource by ID, verify the query includes an ownership check.
2. Check that the user from the auth token is used to filter results.
3. Verify GraphQL resolvers validate authorization for each resolved object.
4. Check if resource IDs are sequential (easily guessable) or UUIDs.
5. Verify batch/list endpoints filter by the authenticated user.

**Example Exploit Scenario:**
A REST API endpoint `GET /api/v1/orders/:orderId` fetches orders by ID without ownership check. An authenticated user iterates through IDs and accesses other users' orders including personal information and payment details. The server never checks if the order belongs to the requesting user.

**Fix Guide:**
- Always include ownership check in queries: `Order.findOne({ where: { id: orderId, userId: req.user.id } })`.
- Implement authorization middleware that verifies resource ownership.
- Use UUIDs instead of sequential IDs to prevent enumeration.
- Implement a centralized authorization layer (e.g., CASL, Casbin, OPA).

---

## API2 - Broken Authentication

**CWE References:** CWE-204, CWE-287, CWE-306, CWE-307, CWE-521, CWE-613, CWE-798

**Grep Patterns:**
```
# Missing auth on endpoints
pattern: "(router|app)\.(get|post|put|patch|delete)\s*\([^)]*(?!auth|protect|guard|middleware)"
context: "Endpoint without authentication middleware"

# Weak JWT configuration
pattern: "(HS256|algorithm.*none|expiresIn.*['\"]30d|expiresIn.*['\"]365|verify\s*:\s*false)"
context: "Weak JWT configuration"

# API key only authentication
pattern: "(x-api-key|apiKey|api_key)\s*[:=]"
context: "API key as sole authentication - insufficient for user context"

# Token in query string
pattern: "(token|key|auth|session)\s*=.*?\?(.*?&|$)"
context: "Authentication token in URL query string"

# Missing rate limit on auth
pattern: "(login|authenticate|token|oauth)\s*"
context: "Auth endpoint - check for rate limiting"

# OAuth misconfiguration
pattern: "(redirect_uri|callback_url|state|PKCE|code_challenge)"
context: "OAuth flow - check state parameter and redirect validation"
```

**Context Analysis Instructions:**
1. List all API endpoints and verify each has appropriate authentication.
2. Check JWT configuration: algorithm (RS256 preferred), expiration (short-lived), audience, issuer.
3. Verify token refresh mechanism exists with rotation.
4. Check for rate limiting on authentication endpoints.
5. Verify OAuth flows use state parameter and PKCE.

**Example Exploit Scenario:**
An API uses JWT with HS256 and a weak secret. An attacker brute-forces the JWT secret, then crafts tokens for any user. The login endpoint has no rate limiting, allowing credential stuffing at thousands of requests per second. Refresh tokens never expire, so a single stolen token provides permanent access.

**Fix Guide:**
- Use RS256 or ES256 for JWT signing with properly managed key pairs.
- Set short JWT expiration (15 minutes) with refresh token rotation.
- Implement rate limiting on all auth endpoints: 5 attempts per minute.
- Use OAuth 2.0 with PKCE for public clients; always validate state parameter.

---

## API3 - Broken Object Property Level Authorization

**CWE References:** CWE-213, CWE-915

**Grep Patterns:**
```
# Mass assignment
pattern: "(Object\.assign|spread.*req\.body|\.\.\.(req\.body|request\.body)|\.update\(req\.body|\.create\(req\.body|\*\*request\.(data|POST))"
context: "Mass assignment - all request properties passed to model"

# Over-exposed fields
pattern: "(toJSON|serialize|select\s*\(\s*['\"]?\*|findAll|find\(\s*\{?\s*\}?\s*\))"
context: "All fields returned - check for sensitive field exposure"

# Missing field allowlist
pattern: "(\.update\(|\.create\(|\.save\()(?!.*\{.*only|.*pick|.*whitelist|.*allowedFields)"
context: "Create/update without field allowlist"

# GraphQL over-fetching
pattern: "(type\s+User|type\s+Account|type\s+Profile).*?(password|secret|hash|token|salt|ssn|creditCard)"
context: "GraphQL type exposes sensitive fields"

# Admin fields exposed
pattern: "(isAdmin|role|permissions|internal|private)\s*:"
context: "Administrative field in API response"
```

**Context Analysis Instructions:**
1. Check all API responses for over-exposed fields (password hashes, internal IDs, admin flags).
2. Verify create/update endpoints use field allowlists, not the entire request body.
3. Check GraphQL types for sensitive fields that should not be queryable.
4. Verify admin/internal fields cannot be set via API (mass assignment).

**Example Exploit Scenario:**
A user profile update endpoint passes the full request body to the ORM. An attacker sends `{"name": "Hacker", "role": "admin", "isVerified": true}` and escalates their privileges because the role and isVerified fields are not filtered. The GET endpoint returns all user fields including password hash and salt.

**Fix Guide:**
- Use explicit field allowlists for create/update.
- Create DTOs or serializers that explicitly define response fields.
- Use `select` or `exclude` on queries to filter sensitive fields.
- For GraphQL, use field-level authorization directives.

---

## API4 - Unrestricted Resource Consumption

**CWE References:** CWE-307, CWE-770, CWE-799, CWE-400

**Grep Patterns:**
```
# Missing rate limiting
pattern: "(express-rate-limit|rateLimit|throttle|RateLimiter|slowDown)"
context: "Check if rate limiting is implemented"

# Missing pagination
pattern: "(findAll|find\(\s*\{?\s*\}|SELECT\s+\*|\.list\(|\.getAll\()"
context: "List endpoint without pagination/limit"

# Large file upload
pattern: "(multer|formidable|busboy|upload|multipart)"
context: "File upload - check size limits"

# Missing request size limit
pattern: "(body.*parser|bodyParser|json\(\)|urlencoded)"
context: "Request body parser - check size limit"

# GraphQL complexity
pattern: "(graphql|apolloServer|makeExecutableSchema)"
context: "GraphQL - check query depth/complexity limits"

# Missing timeout
pattern: "(axios|fetch|request|http\.get|urllib)"
context: "HTTP request without timeout configuration"

# Regex DoS (ReDoS)
pattern: "(RegExp\s*\(.*req\.|RegExp\(.*user|regex.*input)"
context: "Regex from user input - ReDoS risk"
```

**Context Analysis Instructions:**
1. Verify rate limiting is applied globally and per-endpoint for sensitive operations.
2. Check all list endpoints have pagination with maximum page size.
3. Verify file upload endpoints have size limits.
4. Check request body size limits are configured.
5. For GraphQL, verify query depth and complexity limits are set.

**Example Exploit Scenario:**
A GraphQL API has no query depth limit. An attacker sends a deeply nested query 50 levels deep, causing the server to exhaust memory. The REST list endpoint returns all records without pagination. No rate limiting exists, so the attacker repeats these requests in parallel.

**Fix Guide:**
- Implement rate limiting: `express-rate-limit` with sensible defaults.
- Add pagination to all list endpoints with max limit (100).
- Set request body size limits: `app.use(express.json({ limit: '1mb' }))`.
- Set file upload size limits.
- For GraphQL: set query depth limit (10), complexity limit (1000).
- Add timeouts to all HTTP client calls.

---

## API5 - Broken Function Level Authorization (BFLA)

**CWE References:** CWE-285, CWE-862, CWE-863

**Grep Patterns:**
```
# Admin endpoints
pattern: "(\/admin|\/internal|\/management|\/config|\/system|\/debug|\/actuator|\/swagger)"
context: "Administrative endpoint - verify role-based access"

# Role-based access
pattern: "(role|permission|isAdmin|is_admin|isStaff|is_staff|isSuperUser|hasRole)"
context: "Role check - verify server-side enforcement"

# HTTP method authorization
pattern: "(app|router)\.(get|post|put|patch|delete)\s*\("
context: "Check if different HTTP methods have different authorization"

# Function elevation
pattern: "(promote|elevate|grant|assign.*role|change.*role|set.*admin)"
context: "Privilege elevation function - verify authorization"

# API versioning bypass
pattern: "(\/v1\/|\/v2\/|\/api\/|\/internal\/)"
context: "Check if older API versions have weaker authorization"
```

**Context Analysis Instructions:**
1. Map all API endpoints and their required authorization levels.
2. Verify admin/management endpoints have role-based access control middleware.
3. Check that different HTTP methods have appropriate authorization.
4. Verify internal/debug endpoints are not accessible in production.
5. Check for missing authorization on state-changing operations (POST, PUT, DELETE).

**Example Exploit Scenario:**
An API has GET /api/v1/users (requires auth) and DELETE /api/v1/users/:id (no auth check). A regular user discovers the admin panel endpoint POST /api/admin/users/promote which has no role validation. Additionally, the older v1 API has no authorization middleware while v2 is properly protected.

**Fix Guide:**
- Implement role-based middleware for all route groups.
- Deny by default: require explicit authorization middleware on every endpoint.
- Disable/remove debug and internal endpoints in production.
- Ensure all API versions have equivalent authorization.

---

## API6 - Unrestricted Access to Sensitive Business Flows

**CWE References:** CWE-799, CWE-840, CWE-841

**Grep Patterns:**
```
# Purchase/payment flow
pattern: "(purchase|checkout|payment|buy|order|subscribe|redeem|coupon)"
context: "Business flow - check for automation abuse protection"

# Registration/signup
pattern: "(register|signup|sign_up|createAccount|create_account)"
context: "Registration flow - check for bot protection"

# Voting/rating
pattern: "(vote|rate|review|like|upvote|downvote|favorite)"
context: "Voting/rating - check for abuse prevention"

# CAPTCHA/bot protection
pattern: "(captcha|recaptcha|hCaptcha|turnstile|bot.*detect|honeypot)"
context: "Bot protection mechanism"

# Coupon/referral
pattern: "(coupon|promo|referral|discount|reward|bonus|voucher)"
context: "Promotional flow - check for abuse prevention"
```

**Context Analysis Instructions:**
1. Identify sensitive business flows (purchase, registration, voting, booking).
2. Check if CAPTCHA or bot protection is implemented for these flows.
3. Verify per-user rate limiting exists for business operations.
4. Verify promotional/coupon flows have abuse prevention.

**Example Exploit Scenario:**
An e-commerce API has a flash sale endpoint with no bot protection. An attacker scripts 1000 purchase requests per second, buying out the inventory before legitimate users can access it. A referral system gives credit per referral with no limit, enabling fraudulent credit generation.

**Fix Guide:**
- Implement CAPTCHA for sensitive business flows.
- Add per-user rate limiting for business operations.
- Implement device fingerprinting for high-value flows.
- Limit coupon/referral redemption per user and per IP.

---

## API7 - Server-Side Request Forgery (SSRF)

**CWE References:** CWE-918

**Grep Patterns:**
```
# URL from user input in API
pattern: "(fetch|axios|request|got|needle|http\.get|urllib|requests)\s*\(\s*(req\.|request\.|args\.|params\.|body\.)"
context: "HTTP request with user-supplied URL"

# Webhook URLs
pattern: "(webhook|callback|notify_url|hook_url|endpoint_url)\s*[:=]\s*(req\.|request\.|body\.)"
context: "Webhook URL from user input"

# PDF/image generation from URL
pattern: "(puppeteer|playwright|wkhtmltopdf|phantomjs|sharp|imagemagick|screenshot)"
context: "URL-to-image/PDF - SSRF vector"

# Cloud metadata protection
pattern: "(169\.254\.169\.254|metadata\.google|metadata\.azure|100\.100\.100\.200)"
context: "Cloud metadata endpoint - verify blocked"
```

**Context Analysis Instructions:**
1. Identify all places the API makes HTTP requests using user-supplied URLs.
2. Check webhook/callback URL features for SSRF protection.
3. Verify URL validation includes protocol restriction.
4. Check that internal network addresses and cloud metadata endpoints are blocked.

**Example Exploit Scenario:**
An API import endpoint accepts a URL to fetch data from. The server fetches the URL, and an attacker provides an internal service address, triggering internal admin actions. A webhook registration endpoint allows registering the cloud metadata URL, receiving AWS credentials when the webhook fires.

**Fix Guide:**
- Implement URL allowlist: only allow specific, trusted domains.
- Block internal IP ranges and cloud metadata endpoints.
- Restrict protocols to http and https only.
- Resolve DNS and validate IP address before making the request.
- For webhooks: validate URL on creation, re-validate before each call.

---

## API8 - Security Misconfiguration

**CWE References:** CWE-2, CWE-16, CWE-209, CWE-319, CWE-388, CWE-444, CWE-489, CWE-942

**Grep Patterns:**
```
# Missing security headers
pattern: "(helmet|X-Content-Type-Options|X-Frame-Options|Content-Security-Policy|Strict-Transport-Security)"
context: "Check if security headers are configured"

# CORS misconfiguration
pattern: "(Access-Control-Allow-Origin|cors\(\)|origin:\s*(true|\*))"
context: "CORS configuration - check for overly permissive settings"

# Error detail exposure
pattern: "(stack|stackTrace|err\.message|error\.message|traceback|DEBUG\s*=\s*True)"
context: "Error details exposed to clients"

# API documentation exposed
pattern: "(swagger|openapi|api-docs|graphql-playground|graphiql|altair)"
context: "API documentation/playground in production"

# Missing request validation
pattern: "(express\.json\(\)|bodyParser\.json\(\))"
context: "Check if request schema validation is implemented"

# Verbose headers
pattern: "(X-Powered-By|Server:|x-aspnet-version)"
context: "Server technology headers exposed"
```

**Context Analysis Instructions:**
1. Check security headers configuration.
2. Verify CORS is configured with specific origins.
3. Check error handling does not expose stack traces.
4. Verify API documentation/playground is disabled in production.
5. Verify server technology headers are removed.
6. Check for request schema validation (Joi, Zod, Pydantic).

**Example Exploit Scenario:**
An API in production has CORS set to reflect any origin with credentials enabled. An attacker creates a malicious website that makes authenticated cross-origin requests, stealing user data. Swagger UI is enabled at /api-docs, exposing all endpoints. Error responses include full stack traces revealing the database schema.

**Fix Guide:**
- Set specific CORS origins.
- Configure security headers using helmet.js or equivalent.
- Implement generic error responses in production.
- Remove API documentation in production.
- Remove `X-Powered-By` header.
- Implement request schema validation on all endpoints.

---

## API9 - Improper Inventory Management

**CWE References:** CWE-1059

**Grep Patterns:**
```
# API versioning
pattern: "(\/v[0-9]+\/|\/api\/v[0-9]+|version.*header)"
context: "API version - check if old versions are decommissioned"

# Debug/test endpoints
pattern: "(\/debug|\/test|\/staging|\/internal|\/dev|\/sandbox|\/mock)"
context: "Non-production endpoint - should not be in production"

# Health/status endpoints
pattern: "(\/health|\/status|\/ping|\/ready|\/alive|\/version|\/info)"
context: "Operational endpoint - check information disclosure"

# Deprecated endpoints
pattern: "(@deprecated|deprecated|DEPRECATED|TODO.*remove|FIXME.*remove)"
context: "Deprecated endpoint still in codebase"
```

**Context Analysis Instructions:**
1. List all API endpoints and verify each is documented.
2. Check for old API versions that are still accessible.
3. Verify debug/test/internal endpoints are not accessible in production.
4. Check health/status endpoints for information disclosure.
5. Look for deprecated endpoints that should be removed.

**Example Exploit Scenario:**
A company's API v1 has a known vulnerability fixed in v2, but v1 is still deployed. An attacker discovers the v1 endpoint through source maps and exploits it. A debug endpoint from development was accidentally deployed to production, exposing the entire user database without authentication.

**Fix Guide:**
- Maintain an API inventory document.
- Decommission old API versions: return 410 Gone or redirect.
- Remove debug/test/internal endpoints from production builds.
- Use API gateway to enforce routing and block undocumented paths.
- Minimize information in health/status endpoints.

---

## API10 - Unsafe Consumption of APIs

**CWE References:** CWE-20, CWE-73, CWE-285, CWE-295, CWE-319, CWE-602, CWE-829

**Grep Patterns:**
```
# Third-party API calls
pattern: "(fetch|axios|request|got|http\.get|requests\.get|HttpClient)\s*\(\s*['\"]https?://"
context: "Third-party API call - check input validation on response"

# Trusting external data
pattern: "(\.data|\.body|\.json\(\)|\.text\(\))\s*[;.]"
context: "External API response used - check if validated"

# Missing TLS verification
pattern: "(rejectUnauthorized\s*:\s*false|verify\s*=\s*False|InsecureSkipVerify)"
context: "TLS verification disabled for external API"

# Webhook data processing
pattern: "(webhook|hook|callback|notify)\s*.*?(req\.body|request\.body|payload)"
context: "Incoming webhook data - check validation"

# API response size
pattern: "(maxContentLength|maxBodyLength|timeout|max_size)"
context: "Check if API response size/timeout limits are set"
```

**Context Analysis Instructions:**
1. Identify all third-party API integrations.
2. Check if responses from external APIs are validated before use.
3. Verify TLS certificate validation is enabled for all external calls.
4. Check if external API data is sanitized before use in SQL, HTML, or commands.
5. Verify webhook data is validated and authenticated (HMAC signature).

**Example Exploit Scenario:**
An API integrates with a third-party shipping service. The shipping API returns tracking URLs that are rendered directly in the frontend without sanitization. An attacker compromises the shipping API and returns a malicious URL, executing script on all users who view their order status. The API also pipes the shipping service response directly into a SQL query for logging, enabling second-order injection.

**Fix Guide:**
- Validate and sanitize all data received from external APIs before use.
- Define schemas for expected API responses and validate against them.
- Enable TLS certificate verification for all external API calls.
- Set timeouts and response size limits on HTTP clients.
- Validate webhook signatures (HMAC) before processing webhook data.
- Use parameterized queries for any external data stored in databases.
- Sanitize external data before rendering.
