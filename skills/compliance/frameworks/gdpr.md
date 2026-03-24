# GDPR / KVKK Code-Level Compliance Checks

Reference framework for code-level GDPR (General Data Protection Regulation) and KVKK (Kisisel Verilerin Korunmasi Kanunu) compliance auditing.

---

## 1. Consent Mechanism

**What to Check:** Application must obtain explicit, informed consent before collecting personal data. Consent must be freely given, specific, and withdrawable.

**Grep Patterns:**
```
# Cookie consent
pattern: "(cookie.*consent|consent.*cookie|cookieBanner|cookie.*banner|cookie.*popup|gdpr.*banner|cookie.*notice)"
context: "Cookie consent implementation"

# Consent API/storage
pattern: "(consent.*accept|accept.*consent|consent.*grant|consent.*revoke|consent.*withdraw|consentStatus)"
context: "Consent management logic"

# Third-party consent managers
pattern: "(cookiebot|onetrust|quantcast|osano|termly|cookieconsent|cookie-consent)"
context: "Third-party consent management platform"

# Marketing consent
pattern: "(marketing.*consent|newsletter.*opt|email.*consent|subscribe.*consent|opt.?in)"
context: "Marketing communication consent"

# Consent record
pattern: "(consent.*log|consent.*record|consent.*audit|consent.*timestamp|consent.*version)"
context: "Consent audit trail"

# Pre-checked boxes
pattern: "(checked|selected|default.*true).*?(consent|agree|accept|subscribe|marketing)"
context: "Pre-checked consent - GDPR violation"
```

**Pass Criteria:**
- Cookie consent banner exists and blocks non-essential cookies until consent is given
- Consent is not pre-checked (opt-in, not opt-out)
- Consent can be withdrawn as easily as given
- Consent records are stored with timestamp and version

**Fail Indicators:**
- No cookie consent implementation found
- Cookies/tracking set before consent
- Pre-checked consent boxes
- No mechanism to withdraw consent

**Fix Guide:**
- Implement cookie consent banner that blocks all non-essential cookies/tracking until consent.
- Use opt-in (unchecked) checkboxes for all consent types.
- Store consent records: `{ userId, consentType, granted: true/false, timestamp, policyVersion }`.
- Provide consent withdrawal in user settings, equal in ease to granting consent.
- Use a consent management platform (CookieBot, OneTrust) for comprehensive coverage.

---

## 2. Data Deletion API (Right to Erasure)

**What to Check:** Users must be able to request deletion of their personal data. The system must delete or anonymize all PII across all storage systems.

**Grep Patterns:**
```
# Deletion endpoint
pattern: "(delete.*account|remove.*account|erase.*data|purge.*user|account.*deletion|gdpr.*delete|right.*forgotten)"
context: "Account/data deletion endpoint"

# Deletion in controllers
pattern: "(deleteUser|removeUser|destroyUser|anonymize|anonymise|pseudonymize)"
context: "User deletion logic"

# Cascade deletion
pattern: "(CASCADE|onDelete|on_delete|dependent.*destroy|cascade.*delete)"
context: "Cascading deletion of related data"

# Soft delete vs hard delete
pattern: "(soft.*delete|isDeleted|deleted_at|deletedAt|paranoid|is_active\s*=\s*false)"
context: "Soft delete may not satisfy GDPR erasure requirement"

# Backup/archive handling
pattern: "(backup|archive|snapshot|retention|cold.*storage)"
context: "Check if deletion extends to backups"

# Third-party data deletion
pattern: "(stripe|sendgrid|mailchimp|intercom|segment|mixpanel|amplitude).*?(delete|remove|erase)"
context: "Third-party service data deletion"
```

**Pass Criteria:**
- API endpoint exists for account/data deletion
- Deletion cascades to all related data
- Deletion extends to third-party services
- Deletion is completable within 30 days
- Hard delete or irreversible anonymization is performed

**Fail Indicators:**
- No deletion endpoint found
- Soft delete only (data still exists)
- No cascade to related tables
- Third-party data not addressed

**Fix Guide:**
- Create `DELETE /api/users/me` or `POST /api/users/me/delete-request` endpoint.
- Implement cascading deletion for all related data (orders, comments, files, etc.).
- For data needed for legal/financial reasons, anonymize instead of delete.
- Queue third-party deletion calls (Stripe, analytics, email providers).
- Process deletion within 30 days; confirm completion to the user.
- Maintain deletion audit log (who requested, when completed, what was deleted/anonymized).

---

## 3. Data Export API (Data Portability)

**What to Check:** Users must be able to export their personal data in a machine-readable format (JSON, CSV, XML).

