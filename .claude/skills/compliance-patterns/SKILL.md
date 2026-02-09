---
name: compliance-patterns
description: Data classification framework, HIPAA safeguards, audit trails, and compliance documentation patterns for regulated environments
---

# Compliance Patterns

Systematic approach to translating compliance requirements (HIPAA, GDPR, SOC2, financial regulations) into technical controls and verifiable acceptance checks.

## Scope

**Use for:** Healthcare data (HIPAA), personal data (GDPR), financial data, audit requirements, data retention policies

**Not for:** General security (see security-engineer), code implementation (see developer)

---

# Data Classification Framework

## Classification Levels

**PHI (Protected Health Information)**
- Individually identifiable health information
- Governed by HIPAA (45 CFR 160.103)
- Requires encryption at rest, encryption in transit, audit trails, minimum necessary access
- Examples: Patient name + medical condition, SSN + diagnosis, treatment records

**PII (Personally Identifiable Information)**
- Information that can identify an individual
- Governed by GDPR (EU), CCPA (California), various state laws
- Requires consent, right to deletion, access logs, breach notification
- Examples: Full name, email, phone, address, IP address, government IDs

**Sensitive PII (High-Risk)**
- PII with higher risk if breached
- Additional protections required: field-level encryption, restricted access, enhanced audit
- Examples: SSN, financial account numbers, biometric data, credentials

**Public**
- Non-identifiable information safe to expose
- No special handling required
- Examples: Product names, public content, aggregate statistics (if properly anonymized)

## Handling Rules by Classification

| Classification | Encryption at Rest | Encryption in Transit | Logging | Access Control | Retention |
|----------------|-------------------|----------------------|---------|----------------|-----------|
| PHI | AES-256 (required) | TLS 1.3 (required) | Audit all access, never log PHI content | RBAC + minimum necessary | HIPAA retention + secure deletion |
| Sensitive PII | AES-256 (required) | TLS 1.3 (required) | Audit all access, never log values | RBAC + need-to-know | Per regulation + secure deletion |
| PII | AES-256 (recommended) | TLS 1.2+ (required) | Log access, mask in logs | RBAC | Per regulation + deletion on request |
| Public | Not required | TLS 1.2+ (recommended) | Standard app logs | Normal access control | Standard retention |

**See `/references/data-classification.md` for field-level examples.**

---

# HIPAA Technical Safeguards

## Required Controls (45 CFR 164.312)

### Access Control (§164.312(a)(1))
- **Unique user identification**: Every user must have unique ID
- **Emergency access procedure**: Break-glass access with full audit trail
- **Automatic logoff**: Session timeout for inactive users
- **Encryption and decryption**: Encrypt PHI at rest and in transit

### Audit Controls (§164.312(b))
- **Audit trail**: Record all PHI access, modifications, deletions
- **Tamper-proof logs**: Logs must be immutable or cryptographically signed
- **Log retention**: Minimum 6 years (state laws may require longer)
- **Log content**: User, timestamp, action, resource, outcome (success/failure)

### Integrity (§164.312(c)(1))
- **Data integrity**: Ensure PHI is not altered or destroyed improperly
- **Mechanisms**: Checksums, digital signatures, version control

### Person or Entity Authentication (§164.312(d))
- **Verify identity**: Authenticate users before granting access to PHI
- **Mechanisms**: Multi-factor authentication for remote access

### Transmission Security (§164.312(e)(1))
- **Encryption in transit**: TLS 1.3 for all PHI transmission
- **Integrity controls**: Verify data hasn't been altered during transmission

## Implementation Patterns

**Encryption at rest:**
```yaml
# Database-level encryption
- Enable transparent data encryption (TDE) on database
- Use AES-256 encryption for PHI columns
- Rotate encryption keys annually
- Store keys in separate key management service (KMS)

# Field-level encryption for high-sensitivity fields
- Encrypt SSN, financial accounts separately from database encryption
- Use deterministic encryption for fields requiring lookup
- Use randomized encryption for fields not requiring lookup
```

**Encryption in transit:**
```yaml
# TLS configuration
- Enforce TLS 1.3 minimum
- Disable weak cipher suites
- Use HSTS headers
- Certificate pinning for mobile apps
```

**Audit logging:**
```yaml
# What to log
- User ID (not username if PHI)
- Timestamp (UTC, ISO 8601)
- Action (view, create, update, delete)
- Resource ID (e.g., patient_id)
- Outcome (success, failure, reason)
- IP address
- Session ID

# What NEVER to log
- PHI content (names, diagnoses, SSNs)
- Full request/response bodies containing PHI
- Unencrypted credentials
```

