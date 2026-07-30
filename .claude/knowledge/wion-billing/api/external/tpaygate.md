# T-PayGate API Documentation

## Overview
Complete API reference for T-PayGate payment gateway integration. This is a third-party API specification where our system is the consumer and T-PayGate is the payment gateway provider.

**Important**: T-PayGate is a **payment gateway provider/aggregator**, not a payment provider itself. T-PayGate abstracts multiple banking partners and provides a unified API for:
- Bank connection management
- QR code generation and routing  
- Payment notification aggregation
- Bank API abstraction

**API Version**: v1  
**Base URLs**:
- Staging: `https://t-paygate.tpos.dev`
- Production: `https://t-paygate.tpos.app`

**Documentation Source**: [T-PayGate Integration Specification](../../../../../stories/billing-service/t-paygate-tai-lieu-tich-hop-cho-doi-tac-thu-3.md)

---

## Authentication

### OAuth 2.0 Client Credentials Flow

#### Token Request
```http
POST /api/v1/oauth/token
Content-Type: application/x-www-form-urlencoded

clientId={clientId}&tenantId={tenantId}&source={source}
```

#### Response (200 OK)
```json
{
  "accessToken": "eyJhbGciOiJIUzI1NiIsInR...",
  "expiresIn": 3600
}
```

#### Usage in API Calls
```http
Authorization: Bearer {accessToken}
```

**Important**:
- Token TTL: 3600 seconds (60 minutes)
- Refresh at T-5 minutes
- Do NOT call per request (rate limited)
- Cache token in memory

---

## API Endpoints

### 1. Bank Connection APIs

#### 1.1 Connect Bank (API-based)

```http
POST /api/v1/public-api/config-bank/connect
Authorization: Bearer {token}
Content-Type: application/json
```

**Request Body**:
```json
{
  "bankCode": "VCB",
  "merchantName": "Cửa hàng A",
  "accountName": "NGUYEN VAN A",
  "accountNo": "108800888060",
  "identity": "012345678901",
  "phone": "0987654321",
  "email": "shop@example.com",
  "prefix": "merchant_xxx",
  "clientId": "provider_xxx"
}
```

**Response (200 OK)**:
```json
{
  "success": true,
  "results": {
    "configBankId": "abc-uuid",
    "isConnected": false,
    "isOTPConfirmation": true
  }
}
```

**Response (400 Bad Request)**: Invalid input
**Response (401 Unauthorized)**: Invalid/expired token
**Response (409 Conflict)**: Duplicate connection

---

#### 1.2 Confirm OTP (if required)

```http
POST /api/v1/public-api/config-bank/confirm
Authorization: Bearer {token}
Content-Type: application/json
```

**Request Body**:
```json
{
  "configBankId": "abc-uuid",
  "otpNumber": "123456"
}
```

**Response (200 OK)**:
```json
{
  "success": true,
  "results": {
    "configBankId": "abc-uuid",
    "vaNumber": "9XYZ200309052356",
    "isConnected": true
  }
}
```

**Response (400 Bad Request)**: Invalid OTP
**Response (401 Unauthorized)**: Expired OTP

---

#### 1.3 List Bank Connections

```http
GET /api/v1/public-api/config-bank/list?bankCode={optional}
Authorization: Bearer {token}
```

**Response (200 OK)**:
```json
[
  {
    "configBankId": "abc-uuid",
    "bankCode": "VCB",
    "accountNo": "108800888060",
    "accountName": "NGUYEN VAN A",
    "vaNumber": "9XYZ200309052356",
    "merchantId": "INTERNAL_ID",
    "clientId": "provider_xxx",
    "phone": "0987654321",
    "isConnected": true,
    "urlLogo": "https://.../logo.png",
    "dateCreated": "2026-05-01T08:00:00Z"
  }
]
```

---

#### 1.4 Disconnect Bank

```http
POST /api/v1/public-api/config-bank/disconnect
Authorization: Bearer {token}
Content-Type: application/json
```

**Request Body**:
```json
{
  "configBankId": "abc-uuid"
}
```

**Response (200 OK)**: Success confirmation

**Important**: Soft delete - record preserved for reconciliation

---

#### 1.5 Get Supported Banks

```http
GET /api/v1/public-api/bank
```

