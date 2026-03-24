# PCI-DSS v4.0 Code-Level Compliance Checks

Reference framework for code-level PCI-DSS v4.0 compliance auditing.
For applications that process, store, or transmit cardholder data.

---

## 1. Content Security Policy (CSP) Headers

**What to Check:** CSP headers must be configured to prevent XSS and data injection attacks that could lead to cardholder data theft.

**Grep Patterns:**
```
# CSP header configuration
pattern: "(Content-Security-Policy|contentSecurityPolicy|csp|CSP)"
context: "CSP header implementation"

# Helmet CSP
pattern: "(helmet.*contentSecurityPolicy|helmet\.csp|helmet\(\))"
context: "Helmet CSP configuration"

# Inline scripts allowed
pattern: "(unsafe-inline|unsafe-eval|script-src.*\*)"
context: "Unsafe CSP directive"

# Report-only mode
pattern: "(Content-Security-Policy-Report-Only|reportOnly)"
context: "CSP in report-only mode - not enforcing"

# CSP directives
pattern: "(default-src|script-src|style-src|img-src|connect-src|font-src|frame-src|object-src)"
context: "CSP directive configuration"

# Meta tag CSP
pattern: "<meta.*Content-Security-Policy"
context: "CSP via meta tag (less secure than header)"
```

**Pass Criteria:**
- CSP header is set (not just report-only)
- `unsafe-inline` is not used for script-src
- `unsafe-eval` is not used
- `default-src 'self'` as base policy
- Report URI is configured for CSP violations
- No wildcard `*` in sensitive directives

**Fail Indicators:**
- No CSP header configured
- CSP in report-only mode only
- `unsafe-inline` or `unsafe-eval` in script-src
- Wildcard `*` in default-src or script-src

**Fix Guide:**
- Configure CSP header: `Content-Security-Policy: default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data:; connect-src 'self' https://api.example.com`.
- Use nonce or hash for inline scripts instead of `unsafe-inline`.
- Set CSP report URI: `report-uri /csp-report; report-to csp-endpoint`.
- Start with report-only mode to identify issues, then switch to enforcing.
- Review and tighten directives quarterly.

---

## 2. Subresource Integrity (SRI)

**What to Check:** External scripts and stylesheets must include SRI hashes to prevent tampering.

**Grep Patterns:**
```
# External scripts without SRI
pattern: "<script[^>]*src=['\"]https?://[^>]*(?!.*integrity)"
context: "External script without SRI hash"

# External stylesheets without SRI
pattern: "<link[^>]*href=['\"]https?://[^>]*\.css[^>]*(?!.*integrity)"
context: "External stylesheet without SRI hash"

# SRI present
pattern: "integrity=['\"]sha(256|384|512)-[A-Za-z0-9+/=]+"
context: "SRI hash present"

# CDN usage
pattern: "(cdn\.jsdelivr|cdnjs\.cloudflare|unpkg\.com|cdn\.bootcss|stackpath\.bootstrapcdn)"
context: "CDN resource - must have SRI"

# Crossorigin attribute
pattern: "crossorigin=['\"]anonymous['\"]"
context: "Crossorigin attribute for SRI"

# Dynamic script loading
pattern: "(createElement.*script|appendChild.*script|document\.write.*script|loadScript)"
context: "Dynamic script loading - check if SRI is applied"
```

**Pass Criteria:**
- All external scripts have `integrity` attribute with SHA-384 or SHA-512 hash
- All external stylesheets have `integrity` attribute
- `crossorigin="anonymous"` is set alongside integrity
- Dynamic script loading also applies SRI checks

**Fail Indicators:**
- External scripts loaded without SRI
- External stylesheets loaded without SRI
- Missing crossorigin attribute with SRI
- CDN resources without integrity verification

**Fix Guide:**
- Add SRI to all external resources: `<script src="https://cdn.example.com/lib.js" integrity="sha384-HASH" crossorigin="anonymous"></script>`.
- Generate SRI hashes: `openssl dgst -sha384 -binary file.js | openssl base64 -A`.
- Use SRI Hash Generator tools or CDN-provided hashes.
- For dynamic script loading, verify hash before execution.
- Monitor for SRI failures in CSP reports.

---

## 3. Card Data Protection (PAN Masking)

