# BiaZap API -- Authentication Reference

**Base URL:** `https://biazap.biasofia.com` (production) | `http://localhost:8088` (development)
**Format:** JSON (`Content-Type: application/json`)

## Quick Register (Recommended for Agent Onboarding)

The fastest way to create an account. Only requires an email:

```
POST /v1/auth/quick-register
```

```bash
curl -X POST https://biazap.biasofia.com/v1/auth/quick-register \
  -H "Content-Type: application/json" \
  -d '{"email": "dev@example.com"}'
```

**Response (201):**
```json
{
  "company_id": 1,
  "api_key": "bza_a1b2c3d4e5f6...",
  "email": "dev@example.com",
  "password": "4f8b2c1a9e3d7f06",
  "base_url": "https://biazap.biasofia.com",
  "message": "account created — credentials sent to your email"
}
```

Auto-generates: company name (from email domain), owner name, 16-char password, API key.
Sends welcome email via Resend with credentials and quick start guide.

**IMPORTANT for LLM agents:** Ask the user for their email, call this endpoint, then use the returned `api_key` for all subsequent API calls. The password is for dashboard login only.

## Authentication Methods

BiaZap supports two authentication methods: JWT Bearer tokens and API keys.

### JWT Bearer Token

Short-lived token obtained via `POST /v1/auth/login`. HS256, default 15-minute expiry.

```
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...
```

Claims: `company_id`, `user_id`, `role`, `is_superadmin`.

On every request, the server re-validates that the company exists and is `active`. Suspended companies receive `403`.

### API Key

Long-lived key obtained via `POST /v1/tokens`. Format: `bza_` prefix + 60 hex characters. Stored as SHA256 hash (never plaintext). Revocable at any time.

```
X-API-Key: bza_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

## Roles

Hierarchy: `owner` > `admin` > `agent`

| Resource              | owner | admin | agent |
|-----------------------|:-----:|:-----:|:-----:|
| Tokens & Users        |   x   |   -   |   -   |
| Instances (CRUD)      |   x   |   x   |   -   |
| Messages              |   x   |   x   |   x   |
| Chats & Contacts      |   x   |   x   |   x   |
| Groups                |   x   |   x   |   x   |
| Media                 |   x   |   x   |   x   |
| Queue                 |   x   |   x   |   -   |
| Webhooks              |   x   |   x   |   -   |
| Admin (superadmin)    | superadmin only       |

## Rate Limiting

Redis-backed, per-company. Default: 60 req/min (configurable per plan). Response headers:

- `X-RateLimit-Limit` -- allowed per minute
- `X-RateLimit-Remaining` -- remaining in current window
- `X-RateLimit-Reset` -- Unix timestamp of reset
- `Retry-After` -- seconds to wait (on 429)

---

## Endpoints

### POST /v1/auth/register

Create a new company (tenant) with an owner user. No auth required.

```bash
curl -X POST https://biazap.biasofia.com/v1/auth/register \
  -H 'Content-Type: application/json' \
  -d '{
    "company_name": "My Company",
    "owner_email": "owner@example.com",
    "owner_name": "Jane Doe",
    "password": "MySecurePass123"
  }'
```

| Field          | Type   | Required | Notes                  |
|----------------|--------|:--------:|------------------------|
| `company_name` | string |    yes   |                        |
| `owner_email`  | string |    yes   |                        |
| `owner_name`   | string |    yes   |                        |
| `password`     | string |    yes   | Minimum 12 characters  |

**Response 201:**
```json
{
  "company_id": 1,
  "token": "bza_xxxx...xxxx",
  "api_token": "bza_xxxx...xxxx",
  "token_id": 1,
  "last4": "xxxx",
  "slug": "my-company",
  "message": "company registered successfully"
}
```

The `token` returned is a bootstrap **API key** (`bza_...`), not a JWT. To start a browser session, call `POST /v1/auth/login`.

**Errors:** `400` missing fields or short password, `409` email or company already exists.

### POST /v1/auth/login

Authenticate a user and receive a JWT. No auth required.

```bash
curl -X POST https://biazap.biasofia.com/v1/auth/login \
  -H 'Content-Type: application/json' \
  -d '{
    "email": "owner@example.com",
    "password": "MySecurePass123"
  }'