**Access control:**
```yaml
# Role-based access control (RBAC)
- Define roles: admin, provider, staff, patient
- Assign minimum necessary permissions per role
- Require explicit grants, no implicit access
- Implement row-level security (RLS) for PHI tables
- Break-glass access: log + alert on use
```

---

# Compliance.md Template

Every feature handling regulated data must have a `compliance.md` file.

```markdown
# Compliance: {Feature Name}

## Overview
{Brief description of feature and compliance scope}

## Applicable Regulations
- [ ] HIPAA
- [ ] GDPR
- [ ] CCPA
- [ ] SOC2
- [ ] Other: {specify}

---

## Data Classification

### PHI
- `{field}`: {description} — {handling requirement}
- `{field}`: {description} — {handling requirement}

### PII
- `{field}`: {description} — {handling requirement}

### Public
- `{field}`: {description}

---

## Technical Controls

### Encryption
- **At rest**: {implementation}
- **In transit**: {implementation}
- **Key management**: {implementation}

### Access Control
- **Authentication**: {mechanism}
- **Authorization**: {RBAC/ABAC rules}
- **Minimum necessary**: {enforcement method}

### Audit Trail
- **Events logged**: {list}
- **Log storage**: {location, retention}
- **Log protection**: {immutability, encryption}

### Data Retention
- **Retention period**: {duration}
- **Deletion process**: {secure deletion method}
- **Backup retention**: {duration}

---

## Business Associate Agreements (BAA)

### Third-Party Services Handling PHI
- {Service name}: {data shared, BAA status, contract reference}

---

## Breach Notification

### Detection
- {How breaches are detected}

### Response Plan
- {Steps to take if breach detected}
- {Notification timeline: 60 days for HIPAA}
- {Who to notify: OCR, affected individuals, media if >500}

---

## Acceptance Checks

- [ ] {Verifiable compliance check 1}
- [ ] {Verifiable compliance check 2}
- [ ] {Verifiable compliance check 3}

---

## References
- Spec: `.ops/build/v{x}/{feature}/specs.md`
- System design: `.ops/build/v{x}/{feature}/system-design.yaml`
- Tasks: `.ops/build/v{x}/{feature}/tasks.yaml`
```

---

# Audit Trail Requirements

## What to Log

**Required for all PHI/PII access:**
- User identifier (ID, not name if name is PHI)
- Timestamp (UTC, ISO 8601 format)
- Action performed (view, create, update, delete, export)
- Resource accessed (patient ID, record ID)
- Outcome (success, failure, error code)
- IP address (if required by regulation)
- Session ID (for correlating related actions)

**Optional but recommended:**
- User agent (for detecting unusual access patterns)
- Geolocation (if consent obtained)
- Source system/application
- Access method (web, API, mobile app)

## What NEVER to Log

**Absolutely forbidden in logs:**
- PHI content: names, diagnoses, SSNs, medical record numbers, treatment details
- PII content: full addresses, phone numbers, email (log hashed versions if needed)
- Unencrypted credentials: passwords, API keys, tokens
- Full request/response bodies containing PHI/PII

**Safe alternatives:**
- Log resource IDs instead of content: `patient_id: 12345` not `patient_name: "John Doe"`
- Log hashed values if lookup needed: `email_hash: sha256(email)`
- Log only non-sensitive fields: `action: "update_patient", field_count: 3` not field values

## Log Storage and Protection

**Requirements:**
- **Immutability**: Logs cannot be edited or deleted (append-only)
- **Encryption**: Encrypt logs at rest if they contain any identifiable info
- **Access control**: Restrict log access to security/compliance personnel
- **Retention**: Minimum 6 years for HIPAA (check state laws for longer requirements)
- **Backup**: Include logs in backup strategy, same retention policy

**Implementation patterns:**
```yaml
# Centralized logging
- Use dedicated log service (e.g., CloudWatch, Datadog, Splunk)
- Enable log immutability/write-once-read-many
- Encrypt log streams
- Set retention policy to auto-delete after required period

# Tamper detection
- Use cryptographic signatures or blockchain for log integrity
- Alert on missing log entries or sequence gaps
```

---

# Data Retention and Deletion

## Retention Policies

**HIPAA:**
- Medical records: 6 years minimum (state laws may require longer, e.g., 10+ years)
- Audit logs: 6 years minimum
- Business associate agreements: 6 years after termination
- Note: Check state-specific requirements (California: 7 years, some states: until patient age 18+7 years)

**GDPR:**
- Personal data: Only as long as necessary for stated purpose
- Right to erasure: Must delete on request unless legal basis to retain
- Consent records: Keep proof of consent for duration + statute of limitations

