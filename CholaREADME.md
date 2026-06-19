# CholaMS

Enterprise-grade Chola Management System with a .NET 8 Web API backend and Angular 18 frontend. Provides a secure CKYC (Central KYC) verification workflow using JWT-based session handling.

---

## Table of Contents

- [Project Structure](#project-structure)
- [Tech Stack](#tech-stack)
- [Architecture Overview](#architecture-overview)
- [Authentication & Authorization](#authentication--authorization)
- [API Reference](#api-reference)
- [Application Flow](#application-flow)
- [Use Cases](#use-cases)
- [Frontend (Angular) Details](#frontend-angular-details)
- [KYC SDK (Chola.CKAC.SDK)](#kyc-sdk-cholackac-sdk)
- [Middleware Pipeline](#middleware-pipeline)
- [Configuration](#configuration)
- [Oracle 11g Compatibility](#oracle-11g-compatibility)
- [KYC Status Enum](#kyc-status-enum)
- [E2E API Testing](#e2e-api-testing)
- [Getting Started](#getting-started)
- [Health Check](#health-check)

---

## Project Structure

```
CholaMS/
├── api/                        # Backend .NET 8 projects
│   ├── CholaMS.API/            # Presentation layer (Controllers, Middleware)
│   ├── CholaMS.Application/    # Business logic (DTOs, Helpers, Mappings)
│   ├── CholaMS.Domain/         # Domain layer (Interfaces, Service Repositories)
│   ├── CholaMS.Data/           # Data access (EF Core, Oracle DB, Oracle11gCommandInterceptor)
│   ├── CholaMS.Shared/         # Cross-cutting concerns (Extensions)
│   └── CholaMS.KYC/            # Chola CKAC SDK integration (KYC service)
├── app/                        # Frontend
│   └── CholaMSUI/              # Angular 18 SPA
├── tests/                      # Test projects
│   ├── CholaMS.Test/           # Unit tests (xUnit, Moq)
│   └── Chola.CKAC.SDKValidation/
├── build/                      # CI/CD & containerization
│   ├── Dockerfile              # Multi-stage Docker build
│   ├── Dockerfile.buildagent   # Jenkins build agent image
│   └── Jenkinsfile             # CI/CD pipeline
├── scripts/                    # Automation scripts
│   ├── install.bat / .ps1      # Install dependencies
│   ├── start.bat / .ps1        # Run API + UI locally
│   ├── publish.bat / .ps1      # Build production artifacts
│   ├── generate-url.bat / .ps1 # Generate a landing page URL for browser testing
│   ├── test-apis.ps1           # End-to-end API test suite
│   └── test-apis.bat           # Batch launcher for test suite
├── db/                         # Database scripts (DDL)
├── docs/                       # Documentation
├── Packages/                   # Local NuGet packages
├── CholaMS.sln                 # Solution file
└── NuGet.config                # NuGet sources
```

---

## Tech Stack

| Layer    | Technology                                        |
| -------- | ------------------------------------------------- |
| API      | .NET 8, ASP.NET Core, EF Core 9                   |
| Database | Oracle 11g (via Oracle.EntityFrameworkCore)       |
| Frontend | Angular 18.2, TypeScript, Angular Material        |
| Auth     | JWT Bearer (HMAC-SHA256, symmetric key)           |
| KYC SDK  | Chola.CKAC.SDK (custom SDK)                       |
| Testing  | xUnit + Moq (backend), Jasmine + Karma (frontend) |
| CI/CD    | Jenkins, Docker, SonarQube                        |
| Logging  | Serilog (file + structured)                       |

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                       Client Application                            │
│  (Third-party system that wants to perform CKYC for its customer)   │
└─────────────┬───────────────────────────────────────────────────────┘
              │ 1. POST /api/v1/ckyc/initiate  (API-Key auth)
              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      CholaMS Backend API                            │
│  ┌─────────────┐   ┌──────────────┐   ┌───────────────────────┐    │
│  │   CkycController   │→│  CkycRepository  │→│  Oracle DB + KYC SDK  │    │
│  └─────────────┘   └──────────────┘   └───────────────────────┘    │
│         │                                                           │
│         │ Returns JWT token embedded in redirect URL                │
│         ▼                                                           │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │   JWT Token (claims: requestId, transactionId, 10 min TTL)  │    │
│  └─────────────────────────────────────────────────────────────┘    │
└─────────────┬───────────────────────────────────────────────────────┘
              │ 2. Redirect user's browser to:
              │    https://ckyc-app.com/start?token=<JWT>
              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     CholaMS Angular Frontend                        │
│  StartComponent → AuthService → KycComponent → KycOtpComponent     │
│         │                                                           │
│         │ All API calls include: Authorization: Bearer <JWT>        │
│         ▼                                                           │
│  GET  /api/v1/ckyc/validate       (validates session)              │
│  POST /api/v1/ckyc/get-otp        (sends OTP to customer)         │
│  POST /api/v1/ckyc/verify-otp     (verifies OTP)                  │
│  POST /api/v1/ckyc/redirect       (gets redirect URL with status) │
└─────────────┬───────────────────────────────────────────────────────┘
              │ 3. Redirect back to client with one-time code
              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                       Client Application                            │
│  POST /api/v1/ckyc/verify  (API-Key auth, exchanges code for data) │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Authentication & Authorization

The system uses **two independent authentication mechanisms** depending on the caller:

### 1. API-Key Authentication (Server-to-Server)

Used by **client applications** to initiate and verify CKYC requests.

| Aspect            | Detail                                                  |
| ----------------- | ------------------------------------------------------- |
| **Header**        | `X-API-KEY`                                             |
| **Configured in** | `appsettings.json` → `CkycSettings:ApiKey`              |
| **Endpoints**     | `POST /initiate` and `POST /verify`                     |
| **Validation**    | Server compares header value against configured key     |
| **Client ID**     | Must be in `CkycSettings:AllowedClientIds` whitelist    |
| **Redirect URL**  | Must match `CkycSettings:AllowedRedirectUrls` whitelist |

### 2. JWT Bearer Authentication (Frontend Session)

Used by the **Angular frontend** for all user-facing KYC operations.

| Aspect              | Detail                                                               |
| ------------------- | -------------------------------------------------------------------- |
| **Algorithm**       | HMAC-SHA256 (symmetric key)                                          |
| **Token Location**  | URL query parameter on initial redirect, then in-memory only         |
| **Header Format**   | `Authorization: Bearer <token>`                                      |
| **Token Lifetime**  | Configurable via `CkycSettings:TokenExpiryMinutes` (default: 10 min) |
| **Clock Skew**      | `TimeSpan.Zero` (no tolerance)                                       |
| **Issuer/Audience** | Validated against `Jwt:Issuer` and `Jwt:Audience` in config          |

#### JWT Token Claims

| Claim           | Description                              |
| --------------- | ---------------------------------------- |
| `requestId`     | Unique GUID identifying the CKYC request |
| `transactionId` | Client-provided transaction identifier   |

#### How JWT Works in This System

```
1. Client calls POST /api/v1/ckyc/initiate with X-API-KEY
       ↓
2. Server generates JWT with claims {requestId, transactionId}
       ↓
3. Server returns redirectUrl = https://ckyc-app.com/start?token=<JWT>
       ↓
4. Client redirects user's browser to that URL
       ↓
5. Angular StartComponent reads token from URL, stores in memory (AuthService)
       ↓
6. HTTP Interceptor attaches "Authorization: Bearer <token>" to every API call
       ↓
7. ASP.NET Core JWT middleware validates token on [Authorize] endpoints
       ↓
8. Controller extracts requestId from JWT claims: User.FindFirst("requestId")
       ↓
   *** requestId is NEVER accepted from the frontend request body ***
   *** It is ALWAYS extracted from the validated JWT token ***
```

#### Security Principles

- **Token is stored in-memory only** — never in localStorage or cookies
- **requestId is never sent by the frontend** — always extracted from JWT claims on the server
- **ClockSkew is zero** — token expires exactly at the configured time
- **401 responses** automatically clear the token and redirect to `/denied`

---

## API Reference

Base URL: `http://<server>/api/v1/ckyc`

### API Endpoints Summary

| #   | Method | Endpoint      | Auth       | Purpose                                             |
| --- | ------ | ------------- | ---------- | --------------------------------------------------- |
| 1   | POST   | `/initiate`   | X-API-KEY  | Client initiates a CKYC session                     |
| 2   | GET    | `/validate`   | JWT Bearer | Frontend validates the session token                |
| 3   | POST   | `/get-otp`    | JWT Bearer | Send OTP to customer's mobile                       |
| 4   | POST   | `/verify-otp` | JWT Bearer | Verify the OTP entered by customer                  |
| 5   | POST   | `/redirect`   | JWT Bearer | Get redirect URL with status/code for UI navigation |
| 6   | POST   | `/verify`     | X-API-KEY  | Client exchanges one-time code for KYC data         |

### Standard Response Envelope

All endpoints return responses in this format:

```json
{
  "status": "success | failure",
  "statusCode": "200 | 400",
  "message": "Operation completed successfully.",
  "result": {},
  "error": {
    "code": "ERROR_CODE",
    "message": "Human-readable error description"
  }
}
```

---

### 1. POST `/api/v1/ckyc/initiate`

**Purpose:** Client application initiates a new CKYC request. Returns a JWT-embedded redirect URL.

**Auth:** `X-API-KEY` header

**Request Headers:**

| Header       | Required | Description            |
| ------------ | -------- | ---------------------- |
| X-API-KEY    | Yes      | API key for the client |
| Content-Type | Yes      | `application/json`     |

**Request Body:**

```json
{
  "clientId": "CHOLA_CLIENT_01",
  "transactionId": "TXN-2026-001",
  "redirectUrl": "https://chola-prutech.com/callback",
  "payload": {
    "firstName": "Rahul",
    "lastName": "Sharma",
    "mobile": "9876543210",
    "email": "rahul@example.com",
    "dob": "1990-05-15"
  }
}
```

| Field             | Type   | Required | Description                                        |
| ----------------- | ------ | -------- | -------------------------------------------------- |
| clientId          | string | Yes      | Registered client identifier                       |
| transactionId     | string | Yes      | Unique transaction ID from the client              |
| redirectUrl       | string | Yes      | URL to redirect after KYC completion (whitelisted) |
| payload           | object | No       | Pre-fill data for the KYC form                     |
| payload.firstName | string | No       | Customer's first name                              |
| payload.lastName  | string | No       | Customer's last name                               |
| payload.mobile    | string | No       | Customer's mobile number (pre-fills form)          |
| payload.email     | string | No       | Customer's email address                           |
| payload.dob       | string | No       | Customer's date of birth (pre-fills form)          |

**Success Response (200):**

```json
{
  "status": "success",
  "statusCode": "200",
  "message": "Operation completed successfully.",
  "result": {
    "requestId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "redirectUrl": "https://ckyc-app.com/start?token=eyJhbGciOiJIUzI1NiIs..."
  },
  "error": null
}
```

**Error Response (400):**

```json
{
  "status": "failure",
  "statusCode": "400",
  "message": "Invalid API key.",
  "result": null,
  "error": {
    "code": "INVALID_API_KEY",
    "message": "The provided API key is invalid or missing."
  }
}
```

**Error Responses:**

| Error Code           | HTTP Status | Reason                                   |
| -------------------- | ----------- | ---------------------------------------- |
| INVALID_API_KEY      | 400         | Missing or incorrect X-API-KEY header    |
| INVALID_CLIENT       | 400         | clientId not in the allowed whitelist    |
| INVALID_REDIRECT_URL | 400         | redirectUrl not in the allowed whitelist |

---

### 2. GET `/api/v1/ckyc/validate`

**Purpose:** Frontend validates the JWT session and retrieves session data.

**Auth:** `Authorization: Bearer <JWT>`

**Request Headers:**

| Header        | Required | Description          |
| ------------- | -------- | -------------------- |
| Authorization | Yes      | `Bearer <jwt-token>` |

**Request Body:** None (GET request)

**Success Response (200):**

```json
{
  "status": "success",
  "statusCode": "200",
  "message": "Operation completed successfully.",
  "result": {
    "requestId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "transactionId": "TXN-2026-001",
    "redirectUrl": "https://client-app.com/callback",
    "status": "INITIATED",
    "payload": {
      "firstName": "Rahul",
      "lastName": "Sharma",
      "mobile": "9876543210",
      "email": "rahul@example.com",
      "dob": "1990-05-15"
    }
  },
  "error": null
}
```

**Error Response (400):**

```json
{
  "status": "failure",
  "statusCode": "400",
  "message": "Request has expired.",
  "result": {
    "redirectUrl": "https://client-app.com/callback"
  },
  "error": {
    "code": "EXPIRED",
    "message": "Request has expired."
  }
}
```

> **Note:** On `EXPIRED` and `ALREADY_PROCESSED` errors, the `result.redirectUrl` is still returned so the frontend can redirect the user back to the client application.

**Error Responses:**

| Error Code        | HTTP Status | Reason                                           |
| ----------------- | ----------- | ------------------------------------------------ |
| INVALID_TOKEN     | 400         | Missing requestId in JWT claims                  |
| NOT_FOUND         | 400         | Request not found in database                    |
| EXPIRED           | 400         | Request has expired (redirectUrl in result)      |
| ALREADY_PROCESSED | 400         | Request already completed/rejected (redirectUrl) |
| _(JWT expired)_   | 401         | Token has expired (auto-rejected by middleware)  |

---

### 3. POST `/api/v1/ckyc/get-otp`

**Purpose:** Triggers OTP to the customer's registered mobile number via the CKYC SDK.

**Auth:** `Authorization: Bearer <JWT>`

**Request Headers:**

| Header        | Required | Description          |
| ------------- | -------- | -------------------- |
| Authorization | Yes      | `Bearer <jwt-token>` |
| Content-Type  | Yes      | `application/json`   |

**Request Body:**

```json
{
  "idNo": "ABCDE1234F",
  "idType": "C",
  "mobileNo": "9876543210"
}
```

| Field    | Type   | Required | Max Length | Description                                        |
| -------- | ------ | -------- | ---------- | -------------------------------------------------- |
| idNo     | string | Yes      | —          | Identity document number (e.g. PAN)                |
| idType   | string | Yes      | —          | ID type code (e.g. `C` for CKYC, `E` for Voter ID) |
| mobileNo | string | Yes      | 10         | Customer's 10-digit mobile number                  |

> **Note:** `dOB` was removed from the get-otp request. DOB is no longer required by the HyperVerge CKYC API.

**Success Response (200):**

```json
{
  "status": "success",
  "statusCode": "200",
  "message": "Operation completed successfully.",
  "result": {
    "status": "Success",
    "result": {
      "message": "OTP sent successfully",
      "requestId": "hvreq-abc123"
    },
    "error": null
  },
  "error": null
}
```

> The `result.result.requestId` from the SDK is stored and used in the subsequent `verify-otp` call.

**Error Response (400):**

```json
{
  "status": "failure",
  "statusCode": "400",
  "message": "Request has expired.",
  "result": null,
  "error": {
    "code": "EXPIRED",
    "message": "The CKYC request has expired. Please initiate a new request."
  }
}
```

**Error Responses:**

| Error Code      | HTTP Status | Reason                                 |
| --------------- | ----------- | -------------------------------------- |
| NOT_FOUND       | 400         | Request not found for requestId in JWT |
| EXPIRED         | 400         | Request has expired                    |
| INVALID_PAN     | 400         | PAN number is invalid                  |
| MOBILE_MISMATCH | 400         | Mobile number does not match records   |
| CKYC_ERROR      | 400         | CKYC SDK returned an error             |

---

### 4. POST `/api/v1/ckyc/verify-otp`

**Purpose:** Verifies the OTP entered by the customer.

**Auth:** `Authorization: Bearer <JWT>`

**Request Headers:**

| Header        | Required | Description          |
| ------------- | -------- | -------------------- |
| Authorization | Yes      | `Bearer <jwt-token>` |
| Content-Type  | Yes      | `application/json`   |

**Request Body:**

```json
{
  "otp": "123456",
  "requestId": "hvreq-abc123",
  "retry": "no"
}
```

| Field     | Type   | Required | Max Length | Description                              |
| --------- | ------ | -------- | ---------- | ---------------------------------------- |
| otp       | string | Yes      | —          | 6-digit OTP received by customer         |
| requestId | string | Yes      | —          | SDK request ID from the get-otp response |
| retry     | string | No       | —          | `"yes"` to resend OTP, default `"no"`    |

> **Note:** `mobileNo` was replaced with `requestId` to match the HyperVerge CKYC API. The SDK's `requestId` (returned from get-otp) is used to correlate the OTP validation.

**Success Response (200):**

```json
{
  "status": "success",
  "statusCode": "200",
  "message": "Operation completed successfully.",
  "result": {
    "status": "success",
    "statusCode": "200",
    "result": {
      "constituitonType": "Individual",
      "accountType": "Normal",
      "ckycNo": "50012345678",
      "preFix": "Mr",
      "firstName": "Rahul",
      "lastName": "Sharma",
      "fullName": "Rahul Sharma",
      "fatherOrSpouse": "Ramesh Sharma",
      "gender": "M",
      "dob": "15-05-1990",
      "city": "Mumbai",
      "state": "Maharashtra",
      "country": "India",
      "mobileNumber": "9876543210"
    },
    "error": null
  },
  "error": null
}
```

**Error Response (400):**

```json
{
  "status": "failure",
  "statusCode": "400",
  "message": "OTP verification failed.",
  "result": null,
  "error": {
    "code": "CKYC_ERROR",
    "message": "Invalid OTP. Please try again."
  }
}
```

**Error Responses:**

| Error Code | HTTP Status | Reason                          |
| ---------- | ----------- | ------------------------------- |
| NOT_FOUND  | 400         | Request not found for requestId |
| EXPIRED    | 400         | Request has expired             |
| CKYC_ERROR | 400         | OTP verification failed         |

---

### 5. POST `/api/v1/ckyc/redirect`

**Purpose:** Called by the Angular frontend after OTP verification. For successful verification, generates a one-time code and returns the client's `redirectUrl` with status and code appended. For failed verification, returns the `redirectUrl` with error details.

**Auth:** `Authorization: Bearer <JWT>`

**Request Headers:**

| Header        | Required | Description          |
| ------------- | -------- | -------------------- |
| Authorization | Yes      | `Bearer <jwt-token>` |
| Content-Type  | Yes      | `application/json`   |

**Request Body (Success case):**

```json
{
  "kycStatus": "COMPLETED"
}
```

**Request Body (Failure case):**

```json
{
  "kycStatus": "REJECTED",
  "errorCode": "MAX_ATTEMPTS",
  "errorMessage": "Maximum OTP attempts exceeded."
}
```

| Field        | Type   | Required | Description                                              |
| ------------ | ------ | -------- | -------------------------------------------------------- |
| kycStatus    | string | Yes      | KYC status (`COMPLETED`, `VERIFIED`, or `REJECTED`)      |
| errorCode    | string | No       | Error code (used when kycStatus is not COMPLETED)        |
| errorMessage | string | No       | Error description (used when kycStatus is not COMPLETED) |

**Success Response — COMPLETED (200):**

```json
{
  "status": "success",
  "statusCode": "200",
  "message": "Operation completed successfully.",
  "result": {
    "redirectUrl": "https://client-app.com/callback?code=xK9mN2pQ7rT4vW1yB5dF8gH3jL6&status=completed&transactionId=TXN-2026-001"
  },
  "error": null
}
```

**Success Response — REJECTED (200):**

```json
{
  "status": "success",
  "statusCode": "200",
  "message": "Operation completed successfully.",
  "result": {
    "redirectUrl": "https://client-app.com/callback?status=failed&error=MAX_ATTEMPTS&message=Maximum%20OTP%20attempts%20exceeded.&transactionId=TXN-2026-001"
  },
  "error": null
}
```

**Error Responses:**

| Error Code      | HTTP Status | Reason                                 |
| --------------- | ----------- | -------------------------------------- |
| NOT_FOUND       | 400         | Request not found for requestId        |
| NO_REDIRECT_URL | 400         | No redirect URL configured for request |

---

### 6. POST `/api/v1/ckyc/verify`

**Purpose:** Client application exchanges the one-time code for KYC verification results.

**Auth:** `X-API-KEY` header

**Request Headers:**

| Header       | Required | Description            |
| ------------ | -------- | ---------------------- |
| X-API-KEY    | Yes      | API key for the client |
| Content-Type | Yes      | `application/json`     |

**Request Body:**

```json
{
  "clientId": "CHOLA_CLIENT_01",
  "transactionId": "TXN-2026-001",
  "code": "xK9mN2pQ7rT4vW1yB5dF8gH3jL6"
}
```

| Field         | Type   | Required | Description                              |
| ------------- | ------ | -------- | ---------------------------------------- |
| clientId      | string | Yes      | Must match the original initiate request |
| transactionId | string | Yes      | Must match the original initiate request |
| code          | string | Yes      | One-time code from the redirect step     |

**Success Response (200):**

```json
{
  "status": "success",
  "statusCode": "200",
  "message": "Operation completed successfully.",
  "result": {
    "requestId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "transactionId": "TXN-2026-001",
    "kycStatus": "VERIFIED",
    "kycData": {
      "pan": "XXXXXX234F",
      "aadhaarMasked": "XXXX-XXXX-5678"
    }
  },
  "error": null
}
```

> PAN is returned in masked format (only last 4 characters visible).

**Error Response (400) — Code Already Used:**

```json
{
  "status": "failure",
  "statusCode": "400",
  "message": "Code has already been used.",
  "result": null,
  "error": {
    "code": "CODE_USED",
    "message": "This one-time code has already been redeemed."
  }
}
```

**Error Response (400) — Code Expired:**

```json
{
  "status": "failure",
  "statusCode": "400",
  "message": "Code has expired.",
  "result": null,
  "error": {
    "code": "CODE_EXPIRED",
    "message": "The one-time code has expired. Codes are valid for 5 minutes."
  }
}
```

**Error Responses:**

| Error Code      | HTTP Status | Reason                                     |
| --------------- | ----------- | ------------------------------------------ |
| INVALID_API_KEY | 400         | Missing or incorrect X-API-KEY             |
| INVALID_CODE    | 400         | Code does not exist                        |
| CODE_USED       | 400         | Code has already been used (one-time only) |
| CODE_EXPIRED    | 400         | Code has expired (default: 5 minutes)      |
| MISMATCH        | 400         | transactionId or clientId does not match   |
| NOT_FOUND       | 400         | Associated CKYC request not found          |

---

### 7. GET `/api/v1/HealthCheck/db`

**Purpose:** Verify database connectivity.

**Auth:** None

**Success Response (200):**

```json
{
  "status": "Healthy",
  "database": "Connected"
}
```

---

## Application Flow

### End-to-End KYC Verification Flow

```
Step 1: CLIENT APPLICATION
    │
    │  POST /api/v1/ckyc/initiate
    │  Headers: { X-API-KEY: "your-api-key" }
    │  Body: { clientId, transactionId, redirectUrl, payload }
    │
    ▼
Step 2: SERVER generates JWT token, stores request in DB
    │  Response: { requestId, redirectUrl: "https://ckyc-app.com/start?token=<JWT>" }
    │
    ▼
Step 3: CLIENT redirects user's browser to the redirectUrl
    │  Browser opens: https://ckyc-app.com/start?token=eyJhbGciOi...
    │
    ▼
Step 4: ANGULAR APP (StartComponent)
    │  • Reads ?token= from URL
    │  • Stores JWT in AuthService (in-memory)
    │  • Calls GET /api/v1/ckyc/validate (with Bearer token)
    │  • On success → navigates to /kyc-user-verification
    │  • On failure → shows error page
    │
    ▼
Step 5: ANGULAR APP (KycComponent)
    │  • Displays KYC form (pre-filled from session payload if available)
    │  • User enters: ID Number, ID Type, DOB, Mobile Number
    │  • Calls POST /api/v1/ckyc/get-otp
    │  • On success → shows OTP input screen
    │
    ▼
Step 6: ANGULAR APP (KycOtpComponent)
    │  • User enters 6-digit OTP
    │  • Calls POST /api/v1/ckyc/verify-otp
    │  • On success → calls POST /api/v1/ckyc/redirect (status=COMPLETED)
    │  • Shows success screen with 10-second countdown
    │  • Redirects user back to client's redirectUrl with ?code=...&status=completed
    │  • On failure (max attempts) → calls POST /api/v1/ckyc/redirect (status=REJECTED)
    │  • Redirects user immediately with ?status=failed&error=...
    │
    ▼
Step 7: CLIENT APPLICATION
    │  POST /api/v1/ckyc/verify
    │  Headers: { X-API-KEY: "your-api-key" }
    │  Body: { clientId, transactionId, code }
    │
    ▼
Step 8: SERVER returns KYC verification result
    Response: { requestId, transactionId, kycStatus, kycData }
```

### Angular Route Map

| Route                    | Component           | Guard     | Description                        |
| ------------------------ | ------------------- | --------- | ---------------------------------- |
| `/start?token=<JWT>`     | StartComponent      | None      | Entry point, validates JWT session |
| `/kyc-user-verification` | KycComponent        | AuthGuard | KYC form (ID, DOB, mobile, OTP)    |
| `/denied`                | DeniedPageComponent | None      | Error/unauthorized page            |

---

## Use Cases

### Use Case 1: Happy Path — Successful KYC Verification

| Step | Actor            | Action                                     | API Called         |
| ---- | ---------------- | ------------------------------------------ | ------------------ |
| 1    | Client App       | Initiates CKYC with customer details       | `POST /initiate`   |
| 2    | Client App       | Redirects customer browser to returned URL | —                  |
| 3    | Angular App      | Validates session token                    | `GET /validate`    |
| 4    | Customer         | Fills KYC form and submits                 | `POST /get-otp`    |
| 5    | Customer         | Enters OTP received on mobile              | `POST /verify-otp` |
| 6    | Angular App      | Calls redirect API, gets URL with code     | `POST /redirect`   |
| 7    | Customer Browser | Sees 10s countdown, redirected to client   | —                  |
| 8    | Client App       | Exchanges code for KYC data                | `POST /verify`     |

### Use Case 2: Expired Session

| Step | Actor       | Action                                 | Result                               |
| ---- | ----------- | -------------------------------------- | ------------------------------------ |
| 1    | Customer    | Opens KYC link after token has expired | JWT middleware rejects with 401      |
| 2    | Angular App | Interceptor catches 401                | Clears token, redirects to `/denied` |

### Use Case 3: Invalid API Key

| Step | Actor      | Action                                 | Result                          |
| ---- | ---------- | -------------------------------------- | ------------------------------- |
| 1    | Client App | Calls `/initiate` with wrong X-API-KEY | Returns `INVALID_API_KEY` error |

### Use Case 4: Invalid OTP

| Step | Actor    | Action                | Result                                                    |
| ---- | -------- | --------------------- | --------------------------------------------------------- |
| 1    | Customer | Enters wrong OTP      | Error message, attempt counter decremented                |
| 2    | Customer | Exceeds 3 attempts    | Verify button disabled, redirect API called with REJECTED |
| 3    | UI       | Redirect API responds | Immediate redirect to client's redirectUrl with error     |

### Use Case 5: One-Time Code Already Used

| Step | Actor      | Action                         | Result                    |
| ---- | ---------- | ------------------------------ | ------------------------- |
| 1    | Client App | Calls `/verify` with same code | Returns `CODE_USED` error |

---

## Frontend (Angular) Details

### Key Services

| Service           | Responsibility                                                                                                                               |
| ----------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| `AuthService`     | Stores JWT in memory, provides `getToken()`, `clearToken()`, `isAuthenticated()`, `getAuthHeaders()`, `setSessionData()`, `getSessionData()` |
| `ApiService`      | HTTP client with `Ckyc.validate()`, `Ckyc.getOtp()`, `Ckyc.verifyOtp()`, `Ckyc.redirect()` methods                                           |
| `AuthGuard`       | Route guard that checks `authService.isAuthenticated()`; redirects to `/denied` if false                                                     |
| `authInterceptor` | HTTP interceptor that attaches Bearer token and handles 401 errors                                                                           |
| `MessageService`  | Toast notification service for success/error messages                                                                                        |

### Frontend Authentication Flow

```
1. User lands on /start?token=<JWT>
      │
      ▼
2. StartComponent:
   ├── Reads token from query params
   ├── Calls authService.setToken(token)
   ├── Calls apiService.Ckyc.validate()
   ├── On success: authService.setSessionData(data), navigate to /kyc-user-verification
   └── On error: authService.clearToken(), show error
      │
      ▼
3. authInterceptor (runs on every HTTP request):
   ├── Gets token from authService.getToken()
   ├── Clones request with Authorization: Bearer <token>
   ├── On 401 response: authService.clearToken(), navigate to /denied
   └── On other errors: shows toast via messageService
      │
      ▼
4. AuthGuard (protects /kyc-user-verification route):
   ├── Checks authService.isAuthenticated()
   ├── If true: allows navigation
   └── If false: redirects to /denied
```

### Frontend UX Features

#### Skeleton Loading (Start Page)

When the user opens the landing page URL, a **blurred skeleton preview** of the KYC form is displayed while the JWT token is being validated. This gives the user a visual cue that the form is loading.

- A shimmer animation runs across placeholder elements (header, image, inputs, button)
- The skeleton card is blurred (`filter: blur(4px)`) and semi-transparent
- A spinner with **"Your CKYC verification is about to begin..."** text overlays the skeleton
- On successful validation, the user navigates to the real KYC form

#### Error Countdown Redirect

When the session validation fails (expired token, already processed, etc.):

- An error message is displayed with a **10-second countdown** timer
- An animated progress bar visually decreases
- If `redirectUrl` is available from the API response → redirects to the client's application
- If no `redirectUrl` (e.g., no token provided) → attempts to close the window
- The denied page also shows a 10-second countdown before redirecting

---

## KYC SDK (Chola.CKAC.SDK)

The SDK integrates with the **HyperVerge CKYC Docker Solution** API for CKYC search and OTP validation.

### SDK Architecture

| File              | Purpose                                                                     |
| ----------------- | --------------------------------------------------------------------------- |
| `KycSDK.cs`       | Entry point — `Initialize(env, baseUrl, appId, appKey)` + `CreateService()` |
| `KycService.cs`   | HTTP client — `StartSearchAsync()` + `ValidateOtpAndDownloadAsync()`        |
| `KycConfig.cs`    | Configuration holder (environment, baseUrl, appId, appKey, useDummy)        |
| `KycConstants.cs` | All hardcoded values centralized (endpoints, headers, timeouts, etc.)       |

### Key Design Decisions

- **Config-driven base URL** — `BaseUrl` comes from `appsettings.json` → `KycSettings:BaseUrl`, not hardcoded in the SDK
- **Centralized constants** — All magic strings are in `KycConstants.cs` (endpoints, header names, timeouts, defaults)
- **SocketsHttpHandler** — Uses a single static `HttpClient` with `PooledConnectionLifetime` for production-grade connection pooling
- **Dummy responses** — In `DEV` environment, `UseDummyResponse` is auto-set to `true`, sending the `getdummyresponse: yes` header
- **Unique transaction IDs** — Each SDK call gets a unique transaction ID (`APIOrch_<guid>`)

### HyperVerge CKYC API Endpoints

| SDK Method                      | HTTP | Endpoint                         | Purpose                    |
| ------------------------------- | ---- | -------------------------------- | -------------------------- |
| `StartSearchAsync()`            | POST | `/api/v1/start`                  | Initiate CKYC search + OTP |
| `ValidateOtpAndDownloadAsync()` | POST | `/api/v1/validateOtpAndDownload` | Verify OTP + download CKYC |

---

## Middleware Pipeline

Requests pass through the following middleware in order:

| Order | Middleware                  | Purpose                                                                                                                                                    |
| ----- | --------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1     | ExceptionHandlingMiddleware | Catches unhandled exceptions, logs to DB and file, returns structured JSON error                                                                           |
| 2     | SecurityHeadersMiddleware   | Adds security headers: `X-Content-Type-Options`, `X-Frame-Options`, `X-XSS-Protection`, `Referrer-Policy`, `Permissions-Policy`, `Cache-Control`, `Pragma` |
| 3     | Serilog Request Logging     | Logs HTTP request/response details                                                                                                                         |
| 4     | CORS Policy                 | Allows configured origins                                                                                                                                  |
| 5     | Authentication              | JWT Bearer token validation                                                                                                                                |
| 6     | Authorization               | Enforces `[Authorize]` on protected endpoints                                                                                                              |
| 7     | ResponseHandlerMiddleware   | Maps business status codes to HTTP status codes, logs errors to DB                                                                                         |

---

## Configuration

### Key Configuration Settings (`appsettings.json`)

| Section             | Key                    | Description                                | Example                    |
| ------------------- | ---------------------- | ------------------------------------------ | -------------------------- |
| `Jwt`               | `Key`                  | HMAC-SHA256 signing key                    | (min 32 chars)             |
| `Jwt`               | `Issuer`               | Token issuer                               | `https://your-domain.com`  |
| `Jwt`               | `Audience`             | Token audience                             | `https://your-domain.com`  |
| `Jwt`               | `TestTokenExpiryDays`  | Dev-only token expiry for /GenerateToken   | `2`                        |
| `CkycSettings`      | `ApiKey`               | API key for client authentication          | (change in production)     |
| `CkycSettings`      | `AllowedClientIds`     | Array of allowed client IDs                | `["CHOLA_CLIENT_01"]`      |
| `CkycSettings`      | `AllowedRedirectUrls`  | Array of whitelisted redirect URLs         | `["https://your-app.com"]` |
| `CkycSettings`      | `CkycAppBaseUrl`       | Base URL of the Angular frontend           | `https://ckyc-app.com`     |
| `CkycSettings`      | `TokenExpiryMinutes`   | JWT token lifetime                         | `10`                       |
| `CkycSettings`      | `RequestExpiryMinutes` | CKYC request lifetime in DB                | `10`                       |
| `CkycSettings`      | `CodeExpiryMinutes`    | One-time code lifetime                     | `5`                        |
| `ConnectionStrings` | `DefaultConnection`    | Oracle connection string                   | —                          |
| `KycSettings`       | `Environment`          | KYC SDK environment (`DEV`, `UAT`, `PROD`) | `DEV`                      |
| `KycSettings`       | `BaseUrl`              | KYC SDK base URL (passed to SDK init)      | `http://52.66.180.185:90`  |
| `KycSettings`       | `AppId`                | KYC SDK application ID                     | —                          |
| `KycSettings`       | `AppKey`               | KYC SDK application key                    | —                          |

### Session Expiry Timers

Three independent timers control the session lifecycle:

| Setting                | Default | Purpose                                                                                            |
| ---------------------- | ------- | -------------------------------------------------------------------------------------------------- |
| `TokenExpiryMinutes`   | 10 min  | JWT token lifetime. User must open the landing page URL and complete KYC within this window.       |
| `RequestExpiryMinutes` | 10 min  | Database request record validity. The overall window for the entire initiate-to-completion flow.   |
| `CodeExpiryMinutes`    | 5 min   | One-time code lifetime. After KYC completes, the client must exchange the code within this window. |

All values are configurable in `appsettings.json` under `CkycSettings`.

---

## Oracle 11g Compatibility

The application targets **Oracle 11g** which does not support the `FETCH FIRST n ROWS ONLY` syntax used by EF Core's Oracle provider. This is handled transparently by the `Oracle11gCommandInterceptor`.

### How It Works

The interceptor hooks into EF Core's `DbCommandInterceptor` pipeline and rewrites SQL **before** it reaches the database:

| EF Core Generated SQL (12c+)                       | Interceptor Rewrites To (11g)                                                                    |
| -------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| `SELECT ... WHERE ... FETCH FIRST 1 ROWS ONLY`     | `SELECT ... WHERE ... AND ROWNUM <= 1`                                                           |
| `SELECT ... OFFSET 5 ROWS FETCH NEXT 10 ROWS ONLY` | `SELECT * FROM (SELECT a__.*, ROWNUM rnum__ FROM (...) a__ WHERE ROWNUM <= 15) WHERE rnum__ > 5` |

### Key Details

- **File:** `api/CholaMS.Data/Oracle11gCommandInterceptor.cs`
- **Registered in:** `Program.cs` via `.AddInterceptors(new Oracle11gCommandInterceptor())`
- **SQL Compatibility:** `UseOracleSQLCompatibility(OracleSQLCompatibility.DatabaseVersion19)` (lowest available)
- Handles queries with JOINs (e.g., `.Include()`) without column ambiguity
- Correctly inserts `ROWNUM` before `ORDER BY` clauses
- Uses `\sWHERE\s` regex to detect WHERE clauses regardless of whitespace (newlines, tabs)

### Upgrading to Oracle 12c+

When the database is upgraded to Oracle 12c or later:

1. Remove `.AddInterceptors(new Oracle11gCommandInterceptor())` from `Program.cs`
2. Optionally update `UseOracleSQLCompatibility` to match the target version
3. No other code changes required

---

## KYC Status Enum

The `KycStatus` enum (`api/CholaMS.Application/Enums/KycStatus.cs`) defines the lifecycle states of a CKYC request:

| Value         | Description                               |
| ------------- | ----------------------------------------- |
| `INITIATED`   | Request created, awaiting customer action |
| `IN_PROGRESS` | Customer has started KYC verification     |
| `VERIFIED`    | KYC successfully verified                 |
| `REJECTED`    | KYC verification failed                   |
| `EXPIRED`     | Request or session timed out              |
| `COMPLETED`   | Full workflow completed (code exchanged)  |

Serialized as **string values** in JSON responses using `JsonStringEnumConverter` for backward compatibility.

---

## E2E API Testing

An automated end-to-end test script validates all 7 API endpoints in sequence.

### Running Tests

**Prerequisites:** API server must be running on `http://localhost:5129`

```bash
# Option 1: Batch file (double-click or from cmd)
scripts\test-apis.bat

# Option 2: PowerShell directly
powershell -ExecutionPolicy Bypass -File scripts\test-apis.ps1
```

### What It Does

1. **Cleanup** - Deletes previous test data from Oracle (CKYC_CODES and CKYC_REQUESTS) matching the test clientId/transactionId
2. **Runs all 7 tests** sequentially, chaining values across calls:
   - Health Check → Initiate → Validate → Get OTP → Verify OTP → Redirect → Verify (Exchange Code)
3. **Displays summary table** with colour-coded results:

```
+-----+----------------------+--------+----------------------------+------+--------+---------+
| #   | Test Name            | Method | Endpoint                   | HTTP | Status | Time    |
+-----+----------------------+--------+----------------------------+------+--------+---------+
| 1   | 1. Health Check      | GET    | /api/v1/HealthCheck/db     | 200  | PASS   | 1695 ms |
| 2   | 2. Initiate CKYC     | POST   | /api/v1/ckyc/initiate      | 200  | PASS   | 1697 ms |
| 3   | 3. Validate Session  | GET    | /api/v1/ckyc/validate      | 200  | PASS   |  626 ms |
| 4   | 4. Get OTP           | POST   | /api/v1/ckyc/get-otp       | 200  | PASS   |  736 ms |
| 5   | 5. Verify OTP        | POST   | /api/v1/ckyc/verify-otp    | 200  | PASS   |  221 ms |
| 6   | 6. Redirect          | POST   | /api/v1/ckyc/redirect      | 200  | PASS   |  432 ms |
| 7   | 7. Exchange Code     | POST   | /api/v1/ckyc/verify        | 200  | PASS   |  851 ms |
+-----+----------------------+--------+----------------------------+------+--------+---------+

  Total: 7 tests | Passed: 7 | Failed: 0 | Total Time: 6207 ms

  ALL TESTS PASSED
```

### Test Configuration

Edit the variables at the top of `scripts/test-apis.ps1`:

| Variable             | Default                 | Description                      |
| -------------------- | ----------------------- | -------------------------------- |
| `$base`              | `http://localhost:5129` | API base URL                     |
| `$testClientId`      | `CHOLA_CLIENT_01`       | Client ID for test requests      |
| `$testTransactionId` | `TXN-E2E-002`           | Transaction ID for test requests |

---

## Getting Started

### Prerequisites

- [.NET 8 SDK](https://dotnet.microsoft.com/download/dotnet/8.0)
- [Node.js 18+](https://nodejs.org/) with npm
- Oracle database access (for API)

### Install

```bash
scripts\install.bat        # or .\scripts\install.ps1
```

### Run Locally

```bash
scripts\start.bat          # or .\scripts\start.ps1
```

- **API**: http://localhost:5129 (Swagger at /swagger)
- **UI**: http://localhost:4200

### Generate a Landing Page URL (Dev)

To quickly generate a testing URL with a valid JWT token:

```bash
scripts\generate-url.bat     # or: powershell -ExecutionPolicy Bypass -File scripts\generate-url.ps1
```

The script:

1. Checks the API is running on `localhost:5129` (TCP port check)
2. Calls `POST /api/v1/ckyc/initiate` with demo payload
3. Prints the `http://localhost:4200/start?token=...` URL
4. Copies the URL to clipboard

> **Prerequisite:** API and UI must be running (`scripts\start.bat`)

### Run Tests

```bash
# Backend unit tests (xUnit)
dotnet test

# Frontend unit tests (Jasmine + Karma)
cd app/CholaMSUI
npx ng test --watch=false --browsers=ChromeHeadless

# End-to-end API tests (requires API server running)
scripts\test-apis.bat
```

### Publish for Deployment

```bash
scripts\publish.bat        # or .\scripts\publish.ps1
```

Output goes to `publish-output/api/` and `publish-output/ui/`.

---

## Health Check

After deployment, verify the database connection:

```
GET http://<server-url>/api/v1/HealthCheck/db
```

Returns `200 OK` with `{"status":"Healthy","database":"Connected"}` on success.