**Response (200 OK)**:
```json
[
  {
    "code": "VCB",
    "name": "Vietcombank",
    "logo": "https://...",
    "bin": "970422",
    "napasCode": "NAPAS_VCB"
  }
]
```

**Important**: Do NOT hardcode bank list - use this endpoint

---

### 2. Bill/Payment APIs

#### 2.1 Create Bill

```http
POST /api/v1/public-api/order/bill
Authorization: Bearer {token}
Content-Type: application/json
x-client-id: {clientId}
x-tenant-id: {tenantId}
x-source: {source}
x-config-id: {configBankId}
```

**Request Body**:
```json
{
  "refTransactionId": "ORDER-2026-0001",
  "amount": 250000,
  "description": "Thanh toan don hang 0001"
}
```

**Response (200 OK)**:
```json
{
  "success": true,
  "results": {
    "billCode": "B2026050500001",
    "vaNumber": "9XYZ200309052356",
    "amount": 250000,
    "qrContent": "00020101021238...",
    "qrImageBase64": "iVBORw0KGgo...",
    "expiredAt": "2026-05-05T15:30:00Z"
  }
}
```

**Response (409 Conflict)**: Duplicate refTransactionId (returns existing bill)

---

#### 2.2 Query Bill by Reference Transaction ID

```http
GET /api/v1/public-api/order/get-refTransactionId?refTransactionId=ORDER-2026-0001
Authorization: Bearer {token}
```

**Response (200 OK)**: Array of bills with same refTransactionId

---

#### 2.3 Query Bill by Bill Code

```http
GET /api/v1/public-api/order/get-billCode?billCode=B2026050500001
Authorization: Bearer {token}
```

**Response (200 OK)**: Single bill details with current status

---

### 3. Embedded UI APIs

#### 3.1 Open Connect UI

```http
GET /api/v1/public-api/view/connect?bankCode={bankCode}&token={access_token}
```

**Response**: HTML page with bank-specific form

---

#### 3.2 Submit OTP (Embedded UI)

```http
POST /api/v1/public-api/view/otp
```

**Request Body**: OTP and verification metadata

---

## Webhook API

### Webhook Endpoint (We Provide)

Our system provides a webhook endpoint for T-PayGate to call when payments are completed.

**Endpoint**: HTTPS URL registered during onboarding  
**Method**: POST  
**Content-Type**: application/json  
**Timeout**: 10 seconds (PROD), 30 seconds (UAT)

#### Webhook Payload v1

```json
{
  "RefTransactionId": "TXN202606030001",
  "BillCode": "HD20260603001",
  "Amount": 1500000.00,
  "VirtualAccount": "970422000123456789",
  "ActualAccount": "1234567890123",
  "PaymentTime": "2026-06-03T14:30:00"
}
```

#### Webhook Payload v2 (Pending Release)

```json
{
  "EventType": "PAYMENT_RECEIVED",
  "Timestamp": "2026-05-05T08:30:00Z",
  "Data": {
    "RefTransactionId": "TXN202606030001",
    "BillCode": "HD20260603001",
    "Amount": 1500000.00,
    "VirtualAccount": "970422000123456789",
    "ActualAccount": "1234567890123",
    "PaymentTime": "2026-06-03T14:30:00"
  }
}
```

#### Expected Response

**HTTP 200 OK** with any body (e.g., `{"MessageError": true}`)

**Failure Handling**: T-PayGate retries 3 times × 5-minute intervals

---

## Error Codes

### HTTP Status Codes

| Code | Meaning | Retryable |
|------|---------|-----------|
| 200 | Success | N/A |
| 400 | Bad Request / Validation Error | No |
| 401 | Unauthorized (invalid/expired token) | No (refresh token) |
| 403 | Forbidden (no permission / invalid signature) | No |
| 404 | Resource Not Found | No |
| 409 | Conflict (duplicate refTransactionId) | No (return cached) |
| 429 | Rate Limit Exceeded | Yes (with backoff) |
| 5xx | Server Error | Yes (with backoff) |

### Business Error Codes

| Code | Meaning |
|------|---------|
| 0 / 00 | Success |
| 01 | Invalid Signature |
| 02 | Customer/Bill Not Found |
| 03 | Bank Business Rule Error |
| 05 | Duplicate Transaction ID |
| 99 | Unknown Error |

---

## Rate Limiting