```

**Response 200:**
```json
{
  "token": "eyJhbGciOiJIUzI1NiIs...",
  "company_id": 1,
  "user_id": 1,
  "role": "owner",
  "email": "owner@example.com",
  "is_superadmin": false
}
```

Also sets session cookies (see Browser Auth below).

**Protections:** account lockout after 5 failed attempts in 15 minutes; JWT includes `jti` for revocation on logout; refresh token rotates on each use.

**Errors:** `401` invalid credentials, `403` company suspended, `429` account temporarily locked.

### POST /v1/auth/refresh

Rotate the refresh token and receive a new JWT access token.

**Auth:** Cookie `biazap_refresh_token` (set by login).

```bash
curl -X POST https://biazap.biasofia.com/v1/auth/refresh \
  -H 'X-CSRF-Token: <csrf_token_value>' \
  --cookie 'biazap_refresh_token=<refresh_token_value>'
```

**Response 200:** Same payload as login.

**Errors:** `401` missing/invalid/expired refresh token, `403` CSRF validation failure.

### POST /v1/auth/logout

Revoke the current JWT, invalidate refresh token, clear session cookies.

**Auth:** JWT Bearer token.

```bash
curl -X POST https://biazap.biasofia.com/v1/auth/logout \
  -H 'Authorization: Bearer eyJhbGciOiJIUzI1NiIs...' \
  -H 'X-CSRF-Token: <csrf_token_value>'
```

**Response:** `204 No Content`.

### POST /v1/tokens

Create a new API key. **Auth:** Owner role required.

```bash
curl -X POST https://biazap.biasofia.com/v1/tokens \
  -H 'Authorization: Bearer eyJhbGciOiJIUzI1NiIs...' \
  -H 'Content-Type: application/json' \
  -d '{
    "label": "production",
    "role": "agent"
  }'
```

| Field   | Type   | Required | Notes                                       |
|---------|--------|:--------:|---------------------------------------------|
| `label` | string |    yes   | Identifier for the token                    |
| `role`  | string |    no    | `owner`, `admin`, or `agent` (default: `agent`) |

**Response 201:**
```json
{
  "token_id": 2,
  "token": "bza_xxxx...xxxx",
  "label": "production",
  "role": "agent",
  "last4": "xxxx",
  "created_at": "2026-03-07T10:00:00Z"
}
```

**The token is shown only once.** Store it securely.

**Other token endpoints:**
- `GET /v1/tokens` -- list all tokens (owner only, shows `last4`, never full token)
- `DELETE /v1/tokens/{tokenId}` -- revoke a token permanently (owner only)

---

## Browser Auth (Cookies + CSRF)

For browser-based clients, login sets two cookies:

| Cookie                  | Purpose                          | Flags      |
|-------------------------|----------------------------------|------------|
| `biazap_refresh_token`  | Silent token refresh             | `HttpOnly` |
| `biazap_csrf_token`     | CSRF protection for mutations    | Readable   |

**Flow:**
1. Call `POST /v1/auth/login` -- receive JWT in response body + cookies
2. Store JWT in memory (not localStorage)
3. On unsafe methods (POST, PUT, PATCH, DELETE), send `X-CSRF-Token` header with the value from the `biazap_csrf_token` cookie
4. When JWT expires, call `POST /v1/auth/refresh` -- returns new JWT, rotates cookies

---

## Quick Start

Register, get an API key, and make your first authenticated call:

```bash
# 1. Register a company (returns a bootstrap API key)
API_KEY=$(curl -sf -X POST http://localhost:8088/v1/auth/register \
  -H 'Content-Type: application/json' \
  -d '{
    "company_name": "Demo",
    "owner_email": "dev@demo.com",
    "owner_name": "Dev",
    "password": "DemoPassword123"
  }' | jq -r '.token')

# 2. Use the API key to list instances
curl -sf http://localhost:8088/v1/instances \
  -H "X-API-Key: $API_KEY" | jq

# Or: login for a JWT instead
TOKEN=$(curl -sf -X POST http://localhost:8088/v1/auth/login \
  -H 'Content-Type: application/json' \
  -d '{
    "email": "dev@demo.com",
    "password": "DemoPassword123"
  }' | jq -r '.token')

# 3. Use JWT to list instances
curl -sf http://localhost:8088/v1/instances \
  -H "Authorization: Bearer $TOKEN" | jq
```
