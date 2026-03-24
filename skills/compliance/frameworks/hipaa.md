# HIPAA Code-Level Compliance Checks

Reference framework for code-level HIPAA (Health Insurance Portability and Accountability Act) compliance auditing.
For applications that create, receive, maintain, or transmit Protected Health Information (PHI).

---

## 1. PHI Detection (Health Records, Patient Data Patterns)

**What to Check:** All PHI must be identified, classified, and protected. PHI includes any individually identifiable health information.

**Grep Patterns:**
```
# Patient/health data fields
pattern: "(patient|health|medical|diagnosis|treatment|prescription|medication|allergy|lab.*result|vital.*sign|blood.*pressure|heart.*rate)"
context: "PHI data field - verify protection"

# Personal identifiers with health context
pattern: "(patientId|patient_id|mrn|MRN|medical.*record|health.*id|member.*id|subscriber.*id)"
context: "Patient identifier - PHI field"

# Clinical data
pattern: "(icd.*code|cpt.*code|ndc.*code|snomed|loinc|hl7|fhir|dicom)"
context: "Clinical coding/standard - PHI handling"

# Insurance/billing
pattern: "(insurance|claim|benefit|copay|deductible|payer|billing.*code|npi|NPI)"
context: "Insurance/billing data - PHI related"

# PHI in 18 HIPAA identifiers
pattern: "(name|address|date|phone|fax|email|ssn|social.*security|medical.*record|health.*plan|account|certificate|license|vehicle|device.*identifier|url|ip.*address|biometric|photo)"
context: "HIPAA identifier - check if combined with health data"

# PHI storage
pattern: "(save|store|persist|insert|create|write)\s*.*?(patient|health|medical|diagnosis|prescription)"
context: "PHI storage operation"

# PHI in logs
pattern: "(log|logger|console|print)\s*.*?(patient|health|medical|diagnosis|mrn|prescription)"
context: "PHI in log output - HIPAA violation"
```

**Pass Criteria:**
- All PHI fields are identified and documented
- PHI is classified by sensitivity level
- PHI is encrypted at rest and in transit
- PHI access is logged
- PHI is not present in logs or error messages
- PHI is minimized (only necessary data collected)

**Fail Indicators:**
- PHI fields not identified or classified
- PHI stored in plaintext
- PHI in log output
- PHI accessed without logging
- Excessive PHI collection

**Fix Guide:**
- Create a PHI inventory document listing all PHI fields, their locations, and protection measures.
- Classify PHI: high (SSN, diagnosis), medium (name, address), standard (MRN).
- Encrypt all PHI at rest (AES-256) and in transit (TLS 1.2+).
- Remove PHI from all log statements; use identifiers instead: `log('Accessing patient', { mrn: hash(mrn) })`.
- Implement data minimization: only collect PHI necessary for the specific purpose.
- Regular PHI discovery scans to find unprotected PHI in databases and file systems.

---

## 2. Encryption at Rest and in Transit

**What to Check:** All PHI must be encrypted both at rest (database, files, backups) and in transit (network).

**Grep Patterns:**
```
# Encryption at rest
pattern: "(encrypt|AES|aes-256|pgcrypto|transparent.*data.*encryption|TDE|column.*encrypt|field.*encrypt)"
context: "Encryption at rest implementation"

# TLS/HTTPS
pattern: "(https|TLS|tls|ssl|HSTS|Strict-Transport|SECURE_SSL_REDIRECT)"
context: "Transport encryption"

# Unencrypted storage
pattern: "(plaintext|plain_text|unencrypted|cleartext)\s*.*?(patient|health|medical|phi|ePHI)"
context: "Unencrypted PHI storage"

# Database encryption
pattern: "(database.*encrypt|RDS.*encrypt|encrypted.*volume|encrypt.*backup|LUKS|BitLocker|FileVault)"
context: "Database/volume encryption"

# File storage encryption
pattern: "(S3.*encrypt|SSE|server.*side.*encrypt|client.*side.*encrypt|encrypted.*bucket)"
context: "File storage encryption"

# Key management
pattern: "(KMS|key.*management|key.*vault|HSM|encryption.*key|master.*key)"
context: "Encryption key management"

# HTTP connections
pattern: "http://(?!localhost|127\.0\.0\.1|0\.0\.0\.0)"
context: "Unencrypted HTTP - HIPAA violation for PHI"

# Email encryption
pattern: "(email|smtp|sendmail).*?(encrypt|tls|secure|smtps)"
context: "Email encryption for PHI"
```