**Grep Patterns:**
```
# Export endpoint
pattern: "(export.*data|download.*data|data.*portability|gdpr.*export|data.*download|takeout)"
context: "Data export endpoint"

# Export logic
pattern: "(toJSON|toCSV|generateExport|createExport|exportUser|downloadProfile)"
context: "Data export implementation"

# Export format
pattern: "(application/json|text/csv|application/xml|Content-Disposition|attachment)"
context: "Export file format and download headers"

# Comprehensive export
pattern: "(profile|orders|comments|posts|messages|files|activities|preferences)"
context: "Check if all user data types are included in export"

# Export notification
pattern: "(export.*ready|export.*complete|download.*link|export.*email)"
context: "Export completion notification"
```

**Pass Criteria:**
- Data export endpoint exists
- Export includes all user personal data across all tables
- Export format is machine-readable (JSON or CSV)
- Export is delivered securely (authenticated download link)

**Fail Indicators:**
- No data export functionality found
- Export is incomplete (missing data types)
- Export only available to admins, not self-service

**Fix Guide:**
- Create `GET /api/users/me/export` or `POST /api/users/me/export-request` endpoint.
- Collect all user data: profile, activity, orders, messages, files, preferences.
- Generate JSON or CSV export with clear field labels.
- For large exports, process async and email a secure download link.
- Secure the download link with authentication and expiration (24 hours).
- Include data from all storage systems (database, file storage, CDN).

---

## 4. PII Detection and Masking

**What to Check:** Personal Identifiable Information must be identified, classified, and masked/encrypted where appropriate.

**Grep Patterns:**
```
# PII fields
pattern: "(email|phone|phoneNumber|phone_number|firstName|first_name|lastName|last_name|address|birthDate|birth_date|dateOfBirth|ssn|national_id|tc_kimlik|passport)"
context: "PII field - check storage and display handling"

# PII in logs
pattern: "(console\.(log|info|warn|error)|logger?\.(info|warn|error|debug)|print|NSLog|Log\.).*?(email|phone|name|address|ssn|password|token|card)"
context: "PII in log output"

# PII masking
pattern: "(mask|redact|anonymize|pseudonymize|obfuscate|sanitize).*?(email|phone|name|address|ssn|card)"
context: "PII masking implementation"

# PII in error messages
pattern: "(throw|Error|Exception|reject).*?(email|phone|name|user)"
context: "PII in error messages"

# PII in URLs
pattern: "(email|phone|name|ssn|card).*?[?&=]|[?&].*?(email|phone|name)"
context: "PII in URL query parameters"

# PII in analytics
pattern: "(analytics|track|identify|mixpanel|amplitude|segment).*?(email|phone|name|userId)"
context: "PII sent to analytics - check anonymization"
```

**Pass Criteria:**
- PII fields are identified and classified in the data model
- PII is masked in logs and error messages
- PII is not included in URLs
- PII sent to analytics is anonymized or pseudonymized
- Display masking exists for sensitive fields (email: j***@example.com, phone: ***1234)

**Fail Indicators:**
- PII logged in plaintext
- PII in URL query parameters
- PII sent to third-party analytics without anonymization
- No masking for sensitive field display

**Fix Guide:**
- Implement a PII detection utility that identifies sensitive fields.
- Create a logging middleware that redacts PII: `email: "j***@example.com"`.
- Never include PII in URLs; use request body for POST requests.
- Anonymize PII before sending to analytics: hash or use pseudonymous IDs.
- Implement display masking for UI: show only last 4 digits of phone, mask email.
- Maintain a data classification document listing all PII fields and their sensitivity level.

---

## 5. Encryption at Rest and in Transit

**What to Check:** Personal data must be encrypted both at rest (storage) and in transit (network).

**Grep Patterns:**
```
# TLS/HTTPS
pattern: "(https://|TLS|ssl|createServer.*https|SECURE_SSL_REDIRECT|HSTS|Strict-Transport)"
context: "Transport encryption"

# HTTP (unencrypted)
pattern: "http://(?!localhost|127\.0\.0\.1|0\.0\.0\.0)"
context: "Unencrypted HTTP connection"

# Database encryption
pattern: "(encrypt.*column|encrypted.*field|pgcrypto|AES.*encrypt|at.rest|encryption.at.rest)"
context: "Database-level encryption"

# File encryption
pattern: "(encrypt.*file|encrypted.*storage|encrypt.*upload|AES|cipher)"
context: "File/storage encryption"

# Encryption libraries
pattern: "(crypto|bcrypt|argon2|scrypt|aes-256|chacha20)"
context: "Encryption implementation"

# Unencrypted sensitive storage
pattern: "(plaintext|plain_text|unencrypted|cleartext).*?(password|token|secret|pii|personal)"
context: "Sensitive data stored without encryption"
```