**Financial/SOC2:**
- Varies by regulation and contract
- Common: 7 years for financial records

## Deletion Procedures

**Secure deletion requirements:**
- **Hard delete**: Permanently remove from active database and backups
- **Cryptographic erasure**: Delete encryption keys, rendering data unreadable
- **Overwriting**: For physical media, use secure wipe (DoD 5220.22-M or equivalent)

**Soft delete (not compliant for deletion requests):**
- Marking records as deleted but retaining data
- Only acceptable for operational purposes, not for compliance obligations

**Implementation checklist:**
```yaml
- [ ] Delete from primary database
- [ ] Delete from read replicas
- [ ] Delete from backups (or ensure backup encryption keys are rotated)
- [ ] Delete from caches (Redis, Memcached)
- [ ] Delete from search indexes (Elasticsearch, Algolia)
- [ ] Delete from CDN/edge caches
- [ ] Delete from third-party services (analytics, CRM, support tools)
- [ ] Delete from log files (if PII/PHI present)
- [ ] Delete from file storage (S3, GCS)
- [ ] Delete from archival/cold storage
```

**Verification:**
- Log all deletion operations with user, timestamp, records deleted
- Provide deletion confirmation to user (if GDPR request)
- Retain deletion audit trail for compliance proof

**Automated retention:**
```yaml
# Scheduled cleanup jobs
- Run daily/weekly to identify expired data
- Grace period before final deletion (e.g., 30 days in "to be deleted" state)
- Alert on deletion failures
- Monitor deletion job execution
```

---

# Common Compliance Patterns

## Pattern 1: PHI Access Logging

**Use case:** Log every access to patient records for HIPAA audit trail

**Implementation:**
```yaml
# Database trigger or application middleware
on_patient_record_access:
  log:
    user_id: {authenticated_user_id}
    timestamp: {utc_iso8601}
    action: {view|create|update|delete}
    patient_id: {patient_id}  # ID only, not patient name
    record_type: {appointment|lab_result|prescription}
    outcome: {success|failure}
    error_code: {if failure}

  # Do NOT log
  # patient_name, diagnosis, treatment details, SSN
```

## Pattern 2: Minimum Necessary Access (HIPAA)

**Use case:** Enforce that users only access PHI necessary for their job function

**Implementation:**
```yaml
# Row-level security (RLS) in database
policy: user_access_patient_records
  using: (
    # Users can only access patients they are treating
    EXISTS (
      SELECT 1 FROM care_team
      WHERE care_team.patient_id = patients.id
      AND care_team.provider_id = current_user_id()
    )
    OR
    # Or if they have break-glass access (log this!)
    current_user_has_role('emergency_access')
  )

# Application-level check
if user.role == 'provider':
  # Check care team membership
  if not user.is_on_care_team(patient_id):
    if not user.request_break_glass_access(patient_id):
      deny_access()
    else:
      alert_security_team(user, patient_id, reason)
      log_break_glass_access(user, patient_id)
      grant_temporary_access()
```

## Pattern 3: Right to Deletion (GDPR)

**Use case:** User requests deletion of all personal data

**Implementation:**
```yaml
# Deletion request handler
on_deletion_request:
  verify:
    - User identity confirmed (prevent malicious deletion)
    - Check for legal basis to retain (e.g., outstanding financial obligations)

  if can_delete:
    create_deletion_task:
      - Mark user as "pending_deletion"
      - Grace period: 30 days (allow cancellation)
      - Schedule deletion job

    deletion_job:
      - Delete from all systems (see checklist above)
      - Log deletion with timestamp, user_id, systems deleted
      - Send deletion confirmation email
      - Retain minimal audit trail (user_id, deletion_date, requestor)

  else:
    notify_user:
      - Reason for retention
      - Expected deletion date
      - How to appeal
```

## Pattern 4: Data Export (GDPR Portability)

**Use case:** User requests export of all their personal data

**Implementation:**
```yaml
# Export request handler
on_export_request:
  verify:
    - User identity confirmed

  collect_data:
    - User profile (name, email, phone)
    - User preferences and settings
    - User content (posts, comments, uploads)
    - Usage history (if stored)
    - Consent records

  format:
    - JSON or CSV (machine-readable)
    - Include data dictionary explaining fields

  deliver:
    - Encrypted download link (expire in 7 days)
    - Email notification
    - Log export request (user_id, timestamp, data_scope)
```

---

# Integration with SDD

**Compliance Engineer workflow:**
1. Read `specs.md` and `tasks.yaml`
2. Identify all PHI/PII data flows
3. Create `compliance.md` with controls and acceptance checks
4. Update `specs.md` with compliance acceptance checks
5. If specs conflict with compliance, create `spec-change-requests.yaml` and stop

