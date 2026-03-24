# Web Frontend Security Rules

Security patterns for React, Vue, and Angular web applications.

---

# React (10 Patterns)

## 1. Dangerous InnerHTML

- **Name:** react-dangerous-innerhtml
- **Regex:** `(dangerouslySetInnerHTML\s*=\s*\{\s*\{?\s*__html\s*:)`
- **Severity:** critical
- **Description:** Detects usage of React's dangerous innerHTML API which bypasses XSS protection by injecting raw HTML into the DOM. If the HTML source is user-controlled, this is a direct XSS vector.
- **FP Indicators:**
  - HTML is from a trusted CMS with server-side sanitization
  - Content is sanitized with DOMPurify or similar before injection
  - Static/hardcoded HTML strings only
  - Located in markdown renderer with sanitization pipeline
- **Fix:** Sanitize HTML with DOMPurify before injection. Prefer React components over raw HTML. If rendering markdown, use a safe renderer like `react-markdown`.

---

## 2. localStorage/sessionStorage Secrets

- **Name:** react-localstorage-secrets
- **Regex:** `((localStorage|sessionStorage)\.(setItem|getItem)\s*\(\s*["\'].*(token|jwt|auth|secret|password|session|credential|api.?key)["\'])`
- **Severity:** high
- **Description:** Detects sensitive data stored in localStorage/sessionStorage, which is accessible to any JavaScript running on the page (XSS attack surface).
- **FP Indicators:**
  - Storing non-sensitive UI state or feature flags
  - Token is a CSRF token (designed to be in DOM)
  - Located in test/mock files
- **Fix:** Use HTTP-only secure cookies for auth tokens. If localStorage is required, store only short-lived, non-critical tokens. Implement token rotation and consider using a BFF (Backend For Frontend) pattern.

---

## 3. React Ref DOM Manipulation

- **Name:** react-ref-dom-manipulation
- **Regex:** `(ref\.current\.innerHTML\s*=|useRef\s*\(.*\.innerHTML|createRef\s*\(.*\.innerHTML|\.current\.outerHTML\s*=)`
- **Severity:** high
- **Description:** Detects direct DOM manipulation via React refs that sets innerHTML, bypassing React's virtual DOM and XSS protections.
- **FP Indicators:**
  - Content is sanitized before assignment
  - Setting innerHTML to empty string for cleanup
  - Third-party library integration requiring DOM access
- **Fix:** Use React state and JSX for rendering instead of direct DOM manipulation. If direct DOM access is needed, sanitize content with DOMPurify.

---

## 4. Unsafe Target Blank

- **Name:** react-unsafe-target-blank
- **Regex:** `(target\s*=\s*["\']_blank["\'](?!.*rel\s*=\s*["\'].*noopener))`
- **Severity:** medium
- **Description:** Detects links with `target="_blank"` that lack `rel="noopener noreferrer"`, potentially allowing the opened page to access `window.opener` and redirect the parent page.
- **FP Indicators:**
  - Modern browsers (Chrome 88+) implicitly set `noopener` for `target="_blank"`
  - Links to same-origin pages
  - React 16.9+ handles this for anchor tags
- **Fix:** Always add `rel="noopener noreferrer"` to links with `target="_blank"`.

---

## 5. Unvalidated URL Redirect

- **Name:** react-open-redirect
- **Regex:** `(window\.location\s*=\s*.*\b(params|query|search|hash|props|state)\b|window\.location\.href\s*=\s*(?!["\'](\/|https))|navigate\s*\(\s*.*\b(params|query|searchParams)\b|redirect\s*\(\s*.*req\.(query|params|body))`
- **Severity:** high
- **Description:** Detects redirects using unvalidated user input (URL params, query strings, props), which can be exploited for phishing via open redirect attacks.
- **FP Indicators:**
  - URL is validated against a whitelist before redirect
  - Redirect target is a relative path (same origin)
  - Located in OAuth callback with proper state validation