**Pass Criteria:**
- All PHI encrypted at rest with AES-256 or equivalent
- All connections transmitting PHI use TLS 1.2+
- Database encryption is enabled (volume-level or column-level)
- File storage is encrypted (S3 SSE, encrypted volumes)
- Backups are encrypted
- Key management system is in place
- Email containing PHI is encrypted

**Fail Indicators:**
- PHI stored without encryption
- HTTP connections for PHI transmission
- Database without encryption
- Unencrypted backups
- No key management
- PHI sent via unencrypted email

**Fix Guide:**
- Enable database encryption: RDS encryption, PostgreSQL pgcrypto for column-level.
- Use AES-256-GCM for application-level encryption of PHI fields.
- Enforce TLS 1.2+ for all connections: HTTPS, database TLS, email TLS.
- Enable S3 SSE-KMS for file storage.
- Encrypt all backups: RDS automated backup encryption, volume encryption.
- Use AWS KMS or HashiCorp Vault for key management.
- Use encrypted email or secure messaging for PHI communication.
- Set HSTS header with long max-age.

---

## 3. Access Logging

**What to Check:** All access to PHI must be logged with sufficient detail: who, what, when, where, why.

**Grep Patterns:**
```
# PHI access logging
pattern: "(audit.*log|access.*log|phi.*log|hipaa.*log|activity.*log)\s*.*?(patient|health|medical|record)"
context: "PHI access logging"

# Log fields
pattern: "(userId|user_id|timestamp|action|resource|reason|ip|outcome)\s*.*?(patient|phi|health)"
context: "PHI access log fields"

# Read access logging
pattern: "(read|view|access|retrieve|fetch|get)\s*.*?(patient|health|medical|record)\s*.*?(log|audit|track)"
context: "PHI read access logging"

# Write access logging
pattern: "(create|update|modify|delete|write)\s*.*?(patient|health|medical|record)\s*.*?(log|audit|track)"
context: "PHI write access logging"

# Break-the-glass logging
pattern: "(break.*glass|emergency.*access|override.*access|btg)"
context: "Emergency access logging"

# Log review
pattern: "(log.*review|review.*log|audit.*review|monitor.*access)"
context: "PHI access log review process"

# Log retention
pattern: "(log.*retention|retain.*log|log.*archive|six.*year|6.*year|2190)"
context: "Log retention (HIPAA requires 6 years)"

# Centralized logging
pattern: "(CloudWatch|Splunk|ELK|Datadog|SIEM|syslog|fluentd)"
context: "Centralized log management"
```

**Pass Criteria:**
- All PHI access (read, write, delete) is logged
- Logs include: timestamp, userId, action, resource, reason/purpose, IP, outcome
- Emergency/break-the-glass access is logged with justification
- Logs are centralized and tamper-proof
- Log retention: minimum 6 years (HIPAA requirement)
- Regular log review is performed (minimum weekly)
- Anomalous access patterns are alerted

**Fail Indicators:**
- PHI access not logged
- Incomplete log records
- No emergency access logging
- Logs stored locally only
- Log retention less than 6 years
- No log review process

**Fix Guide:**
- Implement PHI access logging middleware: log every CRUD operation on PHI.
- Include required fields: `{ timestamp, userId, action, resource, patientId, reason, ip, outcome }`.
- Implement break-the-glass with mandatory justification logging.
- Send logs to centralized SIEM (Splunk, CloudWatch, ELK).
- Configure 6-year retention for PHI access logs.
- Set up weekly automated log review with anomaly detection.
- Alert on: bulk PHI access, after-hours access, access to VIP records, break-the-glass events.

---

## 4. Minimum Necessary Principle

**What to Check:** Access to PHI must be limited to the minimum necessary for the user's job function.