**What to Check:** Primary Account Numbers (PAN) must never be stored in plaintext. Display must be masked (first 6 / last 4 digits only).

**Grep Patterns:**
```
# Credit card number patterns
pattern: "(cardNumber|card_number|creditCard|credit_card|pan|PAN|accountNumber|primaryAccountNumber)"
context: "Card number field - check masking and encryption"

# Card data storage
pattern: "(save.*card|store.*card|persist.*card|insert.*card|card.*database|card.*table)"
context: "Card data storage - verify encryption or tokenization"

# Card number regex
pattern: "\d{4}[\s-]?\d{4}[\s-]?\d{4}[\s-]?\d{4}|[0-9]{13,19}"
context: "Possible card number pattern in code"

# Card masking
pattern: "(mask.*card|card.*mask|truncate.*card|xxxx|\\*{4,}|\\.{4,})"
context: "Card masking implementation"

# CVV/CVC storage
pattern: "(cvv|cvc|cvv2|cvc2|securityCode|security_code|cardCode)"
context: "CVV must NEVER be stored after authorization"

# Card data in logs
pattern: "(log|logger|console|print).*?(card|pan|cardNumber|creditCard)"
context: "Card data in logs - PCI violation"

# Tokenization
pattern: "(token.*card|card.*token|vault|payment.*token|stripe.*token|braintree.*token)"
context: "Card tokenization implementation"

# Card data in URL
pattern: "(card|pan|creditCard|cvv)\s*=.*?[?&]"
context: "Card data in URL - critical PCI violation"
```

**Pass Criteria:**
- PAN is never stored in plaintext; tokenization or strong encryption is used
- PAN display is masked: `4111 XXXX XXXX 1234` (first 6, last 4)
- CVV/CVC is never stored after authorization
- Card data is never logged
- Card data is never in URLs
- Payment processor tokenization is used (Stripe, Braintree, etc.)

**Fail Indicators:**
- Plaintext PAN storage
- Full PAN displayed
- CVV stored after authorization
- Card data in logs
- Card data in URL parameters
- Custom card processing instead of PCI-compliant processor

**Fix Guide:**
- Use a PCI-compliant payment processor (Stripe, Braintree, Adyen) for card handling.
- Never store raw PAN; use tokenization: `stripe.paymentMethods.create()`.
- Mask PAN for display: show only first 6 and last 4 digits.
- Never store CVV/CVC after authorization; do not even log it.
- Remove card data from all log statements.
- Never pass card data in URL parameters.
- If PAN must be stored, use AES-256 encryption with key management (KMS).
- Implement data discovery scanning to detect unprotected PAN in databases.

---

## 4. Multi-Factor Authentication (MFA)

**What to Check:** MFA must be implemented for access to cardholder data environments and administrative functions.

**Grep Patterns:**
```
# MFA implementation
pattern: "(mfa|MFA|twoFactor|two_factor|2fa|2FA|totp|TOTP|authenticator)"
context: "MFA implementation"

# OTP generation
pattern: "(otp|OTP|one.*time.*password|speakeasy|pyotp|otpauth|otplib)"
context: "OTP generation library"

# MFA enrollment
pattern: "(enroll.*mfa|setup.*mfa|enable.*2fa|register.*totp|mfa.*setup)"
context: "MFA enrollment flow"

# MFA verification
pattern: "(verify.*mfa|validate.*otp|check.*totp|mfa.*verify|twoFactor.*verify)"
context: "MFA verification"

# MFA on admin
pattern: "(admin|management|dashboard|panel).*?(mfa|2fa|twoFactor)"
context: "MFA requirement for admin access"

# Backup codes
pattern: "(backup.*code|recovery.*code|emergency.*code)"
context: "MFA backup/recovery codes"

# MFA bypass
pattern: "(skip.*mfa|bypass.*2fa|disable.*mfa|mfa.*false|twoFactor.*false)"
context: "MFA bypass - check if legitimate"

# SMS OTP (less secure)
pattern: "(sms.*otp|otp.*sms|text.*message.*code|phone.*verification)"
context: "SMS-based OTP - less secure than TOTP"
```

**Pass Criteria:**
- MFA is implemented (TOTP preferred over SMS)
- MFA is required for admin/CDE access
- MFA enrollment flow exists
- Backup/recovery codes are provided
- MFA cannot be bypassed
- MFA is required for remote access

