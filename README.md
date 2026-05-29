# 📋 Registration (REG) System — Complete Documentation

> **Version:** 2.1.0 | **Last Updated:** May 2026 | **Status:** Active

---

## 📑 Table of Contents

1. [Overview](#overview)
2. [System Architecture](#system-architecture)
3. [Registration Flow](#registration-flow)
4. [API Reference](#api-reference)
5. [User Registration Process](#user-registration-process)
6. [Email Verification Flow](#email-verification-flow)
7. [Role & Permission Assignment](#role--permission-assignment)
8. [Error Handling](#error-handling)
9. [Security Guidelines](#security-guidelines)
10. [Database Schema](#database-schema)
11. [Configuration](#configuration)
12. [Troubleshooting](#troubleshooting)

---

## 🔍 Overview

The **REG (Registration) System** is a centralized, secure, and scalable module responsible for onboarding new users, organizations, and third-party services into the platform. It handles:

- ✅ New user sign-up (individual & enterprise)
- ✅ Email/phone verification
- ✅ Role assignment & permission scoping
- ✅ Account activation & deactivation
- ✅ Third-party OAuth integration (Google, GitHub, Microsoft)
- ✅ Audit logging for compliance

---

## 🏗️ System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     REG SYSTEM ARCHITECTURE                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ┌──────────┐    ┌──────────────┐    ┌────────────────────┐   │
│   │  Client  │───▶│  API Gateway │───▶│  REG Microservice  │   │
│   │  (Web /  │    │  (Auth +     │    │  - Validation      │   │
│   │  Mobile) │    │   Rate Limit)│    │  - Business Logic  │   │
│   └──────────┘    └──────────────┘    └────────┬───────────┘   │
│                                                │               │
│                        ┌───────────────────────┤               │
│                        │                       │               │
│               ┌────────▼──────┐    ┌───────────▼──────────┐   │
│               │  User DB      │    │  Notification Service│   │
│               │  (PostgreSQL) │    │  - Email (SMTP)      │   │
│               └───────────────┘    │  - SMS (Twilio)      │   │
│                                    └──────────────────────┘   │
│                                                                 │
│   ┌──────────────┐    ┌──────────────┐    ┌─────────────────┐  │
│   │  Redis Cache │    │  Audit Log   │    │  OAuth Provider │  │
│   │  (Sessions)  │    │  (Elastic)   │    │  (Google/GH/MS) │  │
│   └──────────────┘    └──────────────┘    └─────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Component Responsibilities

| Component | Technology | Purpose |
|-----------|-----------|---------|
| **API Gateway** | Kong / AWS API Gateway | Auth, rate limiting, routing |
| **REG Service** | Node.js / Express | Core registration logic |
| **User DB** | PostgreSQL 15 | Persistent user data |
| **Cache** | Redis 7 | Session tokens, OTP storage |
| **Notification** | SendGrid + Twilio | Email & SMS verification |
| **Audit Log** | Elasticsearch | Compliance & traceability |
| **OAuth** | OAuth 2.0 / OIDC | Third-party login |

---

## 🔄 Registration Flow

### Complete End-to-End Registration Flowchart

```
                    ┌─────────────────────┐
                    │   USER VISITS       │
                    │   SIGN-UP PAGE      │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │  Fill Registration  │
                    │  Form               │
                    │  - Name             │
                    │  - Email            │
                    │  - Password         │
                    │  - Phone (optional) │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐        ┌──────────────────┐
                    │  Client-Side        │──FAIL──▶│  Show Inline     │
                    │  Validation         │        │  Error Messages  │
                    └──────────┬──────────┘        └──────────────────┘
                               │ PASS
                               ▼
                    ┌─────────────────────┐
                    │  POST /api/register  │
                    │  (Submit to Backend) │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐        ┌──────────────────┐
                    │  Rate Limit Check   │──FAIL──▶│  429 Too Many    │
                    │  (5 req/min/IP)     │        │  Requests        │
                    └──────────┬──────────┘        └──────────────────┘
                               │ PASS
                               ▼
                    ┌─────────────────────┐        ┌──────────────────┐
                    │  Server-Side        │──FAIL──▶│  400 Bad Request │
                    │  Validation         │        │  with details    │
                    └──────────┬──────────┘        └──────────────────┘
                               │ PASS
                               ▼
                    ┌─────────────────────┐        ┌──────────────────┐
                    │  Check Email        │──EXISTS▶│  409 Email       │
                    │  Uniqueness         │        │  Already Taken   │
                    └──────────┬──────────┘        └──────────────────┘
                               │ UNIQUE
                               ▼
                    ┌─────────────────────┐
                    │  Hash Password      │
                    │  (bcrypt, 12 rounds)│
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │  Create User Record │
                    │  (status: PENDING)  │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │  Generate OTP/Token  │
                    │  Store in Redis (TTL │
                    │  = 15 minutes)      │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │  Send Verification  │
                    │  Email via SendGrid │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │  Return 201 Created  │
                    │  + Pending Message   │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │  User Clicks Email  │
                    │  Verification Link  │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐        ┌──────────────────┐
                    │  Token Valid &      │──FAIL──▶│  Show Resend     │
                    │  Not Expired?       │        │  Email Option    │
                    └──────────┬──────────┘        └──────────────────┘
                               │ VALID
                               ▼
                    ┌─────────────────────┐
                    │  Activate Account   │
                    │  (status: ACTIVE)   │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │  Assign Default     │
                    │  Role: USER         │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │  Log Audit Event    │
                    │  REGISTRATION_OK    │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │  ✅ REGISTRATION    │
                    │     COMPLETE        │
                    │  Redirect to Login  │
                    └─────────────────────┘
```

---

## 📡 API Reference

### Base URL

```
Production:  https://api.yourdomain.com/v2/reg
Staging:     https://staging-api.yourdomain.com/v2/reg
```

---

### `POST /register` — Register a New User

**Request Headers**

```http
Content-Type: application/json
X-Client-ID: <your-client-id>
```

**Request Body**

```json
{
  "first_name": "Aarav",
  "last_name": "Shah",
  "email": "aarav.shah@example.com",
  "password": "SecurePass@123",
  "phone": "+91-9876543210",
  "account_type": "individual",
  "referral_code": "REF2024XYZ"
}
```

**Field Definitions**

| Field | Type | Required | Constraints |
|-------|------|----------|-------------|
| `first_name` | string | ✅ Yes | 2–50 chars, alpha only |
| `last_name` | string | ✅ Yes | 2–50 chars, alpha only |
| `email` | string | ✅ Yes | Valid email, unique |
| `password` | string | ✅ Yes | Min 8 chars, 1 upper, 1 number, 1 special |
| `phone` | string | ❌ No | E.164 format |
| `account_type` | enum | ✅ Yes | `individual` or `enterprise` |
| `referral_code` | string | ❌ No | 10-char alphanumeric |

**Success Response — `201 Created`**

```json
{
  "status": "success",
  "message": "Registration successful. Please verify your email.",
  "data": {
    "user_id": "usr_7f3a9bc2d1e4",
    "email": "aarav.shah@example.com",
    "status": "PENDING_VERIFICATION",
    "created_at": "2026-05-29T10:32:00Z"
  }
}
```

**Error Responses**

| HTTP Code | Error Code | Description |
|-----------|-----------|-------------|
| `400` | `VALIDATION_ERROR` | Invalid input fields |
| `409` | `EMAIL_EXISTS` | Email already registered |
| `422` | `WEAK_PASSWORD` | Password does not meet criteria |
| `429` | `RATE_LIMITED` | Too many requests from IP |
| `500` | `SERVER_ERROR` | Internal server error |

---

### `POST /verify-email` — Verify Email Address

**Request Body**

```json
{
  "user_id": "usr_7f3a9bc2d1e4",
  "token": "a4f9e8b2c1d7"
}
```

**Success Response — `200 OK`**

```json
{
  "status": "success",
  "message": "Email verified. Account is now active.",
  "data": {
    "user_id": "usr_7f3a9bc2d1e4",
    "status": "ACTIVE",
    "activated_at": "2026-05-29T10:47:00Z"
  }
}
```

---

### `POST /resend-verification` — Resend Verification Email

**Request Body**

```json
{
  "email": "aarav.shah@example.com"
}
```

> **Rate Limit:** Max 3 resends per hour per email address.

---

### `POST /register/oauth` — Register via OAuth Provider

**Request Body**

```json
{
  "provider": "google",
  "id_token": "eyJhbGciOiJSUzI1NiIsImtpZCI6...",
  "account_type": "individual"
}
```

**Supported Providers**

| Provider | Value | Notes |
|----------|-------|-------|
| Google | `google` | Uses Google OIDC |
| GitHub | `github` | GitHub OAuth App |
| Microsoft | `microsoft` | Azure AD / MSAL |

---

## 👤 User Registration Process

### Step-by-Step Guide for End Users

#### Step 1 — Navigate to Registration Page

Open your browser and visit:

```
https://app.yourdomain.com/register
```

#### Step 2 — Fill in the Registration Form

Complete all required fields:

```
┌─────────────────────────────────────────────┐
│            CREATE YOUR ACCOUNT              │
├─────────────────────────────────────────────┤
│  First Name: [ Aarav              ]         │
│  Last Name:  [ Shah               ]         │
│  Email:      [ aarav@example.com  ]         │
│  Password:   [ ••••••••••••       ]         │
│  Confirm:    [ ••••••••••••       ]         │
│  Phone:      [ +91 98765 43210    ] (opt.)  │
│                                             │
│  Account Type:  ● Individual  ○ Enterprise  │
│                                             │
│  [✓] I agree to Terms of Service           │
│  [✓] I am 18 years or older                │
│                                             │
│         [ CREATE ACCOUNT ]                  │
└─────────────────────────────────────────────┘
```

#### Step 3 — Check Your Email

After submitting, you'll receive an email from `no-reply@yourdomain.com`:

```
Subject: Verify your email address

Hi Aarav,

Thank you for registering! Please click below to verify
your email address within the next 15 minutes.

  [ VERIFY MY EMAIL ]

If you did not register, please ignore this email.
```

#### Step 4 — Activate Your Account

Click the verification link. You'll be redirected to a success page and your account will be **ACTIVE**.

---

## 📧 Email Verification Flow

```
  CLIENT                    REG SERVICE                REDIS              EMAIL
    │                            │                       │                   │
    │── POST /register ─────────▶│                       │                   │
    │                            │── Store pending ──────▶│                   │
    │                            │── Generate token ─────▶│ (TTL: 15min)      │
    │                            │                       │                   │
    │                            │──────────────────────────────────────────▶│
    │                            │            Send verification email        │
    │◀── 201 Created ────────────│                       │                   │
    │                            │                       │                   │
    │   [User clicks email link] │                       │                   │
    │                            │                       │                   │
    │── GET /verify?token=xyz ──▶│                       │                   │
    │                            │── GET token ─────────▶│                   │
    │                            │◀─ token data ─────────│                   │
    │                            │                       │                   │
    │                            │  [Validate token]     │                   │
    │                            │── DELETE token ───────▶│                   │
    │                            │  [Activate user]      │                   │
    │◀── 200 Account Active ─────│                       │                   │
    │                            │                       │                   │
```

### Token Specifications

| Property | Value |
|----------|-------|
| Format | 24-character hex string |
| Storage | Redis (key: `verify:{user_id}`) |
| TTL | 900 seconds (15 minutes) |
| Algorithm | `crypto.randomBytes(12).toString('hex')` |
| Max Resends | 3 per hour per email |

---

## 🔐 Role & Permission Assignment

### Default Role Hierarchy

```
                          ┌──────────────┐
                          │  SUPER_ADMIN │  (Platform Owner)
                          └──────┬───────┘
                                 │
                    ┌────────────┴────────────┐
                    │                         │
             ┌──────▼──────┐          ┌───────▼──────┐
             │    ADMIN    │          │   ORG_ADMIN  │
             └──────┬──────┘          └───────┬──────┘
                    │                         │
          ┌─────────┴──────────┐    ┌─────────┴──────────┐
          │                    │    │                     │
   ┌──────▼──────┐    ┌────────▼──┐ │  ┌─────────────┐   │
   │  MODERATOR  │    │  MANAGER  │ │  │  ORG_MEMBER │   │
   └─────────────┘    └───────────┘ │  └─────────────┘   │
                                    │                     │
                          ┌─────────▼──────────┐         │
                          │       USER         │◀────────┘
                          │  (Default on reg.) │
                          └────────────────────┘
```

### Permissions Matrix

| Action | USER | MANAGER | ADMIN | SUPER_ADMIN |
|--------|------|---------|-------|-------------|
| View own profile | ✅ | ✅ | ✅ | ✅ |
| Edit own profile | ✅ | ✅ | ✅ | ✅ |
| View other users | ❌ | ✅ | ✅ | ✅ |
| Edit other users | ❌ | ❌ | ✅ | ✅ |
| Delete users | ❌ | ❌ | ❌ | ✅ |
| Manage roles | ❌ | ❌ | ✅ | ✅ |
| View audit logs | ❌ | ❌ | ✅ | ✅ |
| System config | ❌ | ❌ | ❌ | ✅ |

---

## ⚠️ Error Handling

### Error Response Format

All errors follow this standard format:

```json
{
  "status": "error",
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "One or more fields failed validation.",
    "details": [
      {
        "field": "password",
        "message": "Password must be at least 8 characters long."
      },
      {
        "field": "email",
        "message": "Please enter a valid email address."
      }
    ]
  },
  "request_id": "req_8b3f1c2a4d9e"
}
```

### Common Error Codes

| Code | HTTP | Cause | Resolution |
|------|------|-------|-----------|
| `VALIDATION_ERROR` | 400 | Missing/invalid field | Check field format |
| `EMAIL_EXISTS` | 409 | Duplicate email | Use different email or login |
| `WEAK_PASSWORD` | 422 | Password too simple | Use stronger password |
| `TOKEN_EXPIRED` | 410 | Verification token TTL elapsed | Request new token |
| `TOKEN_INVALID` | 400 | Tampered or wrong token | Request new token |
| `RATE_LIMITED` | 429 | Too many requests | Wait and retry |
| `ACCOUNT_SUSPENDED` | 403 | Account flagged | Contact support |
| `SERVER_ERROR` | 500 | Unexpected error | Retry or contact support |

### Error Decision Flowchart

```
  Error Received
       │
       ▼
  HTTP 4xx or 5xx?
       │
  ┌────┴────┐
  │         │
 4xx       5xx
  │         │
  ▼         ▼
Client   Server Error
Error    → Retry with
  │        exponential
  │        backoff
  │
  ▼
Which 4xx?
  │
  ├── 400 → Fix request body
  ├── 401 → Re-authenticate
  ├── 403 → Check permissions
  ├── 404 → Check endpoint URL
  ├── 409 → Handle conflict (e.g., email taken)
  ├── 422 → Fix field constraints
  └── 429 → Implement rate-limit retry
```

---

## 🛡️ Security Guidelines

### Password Policy

```
Minimum Length:        8 characters
Maximum Length:        128 characters
Required Characters:
  - Uppercase:         ≥ 1 (A–Z)
  - Lowercase:         ≥ 1 (a–z)
  - Number:            ≥ 1 (0–9)
  - Special:           ≥ 1 (!@#$%^&*)

Hashing:               bcrypt, cost factor 12
Salting:               Automatic (bcrypt built-in)
Storage:               Hash only — plaintext NEVER stored
Breach Check:          HaveIBeenPwned API (optional, async)
```

### Rate Limiting Rules

| Endpoint | Limit | Window | Scope |
|----------|-------|--------|-------|
| `POST /register` | 5 | 1 minute | Per IP |
| `POST /verify-email` | 10 | 1 minute | Per IP |
| `POST /resend-verification` | 3 | 1 hour | Per Email |
| `POST /register/oauth` | 10 | 1 minute | Per IP |

### Security Headers (Required)

```http
Strict-Transport-Security: max-age=31536000; includeSubDomains
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
Content-Security-Policy: default-src 'self'
Referrer-Policy: no-referrer
```

### OWASP Top 10 Mitigations Applied

| Threat | Mitigation |
|--------|-----------|
| Injection | Parameterized queries (no raw SQL) |
| Broken Auth | bcrypt + JWT short-lived tokens |
| XSS | Input sanitization + CSP headers |
| Insecure Deserialization | JSON schema validation (Zod/Joi) |
| Sensitive Data Exposure | HTTPS enforced, PII encrypted at rest |
| CSRF | CSRF tokens on all form submissions |

---

## 🗃️ Database Schema

### `users` Table

```sql
CREATE TABLE users (
    id              UUID            PRIMARY KEY DEFAULT gen_random_uuid(),
    first_name      VARCHAR(50)     NOT NULL,
    last_name       VARCHAR(50)     NOT NULL,
    email           VARCHAR(255)    UNIQUE NOT NULL,
    phone           VARCHAR(20),
    password_hash   VARCHAR(255)    NOT NULL,
    account_type    VARCHAR(20)     NOT NULL DEFAULT 'individual'
                                    CHECK (account_type IN ('individual', 'enterprise')),
    status          VARCHAR(30)     NOT NULL DEFAULT 'PENDING_VERIFICATION'
                                    CHECK (status IN (
                                        'PENDING_VERIFICATION',
                                        'ACTIVE',
                                        'SUSPENDED',
                                        'DELETED'
                                    )),
    role            VARCHAR(30)     NOT NULL DEFAULT 'USER',
    referral_code   VARCHAR(10),
    oauth_provider  VARCHAR(20),
    oauth_id        VARCHAR(255),
    created_at      TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    activated_at    TIMESTAMPTZ,
    last_login_at   TIMESTAMPTZ
);

-- Indexes
CREATE INDEX idx_users_email      ON users(email);
CREATE INDEX idx_users_status     ON users(status);
CREATE INDEX idx_users_created_at ON users(created_at DESC);
```

### `audit_logs` Table

```sql
CREATE TABLE audit_logs (
    id          UUID            PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id     UUID            REFERENCES users(id) ON DELETE SET NULL,
    event       VARCHAR(100)    NOT NULL,
    ip_address  INET,
    user_agent  TEXT,
    metadata    JSONB,
    created_at  TIMESTAMPTZ     NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_audit_user_id    ON audit_logs(user_id);
CREATE INDEX idx_audit_event      ON audit_logs(event);
CREATE INDEX idx_audit_created_at ON audit_logs(created_at DESC);
```

### Entity Relationship Diagram

```
  ┌────────────────────────┐         ┌──────────────────────────┐
  │         users          │         │       audit_logs          │
  ├────────────────────────┤         ├──────────────────────────┤
  │ id (PK, UUID)          │◀────────│ user_id (FK, UUID)        │
  │ first_name             │  1:N    │ id (PK, UUID)             │
  │ last_name              │         │ event                     │
  │ email (UNIQUE)         │         │ ip_address                │
  │ phone                  │         │ user_agent                │
  │ password_hash          │         │ metadata (JSONB)          │
  │ account_type           │         │ created_at                │
  │ status                 │         └──────────────────────────┘
  │ role                   │
  │ referral_code          │         ┌──────────────────────────┐
  │ oauth_provider         │         │       user_roles          │
  │ oauth_id               │         ├──────────────────────────┤
  │ created_at             │◀────────│ user_id (FK, UUID)        │
  │ updated_at             │  1:N    │ role_name                 │
  │ activated_at           │         │ assigned_by               │
  │ last_login_at          │         │ assigned_at               │
  └────────────────────────┘         └──────────────────────────┘
```

---

## ⚙️ Configuration

### Environment Variables

```dotenv
# ── Application ──────────────────────────────
APP_PORT=3000
APP_ENV=production                        # development | staging | production
APP_SECRET=<32-char-hex-secret>

# ── Database ─────────────────────────────────
DB_HOST=db.internal
DB_PORT=5432
DB_NAME=reg_db
DB_USER=reg_user
DB_PASS=<db-password>
DB_POOL_MIN=2
DB_POOL_MAX=20

# ── Redis ─────────────────────────────────────
REDIS_URL=redis://redis.internal:6379
REDIS_TTL_VERIFY=900                      # 15 min (seconds)

# ── Email ─────────────────────────────────────
SENDGRID_API_KEY=SG.xxxxxxxxxxxxxxxxxxxx
EMAIL_FROM=no-reply@yourdomain.com
EMAIL_VERIFY_TEMPLATE_ID=d-xxxx

# ── SMS (optional) ────────────────────────────
TWILIO_ACCOUNT_SID=ACxxxx
TWILIO_AUTH_TOKEN=xxxx
TWILIO_FROM_NUMBER=+1XXXXXXXXXX

# ── OAuth Providers ───────────────────────────
GOOGLE_CLIENT_ID=xxxx.apps.googleusercontent.com
GOOGLE_CLIENT_SECRET=xxxx
GITHUB_CLIENT_ID=xxxx
GITHUB_CLIENT_SECRET=xxxx

# ── Rate Limiting ─────────────────────────────
RATE_LIMIT_WINDOW_MS=60000
RATE_LIMIT_MAX=5

# ── Security ─────────────────────────────────
BCRYPT_ROUNDS=12
JWT_SECRET=<64-char-secret>
JWT_EXPIRES_IN=1h
CORS_ORIGIN=https://app.yourdomain.com
```

---

## 🔧 Troubleshooting

### Common Issues & Fixes

#### Issue 1 — Verification Email Not Received

```
Symptom:   User registers but never receives verification email.

Diagnosis:
  1. Check SENDGRID_API_KEY is valid
  2. Verify sender domain is authenticated in SendGrid
  3. Check spam/junk folder
  4. Query Redis: GET verify:{user_id}
     - If key missing → email was never sent
     - If key present → check email logs in SendGrid dashboard

Fix:
  - Re-send verification via POST /resend-verification
  - Check SendGrid Activity Feed for bounce/block reasons
  - Ensure MX records are properly configured
```

#### Issue 2 — "Email Already Exists" on First Registration

```
Symptom:   User sees 409 error on their first-ever signup attempt.

Diagnosis:
  SELECT id, status, created_at FROM users WHERE email = 'user@example.com';

Possible Causes:
  a) User previously registered but forgot
  b) Email was pre-seeded in test data
  c) Bot/duplicate submission

Fix:
  a) Prompt user to check for existing account / use "Forgot Password"
  b) Clean test data from production DB
  c) Review rate limiting & CAPTCHA integration
```

#### Issue 3 — Token Expired on Verification Click

```
Symptom:   User clicks email link but gets "Token Expired" error.

Cause:     15-minute TTL has elapsed.

Fix:
  1. Direct user to /resend-verification page
  2. Consider extending TTL for low-traffic environments:
     REDIS_TTL_VERIFY=3600  # 1 hour

Prevention:
  - Make verification email subject line urgent
  - Add countdown in email body
```

#### Issue 4 — OAuth Registration Fails

```
Symptom:   Google / GitHub login returns 500 or auth error.

Checklist:
  [ ] GOOGLE_CLIENT_ID and GOOGLE_CLIENT_SECRET are set
  [ ] Redirect URI in OAuth app matches:
      https://api.yourdomain.com/v2/reg/oauth/callback
  [ ] OAuth app is not in "Testing" mode with restricted users
  [ ] id_token is not expired (JWT exp claim)

Debug:
  Set LOG_LEVEL=debug to see full OAuth token exchange logs.
```

---

## 📊 Monitoring & Health

### Health Check Endpoint

```http
GET /health

Response 200:
{
  "status": "healthy",
  "version": "2.1.0",
  "checks": {
    "database": "ok",
    "redis": "ok",
    "email_service": "ok"
  },
  "uptime_seconds": 384920
}
```

### Key Metrics to Monitor

| Metric | Alert Threshold | Description |
|--------|----------------|-------------|
| `reg.registrations.total` | — | Total registrations (counter) |
| `reg.verification.success_rate` | < 80% | Email verification completion |
| `reg.api.latency_p99` | > 500ms | API response time |
| `reg.errors.rate` | > 5% | Error rate per minute |
| `reg.rate_limit.hits` | > 100/min | Possible abuse |

---

## 📎 Appendix

### Changelog

| Version | Date | Changes |
|---------|------|---------|
| 2.1.0 | May 2026 | Added Microsoft OAuth, enhanced audit logging |
| 2.0.0 | Jan 2026 | Microservice split, Redis for OTP |
| 1.5.0 | Aug 2025 | GitHub OAuth added |
| 1.0.0 | Mar 2025 | Initial release |

### Related Documentation

- [Authentication Service Docs](./AUTH.md)
- [User Management API](./USER_MGMT.md)
- [Security Policy](./SECURITY.md)
- [Deployment Guide](./DEPLOY.md)
- [API Postman Collection](./postman/REG_API.json)

---

*© 2026 YourOrganization. Internal documentation — do not distribute externally.*