**Grep Patterns:**
```
# Role-based PHI access
pattern: "(role|permission)\s*.*?(patient|health|medical|phi|record)"
context: "Role-based PHI access control"

# Granular data access
pattern: "(select|fields|attributes|columns|projection)\s*.*?(patient|health|medical)"
context: "PHI field-level access control"

# Over-fetching PHI
pattern: "(SELECT\s+\*|findAll\(\)|find\(\s*\{\s*\}|toJSON\(\))\s*.*?(patient|health|medical)"
context: "Fetching all PHI fields - minimum necessary violation"

# API response filtering
pattern: "(serialize|toJSON|toResponse|transform|dto)\s*.*?(patient|health|medical)"
context: "PHI response filtering"

# Role-based field visibility
pattern: "(visible|hidden|accessible|restricted)\s*.*?(field|column|attribute)\s*.*?(role|permission)"
context: "Field-level visibility based on role"

# Bulk PHI access
pattern: "(export|download|bulk|batch|dump|report)\s*.*?(patient|health|medical|record)"
context: "Bulk PHI access - check authorization and logging"
```

**Pass Criteria:**
- PHI access is role-based (doctors see clinical, billing sees financial)
- API responses return only necessary PHI fields per role
- No SELECT * on PHI tables
- Bulk PHI access requires additional authorization
- Report generation is restricted and logged
- Data segmentation is implemented (department-level access)

**Fail Indicators:**
- All users see all PHI fields
- SELECT * on PHI tables
- No role-based field filtering
- Unrestricted bulk export
- No data segmentation

**Fix Guide:**
- Define PHI access levels per role: doctor (all clinical), nurse (assigned patients), billing (financial only).
- Implement field-level access control: different serializers/DTOs per role.
- Replace SELECT * with explicit field selection based on role.
- Require manager approval for bulk PHI access.
- Implement department/unit-based data segmentation.
- Log all PHI access with purpose justification.

---

## 5. Business Associate Agreement (BAA) Tracking

**What to Check:** All third parties (business associates) that access PHI must have a signed BAA.

**Grep Patterns:**
```
# Third-party services handling PHI
pattern: "(aws|azure|gcp|google.*cloud|heroku|digitalocean|mongodb.*atlas)"
context: "Cloud provider - must have BAA"

# Communication services
pattern: "(twilio|sendgrid|mailgun|ses|sns|push.*notification)"
context: "Communication service handling PHI - must have BAA"

# Analytics/monitoring with PHI
pattern: "(sentry|bugsnag|datadog|newrelic|splunk|cloudwatch)"
context: "Monitoring service - check if PHI is sent"

# Payment processors
pattern: "(stripe|braintree|square|paypal)"
context: "Payment processor with patient data"

# Storage services
pattern: "(s3|blob.*storage|cloud.*storage|firebase.*storage|cdn)"
context: "Storage service for PHI - must have BAA"

# Third-party integrations
pattern: "(api.*integration|third.*party|vendor|external.*service|partner)"
context: "Third-party integration - check BAA status"

# EHR/EMR integrations
pattern: "(epic|cerner|allscripts|athenahealth|drchrono|hl7|fhir)"
context: "EHR integration - BAA required"
```

**Pass Criteria:**
- All third-party services that access PHI are identified
- BAA is signed with each business associate
- BAA status is tracked and documented
- BAAs are reviewed annually
- Subcontractor BAAs are in place (downstream)
- PHI is not sent to services without BAA

**Fail Indicators:**
- Third-party services handle PHI without BAA
- No BAA tracking system
- PHI sent to analytics/monitoring without BAA
- Outdated or unsigned BAAs

**Fix Guide:**
- Create a business associate inventory listing all services that access PHI.
- Ensure BAA is signed with each: AWS, Azure, GCP, Twilio, SendGrid, etc.
- Track BAA status: `{ vendor, service, baaDate, renewalDate, scope, status }`.
- Review BAAs annually; update for scope changes.
- Verify subcontractors of business associates also have BAAs.
- Do not send PHI to services without BAA (especially analytics, error tracking).
- Use HIPAA-eligible services: AWS HIPAA-eligible services, Azure HIPAA offerings.

---

## 6. Audit Trail

**What to Check:** A comprehensive audit trail must exist for all PHI-related activities and system changes.

