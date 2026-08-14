# BankGate API — Auto Loan Banking Platform (MuleSoft API-Led Connectivity)

A MuleSoft-based banking API platform built to demonstrate API-led connectivity, OAuth 2.0 security, API governance and automated testing across a four-tier microservices architecture.

---

## Project Overview

This project presents an end-to-end **Banking API-Led Connectivity** workflow built using **MuleSoft Anypoint Platform, RAML, DataWeave and MUnit**.

The platform converts a monolithic banking-style API into a structured, tiered architecture that separates public-facing concerns from orchestration and system-of-record access. It helps demonstrate customer onboarding, account management, KYC verification, transaction processing, dashboard aggregation, secured access, API governance and automated regression testing.

The project is positioned as an **integration/API engineering case study** focused on architecture design, security implementation, error-handling standardisation, DataWeave transformation and MUnit-based test coverage.

---

## Business Problem

Banking platforms need a way to expose customer, account, KYC and transaction operations to client applications without tying every consumer directly to the core database or a single monolithic service.

An integration or platform engineering team needs to answer questions such as:

* Can public-facing endpoints evolve independently of core system changes?
* Is access to customer and financial data properly authenticated?
* Are errors returned in a consistent, predictable format across every endpoint?
* Is the API discoverable and governed centrally, with usage limits enforced?
* Is the integration logic actually verified, not just assumed to work?

This project solves the problem by building a three-tier API-led connectivity platform (Experience / Process / System) plus a standalone OAuth 2.0 Auth service, with governance and automated testing built in from the start.

---

## Tools Used

| Tool | Purpose |
|---|---|
| Anypoint Studio | Mule application development (4.12.1 EE) |
| DataWeave (DW 2.0) | Payload transformation and business logic |
| RAML | API contract definition |
| Anypoint Exchange / API Manager | API publishing, governance and rate-limiting policy |
| PostgreSQL | System-of-record persistence for customer, account, KYC and transaction data |
| MUnit | Automated flow-level regression testing |
| Postman | Manual endpoint verification |
| Git / GitHub | Version control and portfolio hosting |

---

## Application Modules

| Tier | Application | Port | Focus |
|---|---|---|---|
| Experience | `experience-api` | 8080 | Public-facing tier; all 18 endpoints protected with OAuth 2.0 token validation |
| Process | `process-api` | 8082 | Orchestration tier; pure pass-through proxies to the System tier |
| System | `system-api` | 8081 | Only tier with a direct database connection; owns all business data access |
| Auth | `auth-service-api` | 8084 | Standalone OAuth 2.0 client-credentials token service (self-signed HMAC-SHA256 tokens) |

```
Client → Experience API (8080) → Process API (8082) → System API (8081) → PostgreSQL
                ↑
         Auth Service (8084)
```

---

## Key Metrics Tracked

| Area | Metrics |
|---|---|
| Testing Coverage | 44/44 MUnit tests passing across customer, account, KYC, transaction and dashboard flows |
| Security | 18/18 Experience endpoints OAuth-protected and verified (valid/invalid credentials, missing/valid tokens) |
| Governance | RAML published to Exchange as BankGate API v1.0.0; rate-limiting policy verified with a real HTTP 429 on the 6th rapid request |
| Reliability | Zero unhandled errors across tested flows; standardised BANK-XXXX error envelope on every failure path |
| Deployment | All 4 apps packaged as standalone JARs (`mvn clean package`, BUILD SUCCESS) and verified deploying simultaneously outside the IDE |

---

## Data Model Summary

PostgreSQL schema `banking`, consisting of:

* `customers`
* `accounts`
* `kyc_verifications`
* `transactions`

Foreign keys and CHECK constraints (`account_type` restricted to `SAVINGS`/`CURRENT`, `account_status` to `ACTIVE`/`INACTIVE`/`CLOSED`, `verification_status` to `PENDING`/`VERIFIED`/`FAILED`) are enforced at the database level and re-validated at the API level, so a request can never bypass business rules even if it reaches the System tier directly.

Only `system-api` holds a database connection — the Experience and Process tiers never touch PostgreSQL directly, keeping the data-access boundary at a single, auditable layer.

---

## Error Handling