**Fail Indicators:**
- No MFA implementation
- MFA only for some users, not admins/CDE access
- MFA bypass possible
- Only SMS-based OTP (no TOTP option)
- No backup codes

**Fix Guide:**
- Implement TOTP-based MFA using `speakeasy` (Node.js), `pyotp` (Python), or `devise-two-factor` (Rails).
- Require MFA for all admin access and CDE access.
- Generate backup codes during enrollment (10 single-use codes).
- Store MFA secrets securely (encrypted, not plaintext).
- Prefer TOTP over SMS OTP (SMS is vulnerable to SIM swapping).
- Implement MFA for remote access (VPN, SSH).
- Do not allow MFA bypass; provide recovery through backup codes and support process.

---

## 5. TLS 1.2+ Enforcement

**What to Check:** All network communication must use TLS 1.2 or higher. TLS 1.0 and 1.1 must be disabled.

**Grep Patterns:**
```
# TLS configuration
pattern: "(tls|TLS|ssl|SSL).*?(version|protocol|minVersion|min_version|ciphers)"
context: "TLS version configuration"

# Weak TLS versions
pattern: "(TLSv1\b|TLSv1\.0|TLSv1\.1|SSLv2|SSLv3|ssl3|tls1\b)"
context: "Weak TLS version - must disable"

# HTTPS enforcement
pattern: "(https|HTTPS|HSTS|Strict-Transport|redirect.*https|force.*ssl)"
context: "HTTPS enforcement"

# Certificate configuration
pattern: "(cert|certificate|ca|key|pfx|pem).*?(path|file|load|read)"
context: "TLS certificate configuration"

# Cipher suites
pattern: "(cipherSuites|cipher_suites|CIPHER|ciphers|ECDHE|DHE|AES.*GCM)"
context: "Cipher suite configuration"

# HTTP downgrade
pattern: "(http://(?!localhost)|downgrade|allowHTTP|insecure.*redirect)"
context: "Possible HTTP downgrade"

# Database TLS
pattern: "(database|postgres|mysql|mongo|redis).*?(ssl|tls|sslmode|tls.*true)"
context: "Database connection TLS"
```

**Pass Criteria:**
- TLS 1.2 minimum enforced for all connections
- TLS 1.0 and 1.1 explicitly disabled
- Strong cipher suites configured (ECDHE, AES-GCM)
- HSTS header with long max-age
- Database connections use TLS
- No HTTP downgrade paths

**Fail Indicators:**
- TLS 1.0 or 1.1 allowed
- Weak cipher suites (RC4, DES, 3DES)
- No HSTS header
- Database connections without TLS
- HTTP connections allowed

**Fix Guide:**
- Set minimum TLS version: `tls.createServer({ minVersion: 'TLSv1.2' })`.
- Configure strong cipher suites: ECDHE-RSA-AES256-GCM-SHA384, ECDHE-RSA-AES128-GCM-SHA256.
- Set HSTS: `Strict-Transport-Security: max-age=63072000; includeSubDomains; preload`.
- Enable TLS for database connections: `sslmode=require` (PostgreSQL), `ssl: { rejectUnauthorized: true }` (Node.js).
- Redirect all HTTP to HTTPS.
- Test with SSL Labs (ssllabs.com/ssltest) for configuration validation.
- Disable SSLv2, SSLv3, TLS 1.0, TLS 1.1 explicitly.

---

## 6. Key Management

**What to Check:** Cryptographic keys must be properly generated, stored, rotated, and retired.

**Grep Patterns:**
```
# Key management system
pattern: "(KMS|KeyVault|key.*management|key.*store|HSM|key.*vault|aws.*kms|azure.*keyvault)"
context: "Key management system"

# Key storage
pattern: "(master.*key|encryption.*key|private.*key|signing.*key)\s*[:=]"
context: "Cryptographic key storage location"

# Key rotation
pattern: "(key.*rotation|rotate.*key|key.*version|key.*expire|key.*lifecycle)"
context: "Key rotation mechanism"

# Key generation
pattern: "(generateKey|generate_key|createKey|create_key|randomBytes|SecureRandom)"
context: "Key generation method"

# Hardcoded keys
pattern: "(key|KEY|secret|SECRET)\s*[:=]\s*['\"][A-Za-z0-9+/=]{32,}"
context: "Hardcoded cryptographic key"

# Key access logging
pattern: "(key.*access|access.*key|key.*audit|key.*log)"
context: "Key access audit logging"

# Key escrow/custody
pattern: "(key.*backup|key.*recovery|key.*escrow|key.*custod|split.*key)"
context: "Key backup and recovery procedure"
```

