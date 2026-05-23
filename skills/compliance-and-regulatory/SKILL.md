---
name: compliance-and-regulatory
description: Embeds GDPR, SOC 2 Type II, and ISO 27001 compliance into development from day one. Use when building any system that handles personal data, requires security certification, or is sold to enterprise customers.
---

# Compliance and Regulatory

## Overview

Build compliance in from the start, not as a retrofit. Systems that handle personal data or require security certification must satisfy GDPR, SOC 2 Type II, or ISO 27001 — often all three simultaneously. Compliance added after the fact costs ten times more than compliance built in: schema migrations, retroactive consent collection, emergency audit-log pipelines, and emergency deletion workflows are all avoidable if you treat compliance as a first-class design constraint.

**The core discipline:** Every feature that touches personal data gets a Privacy Impact Assessment before a line of code is written. Every data field gets a classification before it is stored. Every control gets verified, not just implemented.

GDPR, SOC 2, and ISO 27001 overlap significantly. The shared controls section identifies what you build once to satisfy all three.

## When to Use

Activate this skill for any system that:

- Handles personal data of any kind (names, emails, behavioral data, location)
- Requires a security certification for enterprise sales
- Processes employee or customer data
- Is a B2B product sold to enterprise customers
- Introduces any new feature touching Level 3 or Level 4 data (see classification below)

**When NOT to apply full rigor:**
- Internal tooling with no personal data and no external users
- Prototypes or throwaway experiments — but migrate to compliant patterns before production

**The compliance trigger test:**

> "Does this system store, process, or transmit data about a real, identifiable person?"
>
> - No → standard security practices apply
> - Yes → this skill applies in full

When in doubt, apply the skill. Discovering an overlooked compliance requirement during an audit is far more expensive than applying the skill unnecessarily.

## Data Classification

Classify every piece of data before storing it. Classification drives every downstream control decision. No field is exempt — including fields in logs, analytics pipelines, and error tracking.

### Level 1 — Public

**Definition:** Safe to share publicly without restriction.

**Examples:** Product names, public pricing, blog post content, marketing copy.

**Controls required:** None beyond standard security hygiene.

### Level 2 — Internal

**Definition:** For internal use only; not sensitive if disclosed but not intended for the public.

**Examples:** Internal documentation, meeting notes, internal metrics, employee directories.

**Controls required:** Authentication required; basic access control.

### Level 3 — Confidential

**Definition:** Sensitive business data with meaningful harm potential if disclosed.

**Examples:** Visitor schedules, partner contacts, business strategies, employee performance data, aggregate user behavior.

**Controls required:** Encryption at rest and in transit; audit logging of all access; role-based access control; need-to-know basis only.

### Level 4 — Restricted

**Definition:** Highly sensitive data with direct regulatory implications.

**Examples:** PII (names, emails, phone numbers, addresses), financial data, health data, authentication credentials, government IDs, precise location data.

**Controls required:** All Level 3 controls, plus: additional encryption layers; strict access logging with alerting on anomalous access; data masking in non-production environments; explicit consent documented before collection; retention period defined and enforced by automation.

**Classify before you store.** If you are unsure whether a field is Level 3 or Level 4, treat it as Level 4 until proven otherwise.

## Privacy by Design

Seven principles govern every system that handles personal data. These are not aspirational — they are design constraints that must be satisfied before a feature ships.

**1. Proactive, not reactive.** Anticipate privacy risks and prevent them before they occur. Do not wait for a breach, complaint, or audit to surface a gap. Run PIAs before building, not after.

**2. Privacy as the default.** The most privacy-protective option is the default. Users should not have to take any action to protect their privacy — the system should protect it without any configuration. Opt-in beats opt-out; collect-nothing beats collect-and-discard.

**3. Privacy embedded in design.** Privacy is not a layer added on top of a system — it is part of the architecture. Data minimization, access control, and retention automation must be designed at the schema level, not bolted on as middleware.

**4. Full functionality — not privacy vs. security.** Privacy and security are not in tension. Designing a system that is secure but leaks personal data, or private but insecure, is a failure. Both must be fully satisfied; neither can be traded off against the other.

**5. End-to-end security.** Security must hold for the full lifecycle of the data: collection, storage, processing, transmission, and deletion. A secure API that stores data in an unencrypted database is not end-to-end secure.