**Grep Patterns:**
```
# Audit trail implementation
pattern: "(audit.*trail|auditTrail|audit_trail|change.*log|change.*history|revision|versioning)"
context: "Audit trail implementation"

# Data versioning
pattern: "(version|revision|history|changelog|temporal|bitemporal|event.*source)"
context: "Data versioning/history tracking"

# User activity tracking
pattern: "(activity.*log|user.*activity|action.*log|event.*log)"
context: "User activity tracking"

# System change tracking
pattern: "(config.*change|system.*change|deployment.*log|migration.*log)"
context: "System change tracking"

# PHI modification history
pattern: "(updated.*by|modified.*by|created.*by|deleted.*by|changed.*by)\s*.*?(patient|health|record)"
context: "PHI modification attribution"

# Audit report
pattern: "(audit.*report|compliance.*report|activity.*report)"
context: "Audit reporting capability"

# Immutability
pattern: "(immutable|append.*only|write.*once|tamper.*proof|blockchain)"
context: "Audit trail integrity"
```

**Pass Criteria:**
- Audit trail captures all PHI CRUD operations
- Data modifications include before/after values
- System changes (config, deployment) are tracked
- User activities are logged with attribution
- Audit trail is immutable (append-only)
- Audit reports can be generated for compliance reviews
- Retention: minimum 6 years

**Fail Indicators:**
- No audit trail for PHI operations
- Modifications without attribution
- No system change tracking
- Mutable audit records
- No audit reporting
- Insufficient retention

**Fix Guide:**
- Implement event sourcing or change data capture for PHI tables.
- Record: `{ timestamp, userId, action, table, recordId, oldValues, newValues }`.
- Track system changes: configuration, deployments, user provisioning.
- Store audit trail in append-only storage (separate database, immutable log).
- Generate monthly audit reports for compliance review.
- Set 6-year retention for all audit records.
- Implement tamper detection for audit trail integrity.

---

## 7. Secure Messaging

**What to Check:** Communication containing PHI must be encrypted and access-controlled.

**Grep Patterns:**
```
# Messaging implementation
pattern: "(message|chat|inbox|notification|email|sms)\s*.*?(patient|health|medical|phi)"
context: "PHI messaging implementation"

# Email with PHI
pattern: "(email|sendMail|sendEmail|smtp|ses|sendgrid)\s*.*?(patient|health|medical|result|report)"
context: "Email containing PHI"

# Push notifications
pattern: "(push.*notification|fcm|apns|firebase.*messaging)\s*.*?(patient|health|result)"
context: "Push notification with PHI"

# SMS with PHI
pattern: "(sms|text.*message|twilio)\s*.*?(patient|health|result|appointment)"
context: "SMS with PHI"

# In-app messaging
pattern: "(chat|message|conversation|thread)\s*.*?(doctor|patient|provider|clinician)"
context: "In-app clinical messaging"

# Message encryption
pattern: "(encrypt.*message|message.*encrypt|e2e|end.*to.*end|signal.*protocol)"
context: "Message encryption"

# Message access control
pattern: "(recipient|authorized|permission)\s*.*?(message|chat|inbox)"
context: "Message access control"
```

**Pass Criteria:**
- PHI in messages is encrypted in transit and at rest
- Only authorized recipients can access PHI messages
- Email with PHI uses encryption (TLS, S/MIME, or portal-based)
- Push notifications do not contain PHI in the preview
- SMS is avoided for PHI; if used, consent is obtained
- Message retention policy is enforced
- Secure messaging audit trail exists

**Fail Indicators:**
- PHI sent via unencrypted email
- PHI in push notification previews
- PHI in SMS without consent
- No message access control
- No message encryption
- No message audit trail

**Fix Guide:**
- Use encrypted email (TLS enforcement) or secure portal for PHI communication.
- Push notifications: use generic alerts ("You have a new message") without PHI content.
- Avoid SMS for PHI; if necessary, obtain patient consent and minimize content.
- Implement end-to-end encryption for in-app clinical messaging.
- Enforce message access control: only sender, recipient, and authorized providers.
- Log message access: who read/sent what and when.
- Implement message expiration/retention policy.

---

## 8. Data Disposal Procedures

**What to Check:** PHI must be securely disposed of when no longer needed, in all storage media.

