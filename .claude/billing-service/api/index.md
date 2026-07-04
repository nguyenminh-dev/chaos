# Wallet Management - API

## Get Wallet Balance

### Endpoint
```
GET /api/v1/wallets/{tenantId}/balance
```

### Purpose
Retrieve current wallet balance including available, reserved, and asset breakdown.

### Request
**Headers**:
- `Authorization: Bearer {api-key}`
- `X-Tenant-Id: {tenant-id}`

**Path Parameters**:
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| tenantId | string | Yes | Tenant identifier |

### Response
**Success (200 OK)**:
```json
{
  "tenantId": "tenant-12345",
  "availableBalance": 100000,
  "reservedBalance": 5000,
  "totalBalance": 105000,
  "currency": "VND",
  "assets": [
    {
      "assetType": "WI_CREDIT",
      "balance": 80000,
      "reservedBalance": 5000
    },
    {
      "assetType": "PROMOTION",
      "balance": 20000,
      "reservedBalance": 0,
      "expiresAt": "2026-12-31T23:59:59Z"
    }
  ]
}
```

### Errors
| Code | HTTP | Description |
|------|------|-------------|
| WALLET_NOT_FOUND | 404 | Wallet does not exist |
| ACCESS_DENIED | 403 | Cannot access other tenant balance |

---

## Credit Consumption - API

## Consume Credit

### Endpoint
```
POST /api/v1/credit/consume
```

### Purpose
Consume credits from wallet for service usage.

### Request
**Headers**:
- `Authorization: Bearer {api-key}`
- `X-Idempotency-Key: {unique-key}`

**Body**:
```json
{
  "tenantId": "tenant-12345",
  "amount": 1000,
  "service": "AI_GENERATION",
  "referenceId": "req-12345",
  "idempotencyKey": "unique-key-123",
  "metadata": {
    "resourceType": "TOKEN",
    "quantity": 1000
  }
}
```

### Response
**Success (200 OK)**:
```json
{
  "transactionId": "txn-abc-123",
  "status": "COMPLETED",
  "amount": 1000,
  "remainingBalance": 99000,
  "createdAt": "2026-07-04T10:00:00Z"
}
```

### Errors
| Code | HTTP | Description |
|------|------|-------------|
| INSUFFICIENT_BALANCE | 400 | Not enough balance |
| IDEMPOTENCY_CONFLICT | 409 | Duplicate idempotency key |
| WALLET_NOT_FOUND | 404 | Wallet does not exist |

---

## Refund Credit

### Endpoint
```
POST /api/v1/credit/refund
```

### Purpose
Refund previously consumed credits.

### Request
**Body**:
```json
{
  "tenantId": "tenant-12345",
  "originalTransactionId": "txn-abc-123",
  "amount": 1000,
  "reason": "SERVICE_FAILURE",
  "referenceId": "refund-12345"
}
```

### Response
**Success (200 OK)**:
```json
{
  "transactionId": "txn-refund-456",
  "status": "COMPLETED",
  "refundAmount": 1000,
  "newBalance": 100000,
  "createdAt": "2026-07-04T10:00:00Z"
}
```

---

## Adjust Balance (Admin)

### Endpoint
```
POST /api/v1/credit/adjust
```

### Purpose
Manually adjust wallet balance (admin only).

### Request
**Body**:
```json
{
  "tenantId": "tenant-12345",
  "amount": 50000,
  "reason": "PROMOTION_GRANT",
  "adjustedBy": "admin@wion.vn"
}
```

### Response
**Success (200 OK)**:
```json
{
  "transactionId": "txn-adjust-789",
  "status": "COMPLETED",
  "adjustmentAmount": 50000,
  "newBalance": 150000,
  "createdAt": "2026-07-04T10:00:00Z"
}
```

---

## Payment Processing - API

## Initiate Topup

### Endpoint
```
POST /api/v1/payments/topup
```

### Purpose
Initiate QR code payment for wallet topup.

### Request
**Body**:
```json
{
  "amount": 100000,
  "currency": "VND",
  "paymentMethod": "QR",
  "returnUrl": "https://app.wion.vn/billing/return"
}
```

### Response
**Success (200 OK)**:
```json
{
  "paymentId": "pay-12345",
  "status": "PENDING",
  "amount": 100000,
  "qrCode": "https://qr.tpaygate.vn/xyz",
  "expiresAt": "2026-07-04T10:15:00Z"
}
```

---

## Get Payment Status

### Endpoint
```
GET /api/v1/payments/{paymentId}
```

### Response
**Success (200 OK)**:
```json
{
  "paymentId": "pay-12345",
  "tenantId": "tenant-12345",
  "status": "COMPLETED",
  "amount": 100000,
  "paymentMethod": "QR",
  "createdAt": "2026-07-04T10:00:00Z",
  "completedAt": "2026-07-04T10:05:00Z"
}
```

---

## Invoice Generation - API

## Get Invoice by Payment

### Endpoint
```
GET /api/v1/invoices/payment/{paymentId}
```

### Purpose
Retrieve invoice for a specific payment.

### Response
**Success (200 OK)**:
```json
{
  "id": "inv-12345",
  "paymentId": "pay-12345",
  "invoiceNumber": "INV20260704001",
  "invoiceUrl": "https://invoice.wion.vn/INV20260704001.pdf",
  "status": "ISSUED",
  "amount": 100000,
  "taxAmount": 10000,
  "totalAmount": 110000,
  "issuedAt": "2026-07-04T10:06:00Z"
}
```

---

## Get Invoice History

### Endpoint
```
GET /api/v1/invoices?tenantId={tenantId}&from={from}&to={to}&page={page}&size={size}
```

### Response
**Success (200 OK)**:
```json
{
  "invoices": [
    {
      "invoiceNumber": "INV20260704001",
      "amount": 100000,
      "status": "ISSUED",
      "issuedAt": "2026-07-04T10:06:00Z"
    }
  ],
  "pagination": {
    "page": 0,
    "size": 20,
    "total": 50,
    "totalPages": 3
  }
}
```

---

## Webhook Handling - API

## Payment Webhook

### Endpoint
```
POST /api/v1/webhooks/payment
```

### Purpose
Receive payment callbacks from TPayGate.

### Request
**Headers**:
- `X-Webhook-Signature: {signature}`
- `X-Gateway: TPayGate`

**Body**:
```json
{
  "gatewayTransactionId": "TPG-12345",
  "eventType": "PAYMENT_SUCCESS",
  "paymentData": {
    "amount": 100000,
    "currency": "VND",
    "status": "COMPLETED",
    "transactionTime": "2026-07-04T10:05:00Z"
  },
  "timestamp": "2026-07-04T10:05:05Z"
}
```

### Response
**Success (200 OK)**:
```json
{
  "status": "PROCESSED",
  "webhookEventId": "wh-12345",
  "paymentId": "pay-12345"
}
```

**Error (403 Forbidden)**:
```json
{
  "status": "REJECTED",
  "error": "INVALID_SIGNATURE"
}
```

---

## Related Documentation
- [Feature Index](../feature-index.md)
- [API Index](../../api-index.md)