**Pass Criteria:**
- Key management system is used (AWS KMS, Azure Key Vault, HSM)
- Keys are not hardcoded in source code
- Key rotation is automated (at least annually)
- Key generation uses cryptographically secure methods
- Key access is logged and audited
- Key retirement process exists
- Split knowledge / dual control for critical keys

**Fail Indicators:**
- Hardcoded encryption keys
- No key management system
- No key rotation
- Insecure key generation (Math.random, etc.)
- No key access logging

**Fix Guide:**
- Use AWS KMS, Azure Key Vault, or HashiCorp Vault for key management.
- Implement automated key rotation: annual for encryption keys, more frequent for high-risk.
- Generate keys with cryptographically secure methods: `crypto.randomBytes(32)`.
- Never store keys in source code; reference by key ID.
- Log all key access: creation, usage, rotation, deletion.
- Implement split knowledge: require two authorized individuals for critical key operations.
- Define key lifecycle: generation, distribution, storage, rotation, retirement, destruction.

---

## 7. Access Logging for Cardholder Data

**What to Check:** All access to cardholder data must be logged with sufficient detail for forensic analysis.

**Grep Patterns:**
```
# Audit logging for payment/card operations
pattern: "(payment|card|transaction|charge|refund|void)\s*.*?(log|audit|track|record)"
context: "Payment operation audit logging"

# Access logging
pattern: "(access.*log|log.*access|audit.*trail|audit.*log)\s*.*?(card|payment|pan|transaction)"
context: "Cardholder data access logging"

# Log fields
pattern: "(userId|user_id|timestamp|action|resource|ip|outcome|cardId|transactionId)"
context: "Audit log fields for CDE access"

# Unauthorized access logging
pattern: "(unauthorized|forbidden|denied|failed).*?(log|audit|alert)"
context: "Failed access attempt logging"

# Log protection
pattern: "(log.*integrity|log.*tamper|log.*immutable|append.*only|write.*once)"
context: "Log integrity protection"

# Log retention
pattern: "(log.*retention|retain.*log|log.*archive|log.*expire|365|one.*year)"
context: "Log retention period (PCI requires 1 year, 3 months immediately available)"

# Centralized logging
pattern: "(CloudWatch|Splunk|ELK|Datadog|Sumo.*Logic|logstash|fluentd|syslog)"
context: "Centralized log management"
```

**Pass Criteria:**
- All CDE access is logged (read, write, delete)
- Logs include: timestamp, userId, action, resource, IP, outcome
- Failed access attempts are logged and alerted
- Logs are centralized and tamper-proof
- Log retention: 1 year minimum, 3 months immediately available
- Log review is performed regularly (daily for CDE)

**Fail Indicators:**
- CDE access not logged
- Incomplete log records
- Failed access not logged
- Logs stored locally only
- No log retention policy
- No regular log review

**Fix Guide:**
- Implement comprehensive CDE audit logging: every read, write, delete of cardholder data.
- Include required fields: timestamp, user, action, resource, IP, outcome (success/failure).
- Log all failed access attempts and alert on patterns.
- Send logs to centralized, tamper-proof storage (Splunk, CloudWatch with integrity).
- Configure retention: 1 year total, 3 months immediately accessible.
- Implement daily log review for CDE access anomalies.
- Protect log integrity: separate log service account, append-only storage.

---

## 8. Network Segmentation Indicators

**What to Check:** Cardholder data environment (CDE) must be segmented from other networks and systems.