**6. Visibility and transparency.** Users must be able to see what data is collected, why, and how it is used. Privacy policies must be accurate, current, and written in plain language. Data maps must be maintained and available for audit.

**7. Respect for user privacy.** User interests come first. If there is any doubt about whether a data practice respects the user, resolve the doubt in the user's favor.

## The Compliance Workflow

Run this workflow for every feature that touches personal data. Do not begin implementation until steps 1–4 are complete.

### Step 1: Data Mapping

Before writing any code, map every piece of personal data the feature will touch:

```
DATA MAP — [Feature Name]

Field         | Classification | Source        | Storage       | Retention
------------- | -------------- | ------------- | ------------- | ---------
email         | Level 4 (PII)  | User input    | users table   | Until deletion request
ip_address    | Level 4 (PII)  | Request header| access_log    | 90 days, then purge
page_views    | Level 3        | Analytics SDK | events table  | 2 years
session_token | Level 4        | Generated     | sessions table| 30 days inactive
```

If you cannot complete this map, the feature is not ready to design.

### Step 2: Classification

Assign a data level (1–4) to every field identified in step 1. Escalate to the highest applicable level — if a field combination enables re-identification, classify the combination at Level 4 even if individual fields are Level 3.

### Step 3: Privacy Impact Assessment

Complete a PIA for every feature touching Level 3 or Level 4 data. Use the template in the PIA section below. The PIA is not optional — it is the compliance equivalent of a spec.

### Step 4: Control Implementation

Implement the controls required by the classification level before writing any business logic:

- Schema: add classification comments to Level 3+ columns
- Encryption: configure at-rest encryption for Level 3+ tables
- Access control: restrict Level 3+ data to roles that need it
- Audit logging: instrument all access to Level 4 data
- Retention: add automated purge jobs for all Level 4 fields

### Step 5: Security Review

Before merging any feature touching Level 3+ data, route it through `skills/security-and-hardening/SKILL.md` (`security-and-hardening`) with compliance context. The reviewer must confirm: no PII in logs, deletion endpoint works, export endpoint works, retention automation is in place.

### Step 6: Documentation Update

After merge, update:
- The data map (add new fields)
- The privacy policy (if new data types are collected)
- The third-party processor list (if new vendors receive data)
- The data flow diagram (if architecture changed)

Compliance documentation that is out of date is a liability, not an asset.

## GDPR Requirements

GDPR applies to any system handling personal data of EU residents, regardless of where the system is hosted or the company is incorporated.

### Lawful Basis

Every personal data field must have a documented lawful basis for collection. There are six bases; the two most common in software:

| Basis | When to use | What it requires |
|-------|-------------|-----------------|
| **Consent** | Marketing, optional features, analytics | Freely given, specific, informed, unambiguous. Must be withdrawable at any time. |
| **Legitimate interests** | Fraud prevention, security, product improvement | Must be balanced against user rights; document the balancing test. |
| **Contract** | Data required to deliver the service | Collect only what is strictly necessary to perform the contract. |

Document the basis for every field in the data map. "We need it" is not a lawful basis.

### Individual Rights Implementation

These are not optional features — they are legal obligations:

| Right | Implementation required |
|-------|------------------------|
| **Access** | `/api/users/me/export` — returns all personal data in machine-readable format (JSON or CSV) within 30 days of request |
| **Erasure** | `/api/users/me/delete` — removes or anonymizes all personal data; soft-delete with scheduled hard-delete is acceptable if the schedule is enforced |
| **Rectification** | User can correct inaccurate data |
| **Portability** | Export in a standard, machine-readable format |
| **Objection** | User can object to processing based on legitimate interests |
| **Restriction** | User can pause processing while a dispute is resolved |

### Data Minimisation and Purpose Limitation

Collect only what is necessary for the stated purpose. If a field is collected "just in case," remove it. If a field is used for a different purpose than the one disclosed, that is a GDPR violation — update the privacy policy or stop the processing.

### Storage Limitation

Every personal data field must have a defined retention period. Retention periods must be enforced by automation — a policy document with no automation is not compliance.

```
Retention schedule example:
- Session data:     30 days after last activity
- User account:     Until erasure request + 30-day grace
- Audit logs:       7 years (legal hold)
- Analytics events: 2 years
- Error logs:       90 days (must be stripped of PII first)
```

### Data Breach Notification

A personal data breach must be reported to the supervisory authority within **72 hours** of discovery. Document the incident response procedure before you need it, not during an incident.