Every flow routes failures through a shared `global-error-response-subflow`, returning a consistent envelope:

```json
{
  "status": "FAILED",
  "errorCode": "BANK-XXXX",
  "message": "...",
  "timestamp": "...",
  "correlationId": "..."
}
```

| Code | Meaning |
|---|---|
| BANK-4001 | 400 — validation error |
| BANK-4002 | 400 — insufficient balance |
| BANK-4011 | 401 — unauthorized |
| BANK-4041 / 4042 / 4043 | 404 — customer / account / KYC not found |
| BANK-4091 | 409 — conflict / duplicate |
| BANK-4092 | 409 — conflict / foreign-key dependency |
| BANK-5000 | 500 — internal error |
| BANK-5031 / 5032 | 503 — Process / Experience tier downstream unreachable |

---

## Key Insights

* Splitting the platform into four independently deployable apps let the OAuth layer, orchestration logic and database access evolve without touching each other.
* Centralising error handling in one subflow made the BANK-XXXX error codes trivial to keep consistent across 5 separate flow files.
* Rate-limiting policy testing surfaced the exact request threshold (6th rapid request) that triggers a 429 — proof the policy is actually enforced, not just configured.
* MUnit's `flowSucceeded` boolean-variable pattern proved far more reliable for happy-path assertions than asserting directly on `payload.status`, sidestepping a recurring payload-type coercion issue.
* A Windows-specific Java Service Wrapper timeout in the standalone runtime was investigated and ruled out on several fronts (ports, duplicate deployments, antivirus, leftover processes) without a definitive root cause — documented honestly rather than left unexplained.

---

## Validation Approach

MUnit was used as the primary automated validation layer, with Postman used for manual, real-network confirmation of what MUnit couldn't cover end-to-end.

The validation process included:

* Writing MUnit suites per domain (customer, account, KYC, transaction, dashboard)
* Mocking downstream calls with `munit-tools:mock-when` and verifying payloads with `with-attributes`
* Asserting both happy-path outcomes and specific raised error types per flow
* Cross-checking flow behaviour against live Postman calls to the embedded Studio runtime
* Verifying the OAuth layer against both valid and deliberately invalid credentials/tokens
* Confirming the rate-limiting policy with a real burst of requests, not just a configuration review

Validation was performed across all four applications to ensure flows were not only reachable but behaviourally correct under both success and failure conditions.

---

## Project Outputs

| Output | Description |
|---|---|
| Four Mule Applications | Experience, Process, System and Auth tiers, each independently deployable |
| RAML Contract | Published to Anypoint Exchange as BankGate API v1.0.0 |
| MUnit Test Suites | 44 automated tests across 5 domain suites |
| Postman Collection Evidence | Manual verification of security, error handling and CRUD behaviour |
| Documentation | Architecture overview, error code reference, setup instructions (this README) |

---

## Assumptions and Limitations

* The dataset is treated as a local development/portfolio project dataset, not production banking data.
* Credentials for local development are stored only in gitignored `config.properties` files; committed `.example` files hold placeholder values only.
* Standalone (non-IDE) sustained runtime operation was not fully achieved on Windows due to an unresolved Java Service Wrapper issue; this was verified via the embedded Studio runtime instead and documented as such rather than overstated.
* PD/LGD-style business rules (e.g. KYC → Account eligibility logic) are deferred pending further business input, not yet implemented.
* Retry/resilience logic and an audit trail table are planned but not yet implemented.

---

## Skills Demonstrated

* MuleSoft API-led connectivity architecture
* DataWeave transformation and business logic
* OAuth 2.0 security implementation
* RAML API contract design
* API governance via Anypoint Exchange and API Manager
* MUnit automated testing
* PostgreSQL schema design with enforced constraints
* Standardised error-handling design
* Postman-based manual API verification
* Git/GitHub version control and documentation

---

## Project Positioning

This project demonstrates an end-to-end API integration engineering workflow for a banking-style platform. It combines tiered architecture design, OAuth 2.0 security, DataWeave transformation logic, API governance and MUnit-based automated testing to present the project as a professional integration engineering case study.

---

## Author

**Anirva Manchikatla**
B.Tech CSE | Integration Engineering