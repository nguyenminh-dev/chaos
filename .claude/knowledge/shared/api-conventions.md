# API Conventions

**Purpose**: Standard API conventions and patterns used across all WION microservices.

**Scope**: These conventions apply to **all services** in the WION ecosystem. Do not duplicate these in service-specific documentation.

---

## REST API Conventions

### URL Structure

**Base URL Pattern**: `https://{service}.{domain}.com/api/v{version}`

**Examples**:
```bash
✅ CORRECT:
https://billing.wion.vn/api/v1/wallets/{tenantId}/balance
https://billing.wion.vn/api/v1/credit/consume
https://tenant.wion.vn/api/v1/users/{userId}

❌ WRONG:
https://wion.vn/billing/api/wallets  # No version, unclear service
https://api.wion.vn/v1/wallets        # Service unclear
```

---

### Resource Naming

**Plural nouns for collections**:
```bash
✅ CORRECT:
GET  /api/v1/wallets/{tenantId}
POST /api/v1/credits/consume
GET  /api/v1/invoices

❌ WRONG:
GET  /api/v1/wallet/{tenantId}      # Singular
GET  /api/v1/getWallet             # Verb in URL
POST /api/v1/createInvoice         # Verb in URL
```

---

### HTTP Method Usage

**Standard methods**:
- `GET` - Read resource (idempotent, safe)
- `POST` - Create resource or trigger action
- `PUT` - Update resource (idempotent)
- `PATCH` - Partial update (may not be idempotent)
- `DELETE` - Delete resource (idempotent)

**Examples**:
```bash
✅ CORRECT:
GET    /api/v1/wallets/{tenantId}        # Get wallet
POST   /api/v1/credits/consume           # Consume credit
PUT    /api/v1/wallets/{tenantId}       # Update wallet
DELETE /api/v1/wallets/{tenantId}       # Delete wallet

❌ WRONG:
GET    /api/v1/wallets/create           # Wrong method
POST   /api/v1/wallets/{tenantId}       # Conflicting with GET
GET    /api/v1/deleteInvoice            # Wrong method
```

---

### Response Format

**Success Response**:
```json
{
  "data": {
    // Response data
  },
  "metadata": {
    "timestamp": "2026-07-05T10:00:00Z",
    "requestId": "req-12345",
    "version": "1.0"
  }
}
```

**Error Response**:
```json
{
  "error": {
    "code": "INSUFFICIENT_BALANCE",
    "message": "Wallet balance is insufficient",
    "details": {
      "requestedAmount": 1000,
      "availableBalance": 500
    }
  },
  "metadata": {
    "timestamp": "2026-07-05T10:00:00Z",
    "requestId": "req-12345"
  }
}
```

---

## Authentication Conventions

### API Key Authentication

**Headers**:
```http
Authorization: Bearer {api-key}
X-Tenant-Id: {tenant-id}
X-Request-Id: {request-id}
```

**Example**:
```bash
✅ CORRECT:
curl -X POST https://billing.wion.vn/api/v1/credits/consume \
  -H "Authorization: Bearer sk_test_12345" \
  -H "X-Tenant-Id: tenant-123" \
  -H "X-Request-Id: req-abc-123"

❌ WRONG:
curl -X POST https://billing.wion.vn/api/v1/credits/consume \
  -H "key: sk_test_12345"  # Non-standard header
```

---

### Service-to-Service Authentication

**Headers**:
```http
X-Internal-Auth: {service-token}
X-Source-Service: {service-name}
X-Request-Id: {request-id}
```

---

## Error Code Conventions

### Error Code Format

**Format**: `{CONCEPT}_{ERROR}`

**Examples**:
- `WALLET_NOT_FOUND` - Wallet does not exist
- `INSUFFICIENT_BALANCE` - Balance too low
- `IDEMPOTENCY_CONFLICT` - Duplicate idempotency key
- `INVALID_SIGNATURE` - Webhook signature invalid

---

### Standard Error Codes