**Documented Limits**:
- `/oauth/token`: Rate limited (exact limits not specified)
- All other endpoints: Standard limits (not specified)

**Best Practices**:
- Token reuse mandatory
- Implement backoff on 429 responses
- Monitor rate limit errors

---

## Headers Reference

### Standard Headers

```http
Authorization: Bearer {accessToken}
Content-Type: application/json
```

### Custom Headers (Bill Creation)

```http
x-client-id: {clientId}
x-tenant-id: {tenantId}
x-source: {source}
x-config-id: {configBankId}
```

---

## Idempotency

### Client-Side Idempotency (We Generate)

- **Key**: `refTransactionId`
- **Scope**: Per merchant (clientId + tenantId)
- **Duration**: 24 hours (T-PayGate cache)
- **Behavior**: Duplicate requests return existing bill or HTTP 409

### Server-Side Idempotency (We Implement)

- **Key**: `billCode` / `bankTransId`
- **Scope**: Our system database
- **Duration**: 24 hours minimum
- **Behavior**: Check processing status before applying payment

---

## Environment Configuration

### Staging (UAT)
- **URL**: `https://t-paygate.tpos.dev`
- **Webhook Timeout**: 30 seconds
- **Purpose**: Integration testing

### Production
- **URL**: `https://t-paygate.tpos.app`
- **Webhook Timeout**: 10 seconds
- **Purpose**: Live payment processing

### Required Credentials
- `clientId` - OAuth client identifier
- `tenantId` - OAuth tenant identifier
- `source` - Channel identifier

---

## API Integration Guidelines

### 1. Token Management
```csharp
// Proactive refresh (recommended)
if (IsTokenExpiringSoon(minutes: 5))
{
    await RefreshTokenAsync();
}

// Reactive refresh (fallback)
try
{
    response = await CallApiAsync();
}
catch (ApiException ex) when (ex.StatusCode == 401)
{
    await RefreshTokenAsync();
    response = await CallApiAsync(); // Retry
}
```

### 2. Retry Policy
```csharp
// Retry configuration
var retryPolicy = Policy
    .Handle<ApiException>(ex => ex.StatusCode >= 500 || ex.StatusCode == 429)
    .WaitAndRetryAsync(
        retryCount: 3,
        sleepDurationProvider: attempt => TimeSpan.FromSeconds(Math.Pow(2, attempt)),
        onRetry: (exception, delay, retryCount, context) =>
        {
            Logger.LogWarning($"Retry {retryCount} after {delay.TotalSeconds}s delay");
        }
    );
```

### 3. Circuit Breaker
```csharp
// Circuit breaker configuration
var circuitBreaker = Policy
    .Handle<ApiException>()
    .CircuitBreakerAsync(
        exceptionsAllowedBeforeBreaking: 5,
        durationOfBreak: TimeSpan.FromSeconds(30)
    );
```

### 4. Idempotency Implementation
```csharp
// Bill creation idempotency
public async Task<Bill> CreateBillAsync(CreateBillCommand command)
{
    // Check for existing bill
    var existingBill = await _billRepository.FindByRefTransactionIdAsync(command.RefTransactionId);
    if (existingBill != null)
    {
        return existingBill; // Idempotent
    }
    
    // Create new bill
    return await _tpaygateClient.CreateBillAsync(command);
}
```

---

## Testing

### Postman Collections
T-PayGate provides collections during onboarding:
- `t-paygate-staging.postman_collection.json`
- `t-paygate-prod.postman_collection.json`

### Manual Testing
1. Test OAuth flow with valid credentials
2. Test bank connection with OTP
3. Test bill creation and QR generation
4. Test webhook delivery (use test webhooks)
5. Test idempotency with duplicate requests

### Automated Testing
See [T-PayGate Testing Scenarios](../../reference/tpaygate-testing.md) for comprehensive test coverage.

---

## Related Documents
- [T-PayGate Domain Overview](../../domains/payment/tpaygate/overview.md) - Domain context
- [T-PayGate API Client Implementation](../../infrastructure/tpaygate/api-client.md) - Client implementation
- [T-PayGate Resiliency Patterns](../../infrastructure/tpaygate-resiliency.md) - Retry and circuit breaker patterns
- [T-PayGate Implementation Checklist](../../reference/tpaygate-implementation-checklist.md) - Implementation checklist