### GDPR Implementation Checklist

```
□ Privacy policy written, accurate, and linked from the product
□ Every personal data field has a documented lawful basis
□ Consent mechanism implemented where consent is the lawful basis
□ Consent withdrawal mechanism implemented
□ Data export endpoint exists and has been tested
□ Data deletion endpoint exists and has been tested
□ Retention periods defined for every personal data field
□ Automated purge jobs implemented and monitored
□ Third-party data processors documented (name, purpose, data shared)
□ Data breach response procedure documented and team-tested
□ Data Protection Officer designated (if required by volume or type)
```

## SOC 2 Type II Requirements

SOC 2 Type II certifies that security controls were operating effectively over an observation period (typically 6–12 months). Type II is what enterprise buyers require — Type I only certifies that controls exist at a point in time.

### Trust Service Criteria

| Criteria | What it covers | Key controls |
|----------|---------------|-------------|
| **Security (CC)** | Protection from unauthorized access, disclosure, and damage | Access control, MFA, encryption, vulnerability management, incident response |
| **Availability (A)** | System available as committed in SLAs | Uptime monitoring, redundancy, DR testing, capacity planning |
| **Processing Integrity (PI)** | Processing is complete, valid, accurate, timely | Input validation, error handling, reconciliation, audit trails |
| **Confidentiality (C)** | Confidential information protected per agreements | Data classification, NDA enforcement, encryption, access control |
| **Privacy (P)** | Personal information handled per the privacy notice | Overlaps significantly with GDPR requirements |

Security (CC) is mandatory. The remaining four criteria are selected based on what your product promises to customers.

### SOC 2 Implementation Checklist

```
□ Access control: least-privilege principle applied to all systems
□ MFA enforced for all admin, production, and cloud console access
□ Encryption: AES-256 at rest; TLS 1.2+ in transit (TLS 1.3 preferred)
□ Audit logging: who did what, when, from where — immutable and retained
□ Vulnerability scanning: automated scans on every build and weekly in prod
□ Penetration testing: annual third-party test; findings tracked to closure
□ Incident response plan: documented, assigned, and tested via tabletop
□ Change management: all production changes reviewed and approved before deploy
□ Vendor assessment: all third-party services assessed for security posture
□ Business continuity: backup and recovery tested at least annually
□ Employee offboarding: access revoked within 24 hours of departure
□ Security awareness training: all staff trained annually
□ Logical access reviews: quarterly review of who has access to what
```

### Evidence Collection

SOC 2 Type II requires evidence that controls operated throughout the audit period — not just that they exist. Every control above must produce evidence:

| Control | Evidence type |
|---------|--------------|
| MFA enforcement | IAM policy configuration + audit of logins |
| Vulnerability scanning | Scan reports with remediation timelines |
| Penetration testing | Third-party report + finding closure evidence |
| Access reviews | Quarterly review sign-offs |
| Incident response | Incident tickets with response timelines |

Start collecting evidence from day one of the observation period. Retroactive evidence is not accepted.

## ISO 27001 Requirements

ISO 27001 is a certification for an Information Security Management System (ISMS) — the policies, processes, and controls that govern how an organisation manages information security risk.

### Core Domains

| Domain | What it governs |
|--------|----------------|
| Information security policies | The written policy framework governing security decisions |
| Risk assessment and treatment | Systematic identification, evaluation, and treatment of security risks |
| Asset inventory and classification | What information assets exist, who owns them, and how they are classified |
| Human resources security | Pre-employment screening, security training, offboarding |
| Physical and environmental security | Physical access to facilities and equipment |
| Access control | Who can access what systems, data, and facilities |
| Cryptography | Standards for encryption, key management, and certificate lifecycle |
| Operations security | Secure operations procedures, capacity management, logging, malware protection |
| Communications security | Network security, information transfer policies |
| Supplier relationships | Security requirements for third-party services and supply chain |

### ISO 27001 Implementation Checklist

```
□ Information security policy: documented, approved by leadership, communicated to staff
□ Risk register: all identified risks documented with likelihood, impact, and treatment
□ Risk reviews: risk register reviewed and updated at least quarterly
□ Asset inventory: all information assets catalogued with owner and classification
□ Access control policy: implemented and enforced; reviewed quarterly
□ Cryptography policy: defines approved algorithms, key lengths, and rotation schedules
□ Security awareness training: all staff trained at onboarding and annually
□ Incident management procedure: documented with roles, escalation paths, and SLAs
□ Business continuity plan: documented and tested; RTO/RPO defined per system
□ Supplier security assessment: all key suppliers assessed before onboarding
□ Internal audit: ISMS audited at least annually
□ Management review: senior leadership reviews ISMS effectiveness annually
```