| Error Code | HTTP Status | Description | Retryable? |
|------------|-------------|-------------|------------|
| `NOT_FOUND` | 404 | Resource not found | No |
| `INSUFFICIENT_BALANCE` | 400 | Balance too low | No |
| `IDEMPOTENCY_CONFLICT` | 409 | Duplicate key | No |
| `INVALID_SIGNATURE` | 403 | Auth failed | No |
| `VALIDATION_ERROR` | 400 | Input validation | No |
| `INTERNAL_ERROR` | 500 | Server error | Yes |
| `SERVICE_UNAVAILABLE` | 503 | Service down | Yes |
| `TIMEOUT` | 504 | Request timeout | Yes |

---

### Error Response Structure

```json
{
  "error": {
    "code": "INSUFFICIENT_BALANCE",
    "message": "Wallet balance is insufficient for requested operation",
    "details": {
      "requestedAmount": 1000,
      "availableBalance": 500,
      "currency": "VND",
      "tenantId": "tenant-123"
    },
    "help": "https://docs.wion.vn/errors/insufficient-balance"
  },
  "metadata": {
    "timestamp": "2026-07-05T10:00:00Z",
    "requestId": "req-12345",
    "version": "1.0"
  }
}
```

---

## Idempotency Conventions

### Idempotency Keys

**Purpose**: Safe retries of POST/PUT/PATCH operations

**Headers**:
```http
X-Idempotency-Key: {unique-key}
```

**Rules**:
- ✅ Use UUID format: `550e8400-e29b-41d4-a716-446655440000`
- ✅ Store idempotency key with result (24-48 hours)
- ✅ Return cached result on duplicate key
- ✅ Include `X-Idempotency-Key` in response

**Example**:
```bash
✅ CORRECT:
curl -X POST https://billing.wion.vn/api/v1/credits/consume \
  -H "X-Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000"

# Response on first call:
HTTP/1.1 200 OK
X-Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
X-Idempotency-Replayed: false

# Response on duplicate call:
HTTP/1.1 200 OK
X-Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
X-Idempotency-Replayed: true
```

---

## Pagination Conventions

### Pagination Format

**Request Parameters**:
```bash
GET /api/v1/invoices?page=0&size=20&sort=createdAt,desc
```

**Response Format**:
```json
{
  "data": [
    // Array of items
  ],
  "pagination": {
    "page": 0,
    "size": 20,
    "total": 150,
    "totalPages": 8,
    "hasNext": true,
    "hasPrevious": false
  },
  "metadata": {
    "timestamp": "2026-07-05T10:00:00Z",
    "requestId": "req-12345"
  }
}
```

---

### Cursor Pagination (for large datasets)

**Request Parameters**:
```bash
GET /api/v1/transactions?limit=100&cursor=eyJpZCI6IjEyMzQ1In0=
```

**Response Format**:
```json
{
  "data": [
    // Array of items
  ],
  "pagination": {
    "limit": 100,
    "nextCursor": "eyJpZCI6IjEyMzQ2In0=",
    "hasMore": true
  }
}
```

---

## Rate Limiting Conventions

### Rate Limit Headers

**Response Headers**:
```http
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 950
X-RateLimit-Reset: 1625520000
```

**Rate Limit Exceeded**:
```http
HTTP/1.1 429 Too Many Requests
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1625520000
Retry-After: 60
```

---

### Rate Limit Tiers

| Tier | Limit | Period | Usage |
|------|-------|--------|-------|
| **Free** | 100 requests/minute | 60s | Trial users |
| **Standard** | 1,000 requests/minute | 60s | Paid users |
| **Premium** | 10,000 requests/minute | 60s | Enterprise |

---

## Versioning Conventions

### API Versioning

**URL Versioning**:
```bash
✅ CORRECT:
https://billing.wion.vn/api/v1/wallets/{tenantId}
https://billing.wion.vn/api/v2/wallets/{tenantId}  # Future version

❌ WRONG:
https://billing.wion.vn/api/wallets/{tenantId}  # No version
```