**Pass Criteria:**
- All external connections use HTTPS/TLS 1.2+
- HSTS header is configured
- Sensitive database fields are encrypted (column-level encryption)
- Uploaded files containing PII are encrypted
- Passwords are hashed with bcrypt/Argon2/scrypt

**Fail Indicators:**
- HTTP connections to external services
- No HSTS header
- Sensitive fields stored in plaintext
- Passwords not properly hashed

**Fix Guide:**
- Enforce HTTPS everywhere: redirect HTTP to HTTPS, set HSTS header.
- Use column-level encryption for highly sensitive fields (SSN, health data).
- Enable database-level encryption at rest (AWS RDS encryption, PostgreSQL pgcrypto).
- Encrypt file uploads containing PII before storage.
- Use bcrypt (cost 12+) or Argon2id for password hashing.
- Configure TLS 1.2 minimum; disable TLS 1.0 and 1.1.

---

## 6. Data Retention Policy Enforcement

**What to Check:** Personal data must not be kept longer than necessary. Automated retention policies must enforce deletion/anonymization.

**Grep Patterns:**
```
# Retention policy
pattern: "(retention|expire|ttl|maxAge|max_age|cleanup|purge|archive|data.*lifecycle)"
context: "Data retention configuration"

# Scheduled cleanup
pattern: "(cron|scheduler|setInterval|periodic.*delete|cleanup.*job|purge.*task)"
context: "Automated data cleanup job"

# Expiration fields
pattern: "(expiresAt|expires_at|expiry|validUntil|valid_until|createdAt.*retention)"
context: "Expiration timestamp on data"

# Session/token expiry
pattern: "(session.*expire|token.*expire|maxAge|cookie.*age)"
context: "Session/token retention"

# Log retention
pattern: "(log.*rotation|log.*retention|log.*expire|logrotate|maxFiles|maxSize)"
context: "Log retention configuration"

# Old data queries
pattern: "(WHERE.*created.*<|WHERE.*date.*<|older.*than|before.*date)"
context: "Query for old data - check if used for cleanup"
```

**Pass Criteria:**
- Data retention policy is defined and documented
- Automated jobs enforce retention (delete/anonymize expired data)
- Session/token expiration is configured
- Log rotation with retention limits is in place
- User-uploaded content has retention lifecycle

**Fail Indicators:**
- No retention policy or automated cleanup
- Data kept indefinitely
- No session/token expiration
- Unbounded log storage

**Fix Guide:**
- Define retention periods for each data category (user profiles: until deletion, logs: 90 days, analytics: 2 years).
- Implement cron jobs to delete/anonymize data past retention period.
- Set session expiry (idle: 30 min, absolute: 24 hours).
- Configure log rotation with retention limits (90 days for app logs, 1 year for security logs).
- Add `expiresAt` fields to temporary data (invitations, password reset tokens).
- Document retention policy and make it accessible to users.

---

## 7. Third-Party Data Sharing Controls

**What to Check:** Sharing personal data with third parties must be controlled, documented, and consented to.

**Grep Patterns:**
```
# Third-party SDKs and APIs
pattern: "(analytics|tracking|facebook|google|twitter|linkedin|stripe|sendgrid|mailchimp|intercom|segment|mixpanel|amplitude|hotjar|fullstory|sentry|bugsnag|datadog)"
context: "Third-party service - check data shared"

# Data sent to external services
pattern: "(fetch|axios|request|post|put)\s*\(\s*['\"]https?://(?!.*localhost|.*127\.0\.0\.1)"
context: "Data sent to external endpoint"

# Pixel/tracking
pattern: "(fbq|gtag|ga\(|analytics\.track|pixel|beacon|sendBeacon)"
context: "Tracking pixel/beacon sending user data"

# Social login
pattern: "(oauth|openid|social.*login|facebook.*login|google.*login|apple.*login)"
context: "Social login - check data received and shared"

# Third-party scripts
pattern: "<script.*src=['\"]https?://(?!.*localhost)"
context: "Third-party script loaded - check data access"

# Data processing agreement
pattern: "(DPA|data.*processing.*agreement|processor|sub-processor)"
context: "Data processing agreement reference"
```

**Pass Criteria:**
- All third-party data sharing is documented
- User consent is obtained before sharing data with third parties
- Third-party scripts are loaded conditionally based on consent
- Data processing agreements (DPAs) are in place with processors
- Minimum data is shared (data minimization)