**Grep Patterns:**
```
# Data deletion
pattern: "(delete|destroy|purge|wipe|dispose|shred|erase)\s*.*?(patient|health|medical|record|phi)"
context: "PHI disposal implementation"

# Soft delete concern
pattern: "(soft.*delete|isDeleted|deleted_at|paranoid|is_active.*false)\s*.*?(patient|health|medical)"
context: "Soft delete for PHI - data still exists"

# Data retention
pattern: "(retention|expire|ttl|lifecycle)\s*.*?(patient|health|medical|record)"
context: "PHI retention policy"

# Backup disposal
pattern: "(backup.*delete|backup.*purge|backup.*expire|backup.*lifecycle)"
context: "PHI backup disposal"

# Cache cleanup
pattern: "(cache.*clear|cache.*expire|cache.*evict|ttl)\s*.*?(patient|health)"
context: "PHI cache cleanup"

# Log disposal
pattern: "(log.*rotate|log.*delete|log.*archive|log.*expire)"
context: "PHI log disposal"

# Media sanitization
pattern: "(sanitize|overwrite|degauss|crypto.*erase|NIST.*800-88)"
context: "Media sanitization reference"
```

**Pass Criteria:**
- PHI disposal procedure is defined and documented
- Automated disposal based on retention policy
- Hard delete (not just soft delete) for expired PHI
- Backup disposal aligned with retention policy
- Cache containing PHI has TTL and cleanup
- Log disposal aligned with retention policy (6 years minimum)
- Media sanitization follows NIST 800-88

**Fail Indicators:**
- No PHI disposal procedure
- PHI kept indefinitely
- Only soft delete (data still recoverable)
- Backups with expired PHI not disposed
- No cache TTL for PHI

**Fix Guide:**
- Define retention periods: active treatment (indefinite), post-discharge (6 years minimum), legal holds (as required).
- Implement automated disposal jobs: cron to hard-delete expired PHI.
- For databases: hard delete + vacuum to reclaim space.
- Implement backup lifecycle: auto-delete backups older than retention period.
- Set cache TTL for PHI: `cache.set(key, phi, { ttl: 3600 })` (1 hour max).
- Log disposal actions: what was deleted, when, by whom/what.
- Reference NIST 800-88 for media sanitization: clear, purge, destroy.

---

## 9. Emergency Access Procedures

**What to Check:** Emergency access (break-the-glass) must exist for legitimate urgent access to PHI, with proper logging and review.

**Grep Patterns:**
```
# Break-the-glass
pattern: "(break.*glass|btg|emergency.*access|override.*access|urgent.*access)"
context: "Emergency access implementation"

# Emergency role
pattern: "(emergency.*role|emergency.*permission|override.*role|superuser.*emergency)"
context: "Emergency access role/permission"

# Emergency justification
pattern: "(reason|justification|rationale|explanation)\s*.*?(emergency|override|break.*glass)"
context: "Emergency access justification requirement"

# Emergency alerting
pattern: "(alert|notify|page|email)\s*.*?(emergency.*access|break.*glass|override)"
context: "Emergency access alerting"

# Emergency review
pattern: "(review|audit)\s*.*?(emergency|break.*glass|override)"
context: "Emergency access review process"

# Auto-expiration
pattern: "(expire|timeout|duration|limit)\s*.*?(emergency|override|break.*glass)"
context: "Emergency access auto-expiration"

# Emergency logging
pattern: "(log|audit|record)\s*.*?(emergency|break.*glass|override)\s*.*?(access|event)"
context: "Emergency access logging"
```

**Pass Criteria:**
- Break-the-glass procedure exists for urgent PHI access
- Emergency access requires documented justification
- Emergency access triggers immediate alert to privacy officer
- Emergency access has auto-expiration (1 hour maximum)
- All emergency access is logged with full detail
- Emergency access is reviewed within 24 hours
- Regular testing of emergency access procedures

**Fail Indicators:**
- No emergency access procedure
- Emergency access without justification
- No alerting on emergency access
- No auto-expiration for emergency access
- Emergency access not logged or reviewed

**Fix Guide:**
- Implement break-the-glass feature: special button/API with justification requirement.
- Log: `{ userId, patientId, justification, timestamp, accessLevel, autoExpireAt }`.
- Alert privacy officer immediately on emergency access.
- Auto-expire emergency access after 1 hour; require renewal with new justification.
- Mandatory review of all emergency access within 24 hours by privacy officer.
- Test emergency access procedures quarterly.
- Train all users on proper emergency access use.
- Implement post-access review workflow: privacy officer approves or flags for investigation.