---

### Version Deprecation

**Deprecation Timeline**:
- **Announcement**: 6 months before deprecation
- **Warning Period**: 3 months (return `Deprecation` header)
- **Sunset**: Remove old version

**Deprecation Header**:
```http
X-API-Deprecation: This API version is deprecated. See https://docs.wion.vn/api/v2
Sunset: 2026-12-31
```

---

## Webhook Conventions

### Webhook Delivery

**Webhook Headers**:
```http
X-Webhook-Id: {webhook-id}
X-Webhook-Timestamp: {unix-timestamp}
X-Webhook-Signature: {hmac-sha256-signature}
X-Gateway: {gateway-name}
```

**Webhook Signature**:
```bash
# Generate signature
signature = HMAC-SHA256(webhook_secret, payload + timestamp)

# Verify signature
expected_signature = HMAC-SHA256(webhook_secret, payload + timestamp)
if (signature != expected_signature) {
  reject_webhook();
}
```

---

### Webhook Retry Policy

**Standard Retry**:
- **Max retries**: 3
- **Backoff**: Exponential (1s, 2s, 4s)
- **Timeout**: 30 seconds per attempt

**Dead Letter Queue**:
- After 3 failed attempts
- Manual retry available
- 7-day retention

---

## API Documentation Conventions

### API Documentation Structure

**Every API must document**:
```markdown
## {API Name}

### Endpoint
`METHOD /api/v{version}/{path}`

### Purpose
{What this API does}

### Domain Concepts
This API operates on the [{Aggregate}](../domains/{domain}/aggregate.md).

### Business Rules
For detailed rules, see [{Aggregate} business rules](../domains/{domain}/business-rules.md).

### Request
**Headers**:
- Authorization
- X-Tenant-Id
- X-Idempotency-Key (if applicable)

**Body**: {Request schema}

### Response
**Success (200 OK)**: {Response schema}
**Error (4xx/5xx)**: {Error schema}

### Error Codes
| Code | Description | Retryable |
|------|-------------|-----------|
| NOT_FOUND | Resource not found | No |
| INSUFFICIENT_BALANCE | Balance too low | No |

### Events Published
Successful calls publish:
- [{Event}](../domains/{domain}/domain-events.md)

### Examples
{Concrete examples}
```

---

## Quick Reference

### API Naming

| Concept | Convention | Example |
|---------|------------|---------|
| **Resource** | Plural noun | `/wallets`, `/invoices` |
| **Action** | Verb in noun | `/credits/consume` |
| **ID** | Singular | `/{tenantId}`, `/{walletId}` |
| **Query** | Parameters | `?page=0&size=20` |

---

### HTTP Methods

| Method | Purpose | Idempotent | Safe |
|--------|---------|------------|------|
| **GET** | Read | ✅ Yes | ✅ Yes |
| **POST** | Create/Action | ❓ Maybe | ❌ No |
| **PUT** | Update | ✅ Yes | ❌ No |
| **PATCH** | Partial update | ❓ Maybe | ❌ No |
| **DELETE** | Delete | ✅ Yes | ❌ No |

---

### Status Codes

| Code | Meaning | Usage |
|------|---------|-------|
| **200** | Success | Operation succeeded |
| **201** | Created | Resource created |
| **204** | No Content | Successful, no return data |
| **400** | Bad Request | Invalid input |
| **401** | Unauthorized | Missing/invalid auth |
| **403** | Forbidden | Auth succeeded, not authorized |
| **404** | Not Found | Resource not found |
| **409** | Conflict | Duplicate/consistency issue |
| **429** | Too Many Requests | Rate limited |
| **500** | Internal Error | Server error |
| **503** | Unavailable | Service down |

---

## Related Documentation

- [DDD Conventions](./ddd-conventions.md) - Domain-driven design patterns
- [Event Conventions](./event-conventions.md) - Event patterns

---

**Convention Version**: 1.0  
**Last Updated**: 2026-07-05  
**Applies To**: All WION microservices