**Compliance Auditor workflow:**
1. Read `compliance.md` for required controls
2. Review PR diff against compliance requirements
3. Verify PHI/PII handling, encryption, audit logging, access control
4. Create remediation tickets in `tasks.yaml` for gaps
5. Write verdict to `checks.yaml` (blocking if non-compliant)

**Developer workflow:**
1. Read `compliance.md` before implementing feature
2. Implement required technical controls
3. Add tests for compliance checks
4. Never log PHI/PII content
5. Request compliance audit before merge

---

# Example: Patient Records Feature

**Scenario:** Building a feature to create, view, and update patient medical records

**compliance.md output:**

```markdown
# Compliance: Patient Medical Records

## Overview
Feature allows healthcare providers to create, view, and update patient medical records including diagnoses, treatments, and lab results.

## Applicable Regulations
- [x] HIPAA
- [ ] GDPR (if serving EU patients)
- [ ] CCPA
- [ ] SOC2

---

## Data Classification

### PHI
- `patient.name`: Full legal name — encrypt at rest, never log, TLS 1.3 in transit
- `patient.ssn`: Social security number — field-level encryption, never log, restricted access
- `patient.dob`: Date of birth — encrypt at rest, never log
- `record.diagnosis`: Medical diagnosis — encrypt at rest, never log
- `record.treatment_plan`: Treatment details — encrypt at rest, never log
- `record.lab_results`: Lab test results — encrypt at rest, never log

### PII
- `provider.email`: Provider email — encrypt at rest, mask in logs

### Public
- `patient.id`: Internal patient identifier — no special handling
- `record.id`: Internal record identifier — no special handling
- `record.created_at`: Timestamp — standard handling

---

## Technical Controls

### Encryption
- **At rest**: AES-256 database encryption (TDE) + field-level encryption for SSN
- **In transit**: TLS 1.3 enforced on all endpoints
- **Key management**: AWS KMS with annual key rotation

### Access Control
- **Authentication**: Multi-factor authentication (MFA) required for remote access
- **Authorization**: RBAC with roles: provider, nurse, admin
- **Minimum necessary**: Row-level security (RLS) enforces care team membership
- **Break-glass access**: Emergency access with full audit + alert to security team

### Audit Trail
- **Events logged**: All record access (view, create, update, delete, export)
- **Log content**: user_id, timestamp, action, patient_id, record_id, outcome
- **Log storage**: AWS CloudWatch with 7-year retention (California requirement)
- **Log protection**: Immutable logs, encrypted at rest

### Data Retention
- **Retention period**: 7 years after last patient visit (California requirement)
- **Deletion process**: Automated job marks records for deletion after 7 years, manual review, then secure deletion
- **Backup retention**: 7 years, then cryptographic erasure

---

## Business Associate Agreements (BAA)

### Third-Party Services Handling PHI
- AWS (hosting): BAA signed, contract ref AWS-BAA-2024
- Twilio (SMS notifications): BAA signed, contract ref TW-BAA-2024-001
- Stripe (billing): No PHI shared, N/A

---

## Breach Notification

### Detection
- Automated monitoring for unusual access patterns (alerting on bulk exports, off-hours access)
- Manual reporting process for staff to report suspected breaches

### Response Plan
1. Contain breach (revoke access, isolate affected systems)
2. Investigate scope (which patients, which data, how accessed)
3. Notify OCR within 60 days (HIPAA requirement)
4. Notify affected patients within 60 days
5. Notify media if >500 patients in same jurisdiction
6. Document breach in breach log
7. Remediate vulnerability

---

## Acceptance Checks

- [ ] All PHI fields encrypted at rest using AES-256
- [ ] TLS 1.3 enforced on all API endpoints handling PHI
- [ ] Audit log entry created for every patient record access
- [ ] No PHI content appears in application logs (verified via log search)
- [ ] Row-level security enforces care team membership for record access
- [ ] Break-glass access triggers security alert
- [ ] MFA required for all provider accounts
- [ ] Data retention policy automated (7 years, then secure deletion)
- [ ] BAA signed with all third parties handling PHI

---

## References
- Spec: `.ops/build/v1/patient-records/specs.md`
- System design: `.ops/build/v1/patient-records/system-design.yaml`
- Tasks: `.ops/build/v1/patient-records/tasks.yaml`
```

---

# Commands

- `/compliance-patterns:classify` — Classify data fields as PHI/PII/public
- `/compliance-patterns:audit-check` — Generate compliance audit checklist
- `/compliance-patterns:template` — Create compliance.md from template