- **Fix:** Validate redirect URLs against a whitelist of allowed domains. Use relative paths instead of full URLs. Parse the URL and verify the hostname before redirecting.

---

## 6. Insecure PostMessage

- **Name:** react-insecure-postmessage
- **Regex:** `(postMessage\s*\([^,]+,\s*["\']\*["\']|addEventListener\s*\(\s*["\']message["\'].*(?!.*origin))`
- **Severity:** high
- **Description:** Detects postMessage with wildcard origin or message event listeners that do not verify the sender's origin, allowing cross-origin message injection.
- **FP Indicators:**
  - postMessage to same-origin iframe
  - Origin is checked in the event handler body
  - Development-only cross-origin communication
- **Fix:** Always specify the target origin in postMessage. Always verify origin in message handlers: `if (event.origin !== "https://trusted.com") return;`.

---

## 7. Missing CSRF Protection

- **Name:** react-csrf-missing
- **Regex:** `(fetch\s*\(\s*["\'].*api.*["\'],\s*\{[^}]*method\s*:\s*["\'](POST|PUT|DELETE|PATCH)["\'](?![\s\S]*csrf|[\s\S]*xsrf|[\s\S]*X-CSRF))`
- **Severity:** high
- **Description:** Detects mutation requests (POST/PUT/DELETE/PATCH) that may not include CSRF tokens, making the application vulnerable to cross-site request forgery.
- **FP Indicators:**
  - Using SameSite cookies with Lax/Strict policy
  - CSRF token is added globally via axios interceptor or fetch wrapper
  - API uses token-based auth (Bearer tokens in headers)
  - Located in test files
- **Fix:** Include CSRF tokens in request headers or body. Use a global request interceptor to add tokens. Configure cookies with SameSite=Strict or SameSite=Lax.

---

## 8. Sensitive Data in React State URL

- **Name:** react-state-url-sensitive
- **Regex:** `(useSearchParams\s*\(.*\b(token|password|secret|ssn|credit)\b|URLSearchParams.*\b(token|password|secret)\b|router\.(query|params)\.\b(token|password|secret)\b)`
- **Severity:** high
- **Description:** Detects sensitive data passed through URL parameters or search params, which are logged in browser history, server logs, and referrer headers.
- **FP Indicators:**
  - Token is a short-lived, single-use token (email verification)
  - Located in password reset flow with proper expiration
  - Test code with mock URL params
- **Fix:** Send sensitive data via POST body or HTTP headers, not URL parameters. Use short-lived tokens in URLs only when necessary (email links) and invalidate after use.

---

## 9. Prototype Pollution in Props

- **Name:** react-prototype-pollution
- **Regex:** `(Object\.assign\s*\(\s*\{\},\s*.*props|\.\.\.props\s*(?!.*children)|spread\s*\(\s*.*props|\{.*\.\.\.(?!children).*\}\s*(?=/>))`
- **Severity:** medium
- **Description:** Detects indiscriminate spreading of props/objects that may enable prototype pollution if the source is user-controlled.
- **FP Indicators:**
  - Props are typed/validated with TypeScript or PropTypes
  - Spread is from a trusted internal component
  - Only spreading known safe properties
- **Fix:** Destructure only the props you need instead of spreading all. Use TypeScript strict types. Validate and sanitize props from external sources.

---

## 10. Insecure iframe Configuration

- **Name:** react-insecure-iframe
- **Regex:** `(<iframe[^>]*(?!.*sandbox)|<iframe[^>]*sandbox\s*=\s*["\']["\']|<iframe[^>]*src\s*=\s*\{.*\b(props|state|params|query)\b)`
- **Severity:** high
- **Description:** Detects iframes without sandbox attributes or with dynamic user-controlled src URLs, which can lead to clickjacking or malicious content injection.
- **FP Indicators:**
  - iframe loads trusted first-party content
  - Sandbox attribute is set with appropriate restrictions
  - Static/hardcoded src URL to a known safe origin
