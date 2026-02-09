# Data Classification Reference

Field-level examples and handling rules for PHI, PII, and public data across common application contexts.

---

## Healthcare Applications (HIPAA)

### Patient Information

| Field | Classification | Handling Rules | Notes |
|-------|----------------|----------------|-------|
| `patient.id` | Internal ID | No special handling | Safe to log if not linkable to identity |
| `patient.name` | PHI | Encrypt at rest, never log, TLS 1.3 in transit | Individually identifiable |
| `patient.ssn` | PHI (Sensitive) | Field-level encryption, never log, restricted access | High-risk identifier |
| `patient.dob` | PHI | Encrypt at rest, never log | Combined with other fields = identifiable |
| `patient.email` | PHI | Encrypt at rest, mask in logs (`j***@example.com`) | Electronic contact |
| `patient.phone` | PHI | Encrypt at rest, mask in logs (`***-***-1234`) | Electronic contact |
| `patient.address` | PHI | Encrypt at rest, never log | Geographic identifier |
| `patient.zip_code` | PHI (if <20k population) | Encrypt at rest, truncate to 3 digits if needed | Small populations identifiable |
| `patient.medical_record_number` | PHI | Encrypt at rest, never log, restricted access | Direct identifier |
| `patient.insurance_member_id` | PHI | Encrypt at rest, never log | Health plan beneficiary number |
| `patient.photo` | PHI | Encrypt at rest, never log URL | Biometric identifier |
| `patient.ip_address` | PHI | Hash or truncate, minimal retention | Electronic identifier |

### Medical Records

| Field | Classification | Handling Rules | Notes |
|-------|----------------|----------------|-------|
| `record.id` | Internal ID | No special handling | Not linked to patient without query |
| `record.diagnosis` | PHI | Encrypt at rest, never log | Health information |
| `record.treatment_plan` | PHI | Encrypt at rest, never log | Health information |
| `record.medications` | PHI | Encrypt at rest, never log | Health information |
| `record.lab_results` | PHI | Encrypt at rest, never log | Health information |
| `record.provider_notes` | PHI | Encrypt at rest, never log | Health information |
| `record.created_at` | Metadata | Standard handling | Not PHI unless combined |
| `record.updated_at` | Metadata | Standard handling | Not PHI unless combined |

### Provider Information

| Field | Classification | Handling Rules | Notes |
|-------|----------------|----------------|-------|
| `provider.id` | Internal ID | No special handling | Not PHI |
| `provider.name` | PII | Encrypt at rest, can log (no PHI linkage) | Professional information |
| `provider.email` | PII | Encrypt at rest, mask in logs | Contact information |
| `provider.npi` | Public | Standard handling | National Provider Identifier is public |
| `provider.license_number` | PII | Encrypt at rest, restricted access | Professional credential |

---

## E-Commerce / General Applications (GDPR/CCPA)

### User Account

| Field | Classification | Handling Rules | Notes |
|-------|----------------|----------------|-------|
| `user.id` | Internal ID | No special handling | Not PII if UUID/random |
| `user.username` | PII | Encrypt at rest, can log (if not real name) | Could be pseudonymous |
| `user.email` | PII | Encrypt at rest, mask in logs | Direct identifier |
| `user.password_hash` | Sensitive | Encrypt at rest, never log | Security credential |
| `user.name` | PII | Encrypt at rest, never log | Direct identifier |
| `user.phone` | PII | Encrypt at rest, mask in logs | Direct identifier |
| `user.dob` | PII | Encrypt at rest, never log | Sensitive personal data (GDPR special category if age <16) |
| `user.ip_address` | PII (GDPR) | Hash or truncate, minimal retention | Online identifier |
| `user.device_id` | PII (GDPR) | Hash, minimal retention | Online identifier |
| `user.user_agent` | Metadata | Standard handling, can log | Not PII unless combined |

### Address / Location

| Field | Classification | Handling Rules | Notes |
|-------|----------------|----------------|-------|
| `address.street` | PII | Encrypt at rest, never log | Geographic identifier |
| `address.city` | PII | Encrypt at rest, can log in aggregate | Identifying with other fields |
| `address.state` | Metadata | Standard handling | Not PII alone |
| `address.zip_code` | PII | Encrypt at rest | Geographic identifier |
| `address.country` | Metadata | Standard handling | Not PII alone |
| `geolocation.lat_long` | PII | Encrypt at rest, never log exact coordinates | Precise location = identifiable |

### Payment Information

| Field | Classification | Handling Rules | Notes |
|-------|----------------|----------------|-------|
| `payment.card_number` | Sensitive PII | Never store full number, tokenize, PCI-DSS scope | Use payment processor tokenization |
| `payment.card_last4` | PII | Can store, mask in logs | Not full card number |
| `payment.card_brand` | Metadata | Standard handling | Not PII (Visa, Mastercard, etc.) |
| `payment.cvv` | Sensitive PII | NEVER store (PCI-DSS violation) | Must not persist |
| `payment.billing_address` | PII | Encrypt at rest, never log | Geographic identifier |
| `payment.stripe_customer_id` | Internal ID | No special handling | Tokenized reference |