**Fail Indicators:**
- Third-party tracking loaded without consent
- PII shared with analytics without anonymization
- No documentation of third-party data sharing
- Social login sharing more data than necessary

**Fix Guide:**
- Load third-party scripts conditionally based on consent: only after user accepts.
- Anonymize data sent to analytics: use hashed IDs, not email/names.
- Document all third-party data sharing in privacy policy.
- Ensure DPAs are signed with all data processors.
- Implement tag management (Google Tag Manager) with consent-based triggers.
- Review third-party SDKs quarterly; remove unnecessary ones.

---

## 8. Cookie Consent Implementation

**What to Check:** Cookies must be categorized and only essential cookies set without consent. Non-essential cookies require explicit opt-in consent.

**Grep Patterns:**
```
# Cookie setting
pattern: "(document\.cookie|res\.cookie|response\.set_cookie|Set-Cookie|setCookie|cookies\.set)"
context: "Cookie being set - check category and consent"

# Cookie categories
pattern: "(essential|necessary|functional|analytics|performance|marketing|advertising|preference)"
context: "Cookie categorization"

# Consent before cookies
pattern: "(consent.*cookie|cookie.*consent|checkConsent|hasConsent|isConsented)"
context: "Consent check before setting cookies"

# Third-party cookies
pattern: "(facebook|google|analytics|doubleclick|adsense|twitter|linkedin|hubspot).*cookie"
context: "Third-party cookie - requires consent"

# Cookie policy
pattern: "(cookie.*policy|cookie.*notice|cookie.*information)"
context: "Cookie policy document"

# Local storage as cookie alternative
pattern: "(localStorage|sessionStorage).*?(tracking|analytics|marketing|user.*id)"
context: "Local storage used for tracking - same consent rules apply"
```

**Pass Criteria:**
- Cookie consent banner is implemented
- Cookies are categorized (essential, functional, analytics, marketing)
- Only essential cookies are set before consent
- Non-essential cookies are blocked until explicit opt-in
- Cookie preferences can be changed at any time
- Cookie policy is accessible

**Fail Indicators:**
- No cookie consent banner
- All cookies set before consent
- No cookie categorization
- No way to change cookie preferences after initial choice

**Fix Guide:**
- Implement cookie consent banner with category-based opt-in.
- Categorize all cookies: essential (always), functional, analytics, marketing.
- Block non-essential cookies until consent: wrap tracking scripts in consent check.
- Provide persistent access to cookie settings (footer link or settings page).
- Document all cookies in a cookie policy with name, purpose, expiry, and category.
- Treat localStorage/sessionStorage tracking as cookies for consent purposes.

---

## 9. Privacy Policy Link Accessibility

**What to Check:** Privacy policy must be easily accessible from all pages and at all data collection points.

**Grep Patterns:**
```
# Privacy policy link
pattern: "(privacy.*policy|privacy.*notice|datenschutz|gizlilik.*politikasi|kvkk|kisisel.*veri)"
context: "Privacy policy reference"

# Footer links
pattern: "(footer|Footer|<footer).*?(privacy|policy|legal|terms)"
context: "Privacy policy in footer"

# Registration/signup forms
pattern: "(register|signup|sign.*up|create.*account).*?(privacy|policy|terms|agree)"
context: "Privacy policy at registration"

# Data collection forms
pattern: "(form|Form|<form).*?(privacy|policy|consent|agree)"
context: "Privacy policy at data collection point"

# Contact/support forms
pattern: "(contact|support|feedback|inquiry).*?(form|Form)"
context: "Contact form - check for privacy notice"

# Checkout/payment
pattern: "(checkout|payment|purchase|order).*?(privacy|policy|terms)"
context: "Privacy policy at checkout"
```

**Pass Criteria:**
- Privacy policy link in page footer (all pages)
- Privacy policy link at registration/signup
- Privacy policy link at all data collection forms
- Privacy policy link at checkout
- Privacy policy is up-to-date and comprehensive
- Privacy policy is in user's language

**Fail Indicators:**
- No privacy policy link found
- Privacy policy not accessible from registration
- Privacy policy missing from data collection forms
- Outdated privacy policy

**Fix Guide:**
- Add privacy policy link to global footer component.
- Add privacy policy link with checkbox at registration: "I have read and agree to the Privacy Policy".
- Add privacy policy notice at all data collection forms.
- Ensure privacy policy covers: data collected, purpose, legal basis, retention, third parties, user rights.
- Translate privacy policy to all supported languages.
- Include last updated date and change history.