**Grep Patterns:**
```
# Network configuration
pattern: "(VPC|subnet|CIDR|firewall|security.*group|network.*policy|NetworkPolicy)"
context: "Network configuration for segmentation"

# Microservice isolation
pattern: "(docker.*network|kubernetes.*namespace|service.*mesh|istio|linkerd|envoy)"
context: "Microservice network isolation"

# Database access restriction
pattern: "(allowed.*hosts|bind.*address|listen.*address|pg_hba|bind-address)"
context: "Database access restriction"

# Firewall rules
pattern: "(iptables|ufw|nftables|security.*group.*rule|ingress|egress)"
context: "Firewall rules"

# API gateway
pattern: "(api.*gateway|kong|nginx.*proxy|traefik|envoy|AWS.*API.*Gateway)"
context: "API gateway for access control"

# Zero trust
pattern: "(zero.*trust|mutual.*tls|mTLS|service.*mesh|workload.*identity)"
context: "Zero trust network architecture"

# CDE isolation
pattern: "(cde|cardholder|payment.*service|payment.*gateway)\s*.*?(isolated|segmented|separated)"
context: "CDE isolation references"
```

**Pass Criteria:**
- CDE is in a separate network segment (VPC, subnet, namespace)
- Firewall rules restrict traffic to CDE (only necessary ports/services)
- Database access is restricted to CDE application servers
- API gateway controls access to payment services
- Service-to-service communication is authenticated (mTLS preferred)
- Network policies enforce segmentation in container environments

**Fail Indicators:**
- CDE on same network as other services
- No firewall rules for CDE
- Database accessible from outside CDE
- No network policies in container environments
- No API gateway for payment services

**Fix Guide:**
- Place CDE in a dedicated VPC/subnet with restricted security groups.
- Implement firewall rules: allow only necessary traffic to/from CDE.
- Restrict database access to CDE application servers only.
- Use Kubernetes NetworkPolicy to isolate payment namespace.
- Implement mTLS for service-to-service communication in CDE.
- Use API gateway as single entry point to CDE.
- Regularly test segmentation controls with penetration testing.

---

## 9. Vulnerability Scanning Indicators

**What to Check:** Regular vulnerability scanning must be performed on all systems in CDE scope.

**Grep Patterns:**
```
# Dependency scanning
pattern: "(npm.*audit|pip.*audit|snyk|dependabot|renovate|safety|bundle.*audit|trivy)"
context: "Dependency vulnerability scanning"

# SAST (Static Analysis)
pattern: "(sonarqube|semgrep|codeql|bandit|brakeman|eslint-plugin-security|spotbugs)"
context: "Static application security testing"

# DAST (Dynamic Analysis)
pattern: "(zap|OWASP.*ZAP|burp|nikto|nuclei|dast)"
context: "Dynamic application security testing"

# Container scanning
pattern: "(trivy|clair|anchore|snyk.*container|docker.*scan|grype)"
context: "Container image scanning"

# Infrastructure scanning
pattern: "(nessus|qualys|tenable|rapid7|openvas|nmap)"
context: "Infrastructure vulnerability scanning"

# CI/CD security scanning
pattern: "(security.*scan|scan.*security|vulnerability.*check|security.*gate)"
context: "Security scanning in CI/CD pipeline"

# Scanning schedule
pattern: "(schedule|cron|weekly|monthly|quarterly)\s*.*?(scan|audit|check|vulnerability)"
context: "Scanning schedule configuration"

# Penetration testing
pattern: "(pentest|penetration.*test|pen.*test|security.*assessment)"
context: "Penetration testing references"
```

**Pass Criteria:**
- Dependency scanning runs in CI/CD (npm audit, Snyk, etc.)
- SAST tool is configured and runs on every PR
- DAST scanning is performed quarterly (minimum)
- Container images are scanned before deployment
- Infrastructure scanning is performed quarterly
- Penetration testing is performed annually
- Critical/high findings are remediated within 30 days

**Fail Indicators:**
- No dependency scanning
- No SAST tool configured
- No DAST scanning
- No container scanning
- No infrastructure scanning
- No penetration testing schedule

**Fix Guide:**
- Add `npm audit --audit-level=high` or Snyk to CI/CD pipeline.
- Configure SAST: SonarQube, Semgrep, or CodeQL on every PR.
- Schedule quarterly DAST scans with OWASP ZAP or Burp Suite.
- Add container scanning: Trivy or Snyk Container in CI/CD.
- Schedule quarterly infrastructure scans with Nessus or Qualys.
- Plan annual penetration testing with a qualified assessor.
- Define SLAs for remediation: critical (24h), high (7d), medium (30d), low (90d).
- Track and report vulnerability metrics quarterly.