### Order / Transaction

| Field | Classification | Handling Rules | Notes |
|-------|----------------|----------------|-------|
| `order.id` | Internal ID | No special handling | Not PII if random |
| `order.user_id` | Internal ID | No special handling | Links to user but not PII itself |
| `order.items` | Metadata | Standard handling | Product list not PII |
| `order.total` | Metadata | Standard handling | Amount not PII |
| `order.created_at` | Metadata | Standard handling | Timestamp not PII |
| `order.shipping_address` | PII | Encrypt at rest, never log | Geographic identifier |

---

## Financial Applications (GLBA, SOX, PCI-DSS)

### Account Information

| Field | Classification | Handling Rules | Notes |
|-------|----------------|----------------|-------|
| `account.id` | Internal ID | No special handling | Not PII if random |
| `account.account_number` | Sensitive PII | Encrypt at rest, never log, PCI-DSS scope | Financial account |
| `account.routing_number` | Sensitive PII | Encrypt at rest, never log | Financial institution identifier |
| `account.balance` | PII | Encrypt at rest, never log | Financial information |
| `account.ssn` | Sensitive PII | Field-level encryption, never log | Tax identifier |
| `account.tax_id` | Sensitive PII | Field-level encryption, never log | Business tax identifier |

### Transaction Data

| Field | Classification | Handling Rules | Notes |
|-------|----------------|----------------|-------|
| `transaction.id` | Internal ID | No special handling | Not PII if random |
| `transaction.amount` | PII | Encrypt at rest, can log in aggregate | Financial activity |
| `transaction.type` | Metadata | Standard handling | Not PII (debit, credit) |
| `transaction.timestamp` | Metadata | Standard handling | Not PII unless precise timing reveals identity |
| `transaction.merchant` | Metadata | Standard handling | Not PII |

---

## SaaS / Workplace Applications

### Workspace / Organization

| Field | Classification | Handling Rules | Notes |
|-------|----------------|----------------|-------|
| `workspace.id` | Internal ID | No special handling | Not PII |
| `workspace.name` | Metadata | Standard handling | Organization name, not PII |
| `workspace.domain` | Metadata | Standard handling | Company domain, not PII |

### User / Employee

| Field | Classification | Handling Rules | Notes |
|-------|----------------|----------------|-------|
| `employee.id` | Internal ID | No special handling | Not PII if random |
| `employee.name` | PII | Encrypt at rest, can log (workplace context) | Work-related PII |
| `employee.work_email` | PII | Encrypt at rest, can log | Work contact |
| `employee.personal_email` | PII | Encrypt at rest, never log | Personal contact |
| `employee.title` | Metadata | Standard handling | Not PII |
| `employee.department` | Metadata | Standard handling | Not PII |
| `employee.salary` | PII | Encrypt at rest, never log, restricted access | Sensitive financial data |
| `employee.ssn` | Sensitive PII | Field-level encryption, never log, highly restricted | Tax/benefits identifier |
| `employee.emergency_contact` | PII | Encrypt at rest, never log | Third-party PII |

---

## Special Categories (GDPR Article 9 - Extra Protections Required)

### Health Data
- Medical diagnoses, treatments, prescriptions, lab results
- Mental health records
- Genetic data, biometric data (fingerprints, facial recognition)
- **Handling**: Explicit consent required, extra encryption, access logging, DPO involvement

### Racial/Ethnic Origin
- Race, ethnicity, national origin fields
- **Handling**: Explicit consent, limited to specific lawful purposes, avoid unless necessary

### Religious/Philosophical Beliefs
- Religious affiliation, political beliefs
- **Handling**: Explicit consent, avoid collection unless critical

### Sexual Orientation / Sex Life
- Relationship status, gender identity, sexual orientation
- **Handling**: Explicit consent, extra protections, sensitive category

### Biometric Data
- Fingerprints, facial recognition, iris scans, voice prints
- **Handling**: Explicit consent, encrypt, restricted access, log all use

### Criminal Records
- Arrests, convictions, criminal history
- **Handling**: Legal basis required, government authority often needed

---

## Decision Matrix: Is This Field PHI/PII?

### Is it PHI? (HIPAA)
**Test**: Does it relate to health AND can it identify an individual?

- Health information alone = **Not PHI** (e.g., "diabetes" without linking to person)
- Identifier alone = **Not PHI** (e.g., "John Doe" without health context)
- Health + identifier = **PHI** (e.g., "John Doe has diabetes")