- **Fix:** Always add `sandbox` attribute with minimal permissions. Validate iframe src URLs against a whitelist. Add X-Frame-Options headers for your own pages.

---

# Vue (5 Patterns)

## 11. v-html Directive

- **Name:** vue-v-html
- **Regex:** `(v-html\s*=\s*["\']|v-html\s*=\s*["\'][^"\']*\b(user|input|query|param|data)\b)`
- **Severity:** critical
- **Description:** Detects usage of Vue's `v-html` directive which renders raw HTML and is the primary XSS vector in Vue applications.
- **FP Indicators:**
  - Content is sanitized with DOMPurify before binding
  - HTML comes from a trusted CMS with server-side sanitization
  - Static/hardcoded HTML content
- **Fix:** Sanitize content before using v-html with DOMPurify. Prefer `v-text` or template interpolation `{{ }}` which auto-escapes HTML.

---

## 12. Vue Unvalidated URL Binding

- **Name:** vue-unvalidated-url
- **Regex:** `(:href\s*=\s*["\'].*\b(user|input|query|param)\b|:src\s*=\s*["\'].*\b(user|input|query|param)\b|v-bind:href\s*=|v-bind:src\s*=.*\b(user|input|query)\b)`
- **Severity:** high
- **Description:** Detects dynamic URL bindings in Vue templates that may use unvalidated user input, enabling XSS via javascript: URLs or open redirect attacks.
- **FP Indicators:**
  - URL is validated/sanitized before binding
  - Binding uses a computed property with validation
  - Static or known-safe URL patterns
- **Fix:** Validate URLs before binding. Ensure URLs start with https:// or /. Block javascript: protocol. Use a URL validation library.

---

## 13. Vue Insecure Template Compilation

- **Name:** vue-template-compilation
- **Regex:** `(Vue\.compile\s*\(|template\s*:\s*.*\b(user|input|query|data)\b|\$options\.template\s*=|render\s*:\s*.*new\s+Function)`
- **Severity:** critical
- **Description:** Detects runtime template compilation with potentially user-controlled input, which is equivalent to code injection in Vue applications.
- **FP Indicators:**
  - Template string is hardcoded
  - Using pre-compiled SFC templates (default Vue workflow)
  - Build-time compilation only
- **Fix:** Never compile templates from user input. Use pre-compiled Single File Components (SFC). If dynamic content is needed, use slots and props instead of runtime template compilation.

---

## 14. Vue Router Guard Bypass

- **Name:** vue-router-guard-bypass
- **Regex:** `(router\.push\s*\(.*(?!.*beforeEach)|beforeEach\s*\(\s*\(\s*to\s*,\s*from\s*,\s*next\s*\)\s*=>\s*\{\s*next\s*\(\s*\)\s*\}|\$router\.replace\s*\(.*(?!.*auth))`
- **Severity:** high
- **Description:** Detects Vue Router navigation guards that always call next() without authentication checks, or navigation that bypasses guards.
- **FP Indicators:**
  - Route is intentionally public (login page, landing page)
  - Auth check is performed in a different guard or middleware
  - Located in test/mock router configuration
- **Fix:** Implement proper auth checks in beforeEach: verify token validity, check user roles. Use next(false) or next("/login") for unauthorized access. Use route meta fields for role-based access control.

---

## 15. Vuex State Exposure

- **Name:** vue-vuex-state-exposure
- **Regex:** `(vuex-persistedstate|createPersistedState|vuex-persist|localStorage\.(setItem|getItem).*store\.(state|getters).*\b(token|secret|password|auth)\b)`
- **Severity:** high
- **Description:** Detects Vuex state persistence plugins or manual persistence of sensitive state to localStorage, which is vulnerable to XSS-based theft.
- **FP Indicators:**
  - Only non-sensitive state is persisted (UI preferences, cart)
  - Sensitive fields are excluded via plugin filter
  - Using httpOnly cookies for auth instead