## Shared Controls

These controls, implemented once, satisfy requirements across GDPR, SOC 2, and ISO 27001. Prioritise these first — they deliver the highest compliance return per unit of engineering effort.

| Control | GDPR | SOC 2 | ISO 27001 |
|---------|------|-------|-----------|
| **Encryption at rest and in transit** | Art. 32 | CC6.1 | A.10.1 |
| **Access control with least privilege** | Art. 25 | CC6.2 | A.9 |
| **Audit logging of all data access and changes** | Art. 5(2) | CC7.2 | A.12.4 |
| **Incident response procedure** | Art. 33–34 | CC7.4 | A.16 |
| **Data classification and handling** | Art. 4, 9 | CC3.2 | A.8 |
| **Vendor / third-party assessment** | Art. 28 | CC9.2 | A.15 |
| **Regular security reviews** | Art. 32 | CC4.1 | A.18 |
| **Employee security training** | Art. 39 | CC1.4 | A.7.2 |

Implement shared controls before framework-specific ones.

## Privacy Impact Assessment Template

Run a PIA for every new feature that touches Level 3 or Level 4 data. The PIA is completed before implementation begins — not before it ships.

```markdown
# Privacy Impact Assessment — [Feature Name]

**Date:** YYYY-MM-DD
**Author:** [Name / role]
**Status:** [Draft | Reviewed | Approved]

## 1. Data Collected

| Field | Classification | Lawful Basis |
|-------|---------------|-------------|
| [field name] | [Level 1–4] | [Consent / Contract / Legitimate interest] |

## 2. Purpose of Collection

[Why is each field collected? Be specific. "To send the user their order
confirmation" is acceptable. "For future use" is not.]

## 3. Storage

- Location: [database, region, cloud provider]
- Encryption at rest: [yes/no — if yes, algorithm and key management]
- Encryption in transit: [TLS version]
- Access control: [which roles can access, how access is granted]

## 4. Access

[Who can access this data? List roles, not individuals. Document the
approval process for granting access above the default level.]

## 5. Retention

| Field | Retention period | Deletion method | Automation |
|-------|-----------------|-----------------|-----------|
| [field] | [e.g., 2 years] | [hard delete / anonymise] | [job name / cron] |

## 6. Deletion

[How is data deleted when the retention period expires or a deletion
request is received? Describe the mechanism, not just the policy.]

## 7. Third Parties

| Party | Data shared | Purpose | DPA in place? |
|-------|-------------|---------|--------------|
| [vendor] | [fields] | [why] | [yes/no] |

## 8. Risks to Individuals

| Risk | Likelihood | Severity | Mitigation |
|------|-----------|----------|------------|
| [e.g., unauthorised access to emails] | [High/Med/Low] | [High/Med/Low] | [control] |

## 9. Risk Mitigation

[For each risk above, describe the specific technical or procedural control
that reduces likelihood or severity to acceptable levels.]

## 10. Approval

- [ ] Engineering lead reviewed
- [ ] Privacy / legal reviewed (required for Level 4 data)
- [ ] PIA approved — implementation may proceed
```

## Code-Level Requirements

These rules apply to every line of code that processes personal data. They are non-negotiable — each one corresponds to a real breach pattern.

**Logging**
- Never log personal data: no emails, names, user IDs, IP addresses, session tokens, or any Level 4 field
- Strip PII from error messages before they are logged or returned to the client
- If you need to correlate logs to a user, log an opaque internal ID — never the identifier itself

**URLs**
- Never put personal data in URLs: no email addresses, names, or readable IDs in path or query parameters
- Use POST body or short-lived tokens for operations that reference personal data

**Secrets**
- Secrets never appear in source code — use environment variables or a secrets manager
- Never commit `.env` files; `.env.example` with placeholder values only
- Rotate secrets immediately if they appear in version control history

**Database**
- Mark PII columns in schema comments: `-- PII: Level 4 — subject to GDPR erasure`
- Use parameterized queries everywhere — no string concatenation in SQL
- Mask Level 4 data in non-production environments: anonymise seed data, never copy production PII to staging

**Deletion**
- Prefer soft delete (set `deleted_at`) over hard delete to preserve audit trail
- Automated hard delete or anonymisation runs on schedule after retention period
- Deletion cascade must cover all tables that store the data — test with a real deletion request

**Authentication and authorisation**
- All API endpoints that return personal data require authentication — no unauthenticated PII endpoints
- Rate-limit all authentication endpoints to prevent credential stuffing
- Validate and sanitise all user-supplied input at system boundaries

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| "We'll add compliance later when we need it" | Compliance retrofits average 10× the cost of building it in. Schema migrations, retroactive consent, and emergency audit pipelines are expensive and risky. |
| "Our users are not in the EU so GDPR doesn't apply" | GDPR applies to the data of EU residents regardless of where the company or system is located. One EU customer triggers full applicability. |
| "We log everything — it helps debugging" | Full request/response logging captures PII and is a data breach waiting to happen. Log structured events with opaque IDs. |
| "Compliance is a legal problem, not an engineering problem" | Legal writes the policies; engineering implements and verifies the controls. Unimplemented controls are violations regardless of what the policy says. |
| "We passed the SOC 2 audit — we're done" | SOC 2 Type II is a continuous programme. Controls must operate throughout the audit window every year. |
| "We use a third-party auth provider so we don't have PII" | Your system stores the user ID from that provider and links it to their behaviour. That combination is PII. |

## Red Flags

- A new feature stores an email address with no documented lawful basis
- PII appears in application logs, error messages, or Sentry/Datadog traces
- No data export or deletion endpoint exists in a system with registered users
- Non-production environments use production data (even anonymised dumps that weren't fully sanitised)
- Access to production personal data is not logged
- A single shared admin account is used by multiple engineers
- Third-party services receive personal data with no Data Processing Agreement
- Retention policy exists in a document but no automated purge job enforces it
- Encryption is described in the architecture doc but not verified in the deployed configuration
- Security certifications are treated as one-time projects rather than continuous programmes

## Verification

Before shipping any feature that touches personal data:

```
□ Every personal data field classified (Level 1–4) in the data map
□ PIA completed and approved for Level 3–4 data
□ All shared controls implemented (encryption, access control, audit logging)
□ Lawful basis documented for every Level 4 field collected
□ No PII present in logs, URLs, or client-facing error messages
□ Data export endpoint implemented and tested end-to-end
□ Data deletion endpoint implemented and tested end-to-end
□ Retention period defined and automated purge job deployed for every Level 4 field
□ Level 4 data masked in all non-production environments
□ Third-party services receiving data have signed DPAs
□ Privacy policy updated to reflect new data collected
□ Security review completed (see skills/security-and-hardening/SKILL.md)
```

---

## Compliance Quick Reference

**The ten rules developers must internalise.**

1. **Classify before you store.** Every field is Level 1–4. No field is unclassified.
2. **No PII in logs.** Ever. Mask emails, names, IDs, IPs before they reach a log line.
3. **No PII in URLs.** Use POST body or opaque tokens for anything that identifies a person.
4. **No secrets in code.** Environment variables or a secrets manager. No exceptions.
5. **Parameterize every query.** No string concatenation in SQL, ever.
6. **Every endpoint that returns personal data requires authentication.**
7. **Rate-limit every authentication endpoint.**
8. **Collect only what you need.** If you don't know why you're storing a field, don't store it.
9. **Define retention before you store.** Every Level 4 field needs a purge date and an automated job to enforce it.
10. **Test deletion.** A deletion endpoint that silently fails is not a deletion endpoint.

If a PR touches personal data and violates any rule above, it does not merge.

---

## See Also

- For compliance requirements in the spec, see `skills/spec-driven-development/SKILL.md` (`spec-driven-development`) — compliance constraints belong in the spec's Boundaries section
- For API design that avoids unnecessary PII exposure, see `skills/api-and-interface-design/SKILL.md` (`api-and-interface-design`)
- For security controls that overlap with compliance, see `skills/security-and-hardening/SKILL.md` (`security-and-hardening`)
- For compliance sign-off before release, see `skills/shipping-and-launch/SKILL.md` (`shipping-and-launch`)
- For documenting compliance-driven architectural decisions, see `skills/architecture-decisions/SKILL.md` (`architecture-decisions`)
