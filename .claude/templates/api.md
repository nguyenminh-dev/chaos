# API Documentation Template

**Purpose**: Template for documenting API endpoints following API conventions.

**When to Use**: When documenting REST API endpoints.

---

## API Documentation Structure

Every API endpoint must follow this structure:

```markdown
## {HTTP Method} {Endpoint Path}

### Purpose
{What this API endpoint does}

### Request

**Headers**:
```
Content-Type: application/json
Authorization: Bearer {api-key}
X-Tenant-Id: {tenant-id}
```

**Path Parameters**:
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| {param1} | {type} | Yes/No | {description} |
| {param2} | {type} | Yes/No | {description} |

**Query Parameters**:
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| {param1} | {type} | Yes/No | {description} |

**Request Body** (for POST/PUT):
```json
{
  "{property1}": "{value1}",
  "{property2}": "{value2}",
  "{property3}": "{value3}"
}
```

### Response

**Success Response** (200 OK):
```json
{
  "{responseProperty1}": "{value1}",
  "{responseProperty2}": "{value2}",
  "{responseProperty3}": "{value3}"
}
```

**Error Responses**:

| HTTP Code | Error Type | Description | Retryable |
|-----------|------------|-------------|-----------|
| 400 | ValidationError | Invalid input | No |
| 404 | NotFound | Resource not found | No |
| 500 | InternalError | Server error | Yes |

### Domain References

**Business Rules Enforced**:
- [{Domain} Business Rules](../domains/{domain}/business-rules.md) - {Which rules}
- Specifically: BR-{LETTER}-{NUMBER}, BR-{LETTER}-{NUMBER}

**Domain Events Published**:
- [{Event}](../domains/{domain}/domain-events.md#{event}) - {When published}

### Use Case
This API implements: [{Use Case}](../application/use-cases/{use-case}.md)

### Example

**Request**:
```bash
curl -X {HTTP} "https://billing.wion.vn/api/v1/{endpoint}" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer {api-key}" \
  -H "X-Tenant-Id: {tenant-id}" \
  -d '{request_body}'
```

**Response**:
```json
{
  "success": true,
  "data": {response_data}
}
```
```

---

## Critical Principles

### 1. Reference Domain, Don't Duplicate

**❌ WRONG**: Duplicating business rules in API docs

```markdown
## Validation Rules
- Balance must be sufficient ❌ WRONG
- Amount must be positive ❌ WRONG
```

**✅ CORRECT**: Referencing domain documentation

```markdown
## Domain References
### Business Rules Enforced
- [Wallet Business Rules](../domains/wallet/business-rules.md)
- Specifically: BR-W-003 (Sufficient balance)
```

### 2. API Documentation Responsibilities

**API Docs OWN**:
- ✅ Endpoint contract
- ✅ Request/response format
- ✅ Error codes
- ✅ Authentication
- ✅ Rate limits

**API Docs REFERENCE**:
- ❌ NOT business rules
- ❌ NOT domain model
- ❌ NOT use case logic

---

## Example: Credit Consumption API

```markdown
## POST /api/v1/credit/consume

### Purpose
Consume credits from user wallet for service usage

### Request

**Headers**:
```
Content-Type: application/json
Authorization: Bearer {api-key}
X-Tenant-Id: {tenant-id}
X-Idempotency-Key: {unique-key}
```

**Request Body**:
```json
{
  "userId": "user-123",
  "amount": 50000,
  "service": "AI-Services",
  "referenceId": "inv-456"
}
```

### Response

**Success Response** (200 OK):
```json
{
  "success": true,
  "transactionId": "txn-789",
  "remainingBalance": 50000,
  "consumedAt": "2026-07-27T10:00:00Z"
}
```

**Error Responses**:

| HTTP Code | Error Type | Description | Retryable |
|-----------|------------|-------------|-----------|
| 400 | INVALID_AMOUNT | Amount must be positive | No |
| 400 | INSUFFICIENT_BALANCE | Not enough balance | No |
| 409 | IDEMPOTENCY_CONFLICT | Duplicate idempotency key | No (return cached) |
| 404 | WALLET_NOT_FOUND | Wallet does not exist | No |
| 500 | INTERNAL_ERROR | Server error | Yes |

### Domain References

**Business Rules Enforced**:
- [Wallet Business Rules](../domains/wallet/business-rules.md)
  - Specifically: BR-W-003 (Sufficient balance)
- [CreditTransaction Business Rules](../domains/credit-transaction/business-rules.md)
  - Specifically: BR-C-001 (Positive amount), BR-C-002 (Unique idempotency)

**Domain Events Published**:
- [CreditConsumedDomainEvent](../domains/credit-transaction/domain-events.md#credit-consumed) - On successful consumption
- [InsufficientBalanceDomainEvent](../domains/wallet/domain-events.md#insufficient-balance) - On balance check failure

### Use Case
This API implements: [Consume Credit](../application/use-cases/consume-credit.md)

### Example

**Request**:
```bash
curl -X POST "https://billing.wion.vn/api/v1/credit/consume" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer your-api-key" \
  -H "X-Tenant-Id: tenant-123" \
  -H "X-Idempotency-Key: unique-request-key-123" \
  -d '{
    "userId": "user-456",
    "amount": 50000,
    "service": "AI-Services",
    "referenceId": "inv-789"
  }'