- **Fix:** Exclude sensitive state from persistence via plugin configuration. Use HTTP-only cookies for auth tokens. Clear persisted state on logout.

---

# Angular (5 Patterns)

## 16. Bypass Security Trust

- **Name:** angular-bypass-security
- **Regex:** `(bypassSecurityTrust(Html|Script|Style|Url|ResourceUrl)\s*\()`
- **Severity:** critical
- **Description:** Detects Angular's DomSanitizer bypass methods which disable built-in XSS protection. If the input is user-controlled, this creates a direct XSS vulnerability.
- **FP Indicators:**
  - Input is pre-sanitized with a custom sanitizer
  - Content is from a trusted source (CMS with server-side sanitization)
  - Static/hardcoded content only
  - Used for trusted SVG icons or embed URLs
- **Fix:** Avoid bypass methods whenever possible. If needed, sanitize input first with DOMPurify. Create a custom pipe that sanitizes then trusts.

---

## 17. Angular innerHTML Binding

- **Name:** angular-innerhtml-binding
- **Regex:** `(\[innerHTML\]\s*=\s*["\']|\.nativeElement\.innerHTML\s*=|ElementRef.*innerHTML|Renderer2.*setProperty.*innerHTML)`
- **Severity:** high
- **Description:** Detects innerHTML binding in Angular templates or direct DOM manipulation that may bypass Angular's sanitization.
- **FP Indicators:**
  - Angular's built-in sanitizer handles innerHTML binding (partial protection)
  - Content is additionally sanitized with DOMPurify
  - Using Renderer2 with sanitized content
- **Fix:** Use Angular's template interpolation which auto-escapes. For HTML content, Angular's innerHTML applies basic sanitization but may not catch all vectors. Add DOMPurify for defense in depth.

---

## 18. Angular HTTP Interceptor Missing

- **Name:** angular-http-interceptor
- **Regex:** `(HttpClient\.(post|put|delete|patch)\s*\((?![\s\S]*interceptor|[\s\S]*withCredentials|[\s\S]*Authorization)|provideHttpClient\s*\((?!.*withInterceptors))`
- **Severity:** medium
- **Description:** Detects HTTP mutation requests that may lack proper interceptors for auth headers, CSRF tokens, or error handling.
- **FP Indicators:**
  - Interceptor is registered globally in app.module or app.config
  - Using provideHttpClient with withInterceptors in standalone apps
  - Public API calls that intentionally skip auth
- **Fix:** Register a global HTTP interceptor for auth and CSRF. Handle errors consistently in an error interceptor.

---

## 19. Angular CORS Misconfiguration

- **Name:** angular-cors-proxy
- **Regex:** `(proxy\.conf\.(json|js).*"\/api".*"target"|"changeOrigin"\s*:\s*true|proxyConfig.*"secure"\s*:\s*false)`
- **Severity:** medium
- **Description:** Detects Angular proxy configurations that may mask CORS issues in development but fail in production, or disable SSL verification.
- **FP Indicators:**
  - Proxy is development-only (angular.json serve target)
  - Production uses same-origin API or proper CORS headers
  - Located in development-specific configuration
- **Fix:** Ensure production API has proper CORS headers. Do not rely on dev proxy for production. Set secure: true in proxy config. Configure proper CORS on the backend.

---

## 20. Angular CSP Nonce Missing

- **Name:** angular-csp-nonce
- **Regex:** `(ngCspNonce|CSP_NONCE|content-security-policy|Content-Security-Policy)`
- **Severity:** medium
- **Description:** Checks for Content Security Policy nonce configuration in Angular apps. Proper CSP prevents inline script injection attacks.
- **FP Indicators:**
  - This is a presence check - finding it is good
  - CSP is configured at the server/CDN level, not in Angular code
  - Using hash-based CSP instead of nonces
- **Fix:** Configure CSP headers on your web server. For Angular, use ngCspNonce attribute or CSP_NONCE injection token. Use ng build with --subresource-integrity flag.
