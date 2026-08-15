# BankGate API Reference

This document lists the public-facing endpoints exposed through the Experience API tier (`http://localhost:8080`), the request/response shape for each, and how they map through the Process and System tiers to PostgreSQL.

All endpoints below require a valid OAuth 2.0 bearer token (obtained from `auth-service-api`) except the token endpoint itself.

---

## Authentication

### `POST /oauth/token`
**Tier:** `auth-service-api` (port 8084)
**Description:** Issues a self-signed HMAC-SHA256 access token using client-credentials grant.

**Request body:**
```json
{
  "client_id": "your_client_id",
  "client_secret": "your_client_secret",
  "grant_type": "client_credentials"
}
```

**Success response — `200 OK`:**
```json
{
  "access_token": "base64-encoded-token",
  "token_type": "Bearer",
  "expires_in": 3600
}
```

**Error response — `401 Unauthorized`:** invalid credentials or missing/incorrect `grant_type`.

All subsequent requests must include:
```
Authorization: Bearer <access_token>
```

---

## Customers

### `POST /customers/onboard`
**Description:** Creates a new customer record after validating required fields, email format, minimum age (18+), and checking for duplicates on email/phone/PAN/Aadhaar.

**Request body:**
```json
{
  "first_name": "string",
  "last_name": "string",
  "email": "string",
  "phone": "string",
  "date_of_birth": "yyyy-MM-dd",
  "pan": "string",
  "aadhaar": "string",
  "address_line": "string",
  "city": "string",
  "state": "string",
  "postal_code": "string"
}
```

**Success response — `200 OK`:**
```json
{ "status": "SUCCESS", "message": "Customer inserted successfully" }
```

**Error responses:**
| Code | HTTP | Trigger |
|---|---|---|
| BANK-4001 | 400 | Missing required fields, invalid email format, invalid/malformed date of birth, or customer under 18 |
| BANK-4091 | 409 | Duplicate email, phone, PAN, or Aadhaar |

---

### `GET /customers`
**Description:** Returns all customer records.

**Success response — `200 OK`:** array of customer objects (same shape as the onboarding request, plus `customer_id`).

---

### `GET /customers/{customerId}`
**Description:** Returns a single customer by ID.

**Success response — `200 OK`:** a single customer object.

**Error responses:**
| Code | HTTP | Trigger |
|---|---|---|
| BANK-4001 | 400 | `customerId` is not a positive numeric value |
| BANK-4041 | 404 | No customer found with that ID |

---

### `PUT /customers/{customerId}`
**Description:** Updates an existing customer's contact and address details. Re-validates required fields, email format, and duplicate email/phone against other customers before applying the update.

**Request body:** same shape as onboarding, minus PAN/Aadhaar/date of birth (identity fields are not editable after onboarding).

**Success response — `200 OK`:**
```json
{ "status": "SUCCESS", "message": "Customer updated successfully" }
```

**Error responses:**
| Code | HTTP | Trigger |
|---|---|---|
| BANK-4001 | 400 | Invalid `customerId`, missing required fields, or invalid email format |
| BANK-4041 | 404 | Customer not found |
| BANK-4091 | 409 | Another customer already uses the provided email or phone |

---

### `DELETE /customers/{customerId}`
**Description:** Deletes a customer record.

**Success response — `200 OK`:**
```json
{ "status": "SUCCESS", "message": "Customer deleted successfully" }
```

**Error responses:**
| Code | HTTP | Trigger |
|---|---|---|
| BANK-4001 | 400 | Invalid `customerId` |
| BANK-4041 | 404 | Customer not found (zero rows affected) |
| BANK-4092 | 409 | Customer has dependent accounts or KYC records (foreign-key constraint) |

---

## Accounts, KYC, Transactions, Dashboard

These modules follow the same conventions as the Customers endpoints above — standard CRUD-style operations proxied Experience → Process → System, validated at both the API and database level, and returning the same `BANK-XXXX` error envelope on failure. Each is implemented as its own flow file per tier (e.g. `account-experience.xml`, `account-process.xml`, `account-system.xml`), following the identical validation → duplicate/existence check → operation → structured response pattern documented above for Customers.

For the exact request/response contract of each endpoint in these modules, refer to the RAML definition at `system-api/src/main/resources/api/api.raml` and its supporting files under `api.libraries/` and `api.traits/`, which are the source of truth for the full API contract and are also published to Anypoint Exchange as **BankGate API v1.0.0**.

---

## Standard Error Envelope

Every error response, regardless of endpoint, follows this shape:

```json
{
  "status": "FAILED",
  "errorCode": "BANK-XXXX",
  "message": "Human-readable description",
  "timestamp": "yyyy-MM-ddTHH:mm:ss",
  "correlationId": "..."
}
```

See the main [README](README.md#error-handling) for the full error code table.
