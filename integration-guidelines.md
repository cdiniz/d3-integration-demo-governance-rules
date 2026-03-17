# Norvik Bank — Integration Development Guidelines

| | |
|---|---|
| **Document ID** | NVK-IDG-2023-001 |
| **Version** | 2.2 |
| **Date** | 8 January 2024 |
| **Owner** | Enterprise Architecture — Integration Standards |
| **Scope** | All integration services exposing or consuming APIs within Norvik Bank |

---

## 1. Purpose

These guidelines define mandatory standards for all integration services developed within Norvik Bank. They apply to every API regardless of the project-specific integration canvas. A canvas defines *what* a service does; these guidelines define *how* it must be built.

Compliance is assessed independently of canvas conformance. A service can be fully canvas-compliant and still fail a guidelines audit — or vice versa. Both checks are required before production release.

---

## 2. Guidelines

### IG-001 — Cryptographic Standards

**All cryptographic operations must use algorithms approved by the Norvik Security Office.**

| Operation | Minimum Standard | Prohibited |
|---|---|---|
| Hashing (integrity, identifiers) | SHA-256 | MD5, SHA-1 |
| Symmetric encryption | AES-256-GCM | DES, 3DES, AES-128 |
| Asymmetric encryption | RSA-2048 / ECDSA P-256 | RSA-1024 |

This applies to all uses including — but not limited to — identifier derivation, checksum generation, token signing, and payload encryption. There are no exceptions for "non-security" uses of hash functions. MD5 is prohibited even for non-cryptographic purposes such as generating opaque identifiers, because its presence in the codebase creates audit risk and normalises the use of deprecated algorithms.

**Audit check:** inspect all usages of `MessageDigest`, hashing libraries, or equivalent. Any reference to MD5 or SHA-1 is a finding.

---

### IG-002 — Error Response Format

**All REST APIs must return error responses in RFC 7807 Problem Details format.**

Every error response — regardless of HTTP status code, error origin, or exception type — must conform to the following structure:

```json
{
  "type": "https://api.norvik.eu/problems/{ERROR_CODE}",
  "title": "Human-readable summary",
  "status": <http_status_code>,
  "detail": "Specific explanation of the error instance."
}
```

The `Content-Type` header for error responses must be `application/problem+json`.

This applies to:
- Application-level errors (business logic, validation)
- Framework-level errors (type mismatches, missing parameters, deserialization failures)
- Infrastructure-level errors (timeouts, circuit breaker open)

No alternative error shapes (e.g. `{"error": ..., "message": ...}`, Spring Boot default `WhitelabelError`, raw exception messages) are permitted.

**Audit check:** send requests that trigger each category of error. Verify every response body contains `type`, `title`, `status`, and `detail` fields.

---

### IG-003 — Sensitive Data Classification

**Fields classified as RESTRICTED or HIGHLY SENSITIVE in the Norvik Data Catalogue must not appear in external API responses.**

The Norvik Data Catalogue assigns a classification to every field in every system of record. Integration services are the enforcement boundary — upstream systems do not filter sensitive data.

| Classification | Examples | Rule |
|---|---|---|
| PUBLIC | Transaction amount, currency, booking date | May appear in API responses |
| INTERNAL | Cost centre, contract ID | Must not appear in external responses |
| RESTRICTED | Customer CRM reference, customer legal name | Must not appear in external responses |
| HIGHLY SENSITIVE | AML flags, sanctions screening results, fraud scores | Must not appear in external responses. Must not be logged. Must not be cached. |

For the Finestra Core `TXN.*` record set, the following classifications apply:

| Field | Classification |
|---|---|
| `TXN.ID` | INTERNAL |
| `TXN.CONTRACT.ID` | INTERNAL |
| `TXN.CUSTOMER.REF` | RESTRICTED |
| `TXN.CUSTOMER.NAME` | RESTRICTED |
| `TXN.COST.CENTER` | INTERNAL |
| `TXN.AML.FLAG` | HIGHLY SENSITIVE |
| `TXN.BOOKING.DATE` | PUBLIC |
| `TXN.VALUE.DATE` | PUBLIC |
| `TXN.AMOUNT` | PUBLIC |
| `TXN.CURRENCY` | PUBLIC |
| `TXN.NARRATIVE` | PUBLIC |
| `TXN.COUNTER.IBAN` | PUBLIC |
| `TXN.COUNTER.NAME` | PUBLIC |
| `TXN.CHANNEL` | PUBLIC |
| `TXN.MANDATE.REF` | PUBLIC |

**Exception — one-way cryptographic derivation:** A field derived from an INTERNAL or RESTRICTED field using a one-way hash function approved under IG-001 (e.g. SHA-256) is not considered an exposure, provided: (a) the derivation is irreversible — no lookup table or reverse mapping exists in the service, (b) the integration spec explicitly mandates the derivation, and (c) the hash input includes a context-specific salt (e.g. account IBAN) to prevent cross-entity correlation. This exception does not apply to HIGHLY SENSITIVE fields, which must never be derived in any form.

**Audit check:** inspect API response schemas and source code for any field that maps to INTERNAL, RESTRICTED, or HIGHLY SENSITIVE data. Any exposure — including renamed, aliased, or derived forms — is a finding, unless the one-way cryptographic derivation exception above applies.

---

## 3. Enforcement

These guidelines are checked automatically by governance agents in the CI/CD pipeline. Findings are reported on every pull request and push to `main`. A service with open findings against IG-001, IG-002, or IG-003 may not be promoted to production without a documented exception approved by the Chief Information Security Officer.

---

## 4. Change History

| Version | Date | Change |
|---|---|---|
| 1.0 | 2022-06-15 | Initial publication |
| 2.0 | 2023-03-20 | Added IG-003 (data classification). Expanded IG-001 to prohibit MD5 for all uses. |
| 2.1 | 2024-01-08 | Added Finestra Core TXN.* field classifications. Clarified IG-002 scope to include framework-level errors. |
| 2.2 | 2026-03-17 | Added one-way cryptographic derivation exception to IG-003 for spec-mandated irreversible hashing of INTERNAL/RESTRICTED fields. |