**18 HIPAA identifiers:**
1. Names
2. Geographic subdivisions smaller than state (including street address, city, county, ZIP if <20k population)
3. Dates directly related to an individual (birth, admission, discharge, death)
4. Phone numbers
5. Fax numbers
6. Email addresses
7. Social security numbers
8. Medical record numbers
9. Health plan beneficiary numbers
10. Account numbers
11. Certificate/license numbers
12. Vehicle identifiers and serial numbers
13. Device identifiers and serial numbers
14. URLs
15. IP addresses
16. Biometric identifiers (fingerprints, voiceprints)
17. Full-face photos
18. Any other unique identifying number, characteristic, or code

### Is it PII? (GDPR/CCPA)
**Test**: Can it identify an individual, alone or combined with other data?

**Direct identifiers** (PII alone):
- Name, email, phone, SSN, driver's license, passport number

**Indirect identifiers** (PII when combined):
- DOB + ZIP code + gender = 87% re-identification rate
- IP address + user agent + timestamp = trackable across sessions
- Device ID + location history = identifiable

**Not PII**:
- Aggregated data (no individual records)
- Fully anonymized data (irreversible, no re-identification possible)
- Pseudonymous data IF no linkage table exists (still PII under GDPR if reversible)

---

## Logging Decision Tree

```
Is this field PHI/PII?
├─ No → Safe to log
├─ Yes → Is it necessary for debugging/audit?
   ├─ No → Don't log
   ├─ Yes → Can you log a hashed/masked version?
      ├─ Yes → Log hash or mask (e.g., email_hash, last 4 digits)
      ├─ No → Log only resource ID, never content
```

**Examples:**
- `user_id: 12345` → **Safe** (ID, not identity)
- `email: "user@example.com"` → **Unsafe** (PII)
- `email_hash: "sha256(...)"` → **Safe** (if needed for correlation)
- `email_masked: "u***@example.com"` → **Acceptable** (for support)
- `patient_name: "John Doe"` → **NEVER** (PHI)
- `patient_id: 67890` → **Safe** (ID, not identity)
- `diagnosis: "diabetes"` → **NEVER** (PHI)
- `record_updated: true` → **Safe** (action, no content)

---

## Anonymization vs. Pseudonymization

### Anonymization (Irreversible - Not PII/PHI)
- **Definition**: Data altered so re-identification is impossible
- **Techniques**:
  - Aggregation (e.g., "500 patients with diabetes" not individual records)
  - Generalization (e.g., age ranges "30-40" instead of exact age)
  - Data suppression (remove unique identifiers)
- **GDPR status**: Not personal data if truly irreversible
- **Use case**: Public reporting, research datasets, analytics

### Pseudonymization (Reversible - Still PII/PHI)
- **Definition**: Replace identifiers with pseudonyms, but linkage exists
- **Techniques**:
  - Tokenization (e.g., replace SSN with token, store mapping securely)
  - Hashing with salt (can verify but not reverse without salt)
  - Encryption (can decrypt with key)
- **GDPR status**: Still personal data, but reduces risk (Article 32)
- **Use case**: Internal processing, database storage, third-party sharing with safeguards

**Important**: Pseudonymization does NOT remove PHI/PII classification. It reduces risk but still requires compliance.

---

## Quick Reference: Common Fields

| Field | Healthcare | E-Commerce | Financial | SaaS |
|-------|-----------|-----------|-----------|------|
| Name | PHI | PII | PII | PII |
| Email | PHI | PII | PII | PII |
| Phone | PHI | PII | PII | PII |
| Address | PHI | PII | PII | PII |
| DOB | PHI | PII | PII | PII |
| SSN | PHI (Sensitive) | Sensitive PII | Sensitive PII | Sensitive PII |
| IP Address | PHI | PII (GDPR) | PII | PII (GDPR) |
| Credit Card | N/A | Sensitive PII (PCI-DSS) | Sensitive PII (PCI-DSS) | Sensitive PII (PCI-DSS) |
| Medical Info | PHI | N/A | N/A | N/A |
| Account Number | N/A | N/A | Sensitive PII | N/A |
| User ID (UUID) | Safe to log | Safe to log | Safe to log | Safe to log |
| Timestamps | Safe to log | Safe to log | Safe to log | Safe to log |

---

## When to Escalate

**Ask compliance/legal when:**
- Unsure if a field is PHI/PII in your specific context
- Combining multiple fields that might create identifiability
- Sharing data with third parties (BAA/DPA required?)
- International data transfers (GDPR adequacy decisions)
- Special category data (health, biometric, racial, religious)
- Minors' data (COPPA, GDPR special protections)
- Data breach involving any PHI/PII (immediate escalation)

**Do NOT:**
- Guess at classification if unsure (over-classify to be safe)
- Downgrade classification without documented approval
- Skip compliance review for "small" features (all PHI/PII handling needs review)