```

**Success Response**:
```json
{
  "success": true,
  "transactionId": "txn-456",
  "remainingBalance": 50000,
  "consumedAt": "2026-07-27T10:15:30Z"
}
```

**Insufficient Balance Error**:
```json
{
  "success": false,
  "error": {
    "code": "INSUFFICIENT_BALANCE",
    "message": "Wallet has insufficient balance",
    "details": {
      "requested": 50000,
      "available": 30000,
      "shortfall": 20000
    }
  }
}
```

### Idempotency
Duplicate requests with the same `X-Idempotency-Key` header will return the cached result without consuming additional credits.

### Rate Limits
- 1,000 requests per minute per tenant
- 100 concurrent requests per tenant

### Authentication
- API Key authentication required
- Tenant context must be provided
- Service-to-service calls use internal authentication
```

---

## Common API Patterns

### GET Request Pattern

```markdown
## GET /api/v1/{resource}/{id}

### Purpose
Retrieve {resource} by ID

### Request

**Path Parameters**:
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| id | string | Yes | Unique identifier |

**Query Parameters**:
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| include | string | No | Related data to include |

### Response

**Success Response** (200 OK):
```json
{
  "{resource}": {
    "id": "{id}",
    "{property1}": "{value1}"
  }
}
```

**Error Responses**:

| HTTP Code | Error Type | Description |
|-----------|------------|-------------|
| 404 | NOT_FOUND | Resource not found |
```

### POST Request Pattern

```markdown
## POST /api/v1/{resource}

### Purpose
Create new {resource}

### Request

**Request Body**:
```json
{
  "{property1}": "{value1}",
  "{property2}": "{value2}"
}
```

### Response

**Success Response** (201 Created):
```json
{
  "id": "{new-id}",
  "{property1}": "{value1}",
  "createdAt": "2026-07-27T10:00:00Z"
}
```

**Error Responses**:

| HTTP Code | Error Type | Description |
|-----------|------------|-------------|
| 400 | VALIDATION_ERROR | Invalid input |
| 409 | CONFLICT | Resource already exists |
```

### PUT Request Pattern

```markdown
## PUT /api/v1/{resource}/{id}

### Purpose
Update existing {resource}

### Request

**Path Parameters**:
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| id | string | Yes | Unique identifier |

**Request Body**:
```json
{
  "{property1}": "{newValue1}",
  "{property2}": "{newValue2}"
}
```

### Response

**Success Response** (200 OK):
```json
{
  "id": "{id}",
  "{property1}": "{newValue1}",
  "updatedAt": "2026-07-27T10:00:00Z"
}
```

**Error Responses**:

| HTTP Code | Error Type | Description |
|-----------|------------|-------------|
| 400 | VALIDATION_ERROR | Invalid input |
| 404 | NOT_FOUND | Resource not found |
```

### DELETE Request Pattern

```markdown
## DELETE /api/v1/{resource}/{id}

### Purpose
Delete {resource}

### Request

**Path Parameters**:
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| id | string | Yes | Unique identifier |

### Response

**Success Response** (204 No Content)

**Error Responses**:

| HTTP Code | Error Type | Description |
|-----------|------------|-------------|
| 404 | NOT_FOUND | Resource not found |
```

---

## API Documentation Checklist

Before marking API documentation as complete:

- [ ] Purpose clearly defined
- [ ] Request format documented
- [ ] Response format documented
- [ ] Error responses documented
- [ ] Domain References section included
- [ ] Business Rules referenced (not duplicated)
- [ ] Domain Events referenced (not duplicated)
- [ ] Use Case linked
- [ ] Examples provided
- [ ] Authentication requirements specified
- [ ] Rate limits specified (if applicable)
- [ ] Idempotency explained (if applicable)
- [ ] All links resolve correctly

---

## Quick Reference

### "What goes in API documentation?"

| Section | Content | Source |
|---------|---------|--------|
| **Endpoint Contract** | URL, method, parameters | API docs (own this) |
| **Request/Response** | JSON formats, examples | API docs (own this) |
| **Error Codes** | HTTP codes, error types | API docs (own this) |
| **Domain References** | Links to domain docs | Domain docs (reference) |
| **Business Rules** | Which rules apply | Domain docs (reference) |
| **Use Cases** | Which use case implements | Use case docs (reference) |

---

**Template Version**: 1.0  
**Last Updated**: 2026-07-27  
**Based On**: API Conventions and Wion Engineering Rules