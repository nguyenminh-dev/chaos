# Webhook Handler Service

## Metadata
- **Version:** 1.0
- **Status:** Draft
- **Author:** WION Platform Team
- **Dependencies:** payment-service.md

---

## 1. Business Background

### Context
Webhook Handler Service xử lý webhook callbacks từ Payment Gateway (TPayGate).

TPayGate gửi webhook để báo cáo:
- Payment thành công
- Payment thất bại
- Payment timeout

Webhook Handler phải:
- Xác thực webhook signature
- Xử lý idempotently (TPayGate có thể gửi nhiều lần)
- Trigger credit wallet
- Trigger invoice creation

---

## 2. Business Concepts

### Webhook Flow
TPayGate → Billing Webhook Handler → Payment Service → Wallet → Invoice Hub

### Idempotency
TPayGate có thể gửi webhook nhiều lần vì:
- Network retry
- TPayGate internal retry
- No acknowledgment

Handler phải:
- Detect duplicate webhook
- Process chỉ một lần
- Return 200 OK cho duplicate

### Signature Verification
Mọi webhook phải verify signature để:
- Prevent fake webhook
- Ensure integrity
- Verify authencity

---

## 3. Functional Requirements

### FR-03: Webhook Handling

**Actor:** TPayGate (external)

**Precondition:**
- Payment đã được tạo

**Main Flow:**
1. TPayGate gửi webhook đến Billing
2. Webhook Handler verify signature:
   - Extract signature from header
   - Compute expected signature
   - Compare
3. If invalid: return 403
4. If valid: check idempotency
5. If duplicate: return 200 (already processed)
6. If new: process webhook
7. Based on payment status:
   - SUCCESS: Credit wallet, trigger invoice
   - FAILED: Update payment status
   - TIMEOUT: Update payment status
8. Create ledger entry
9. Return 200 OK

**Postcondition:**
- Payment status được update
- Wallet được credit (nếu success)
- Invoice được trigger (nếu success)
- Ledger được tạo

---

## 4. Business Rules

### BR-003: Invoice Timing
Invoice chỉ được tạo khi Payment thành công.

Sequence:
1. Payment SUCCESS → Credit wallet → Create Invoice
2. Payment FAILED/TIMEOUT → No credit, no invoice

### BR-004: Webhook Idempotency
Webhook phải idempotent.

TPayGate có thể gửi webhook nhiều lần:
- System phải detect duplicate bằng (gatewayTransactionId + event)
- Process chỉ một lần
- Return 200 OK cho duplicate

### BR-006: Signature Verification
Webhook signature phải valid.

Nếu signature invalid:
- Reject webhook
- Return 403 Forbidden
- Log security event

---

## 5. Domain Model

### WebhookEvent
```
WebhookEvent {
  id: string (PK)
  gatewayTransactionId: string (indexed)
  eventType: enum (PAYMENT_SUCCESS, PAYMENT_FAILED, PAYMENT_TIMEOUT)
  payload: json
  signature: string
  status: enum (RECEIVED, PROCESSED, FAILED, DUPLICATE)
  processingAttempts: integer
  receivedAt: datetime
  processedAt: datetime
  errorMessage: string (nullable)
}
```

### WebhookLog
```
WebhookLog {
  id: string (PK)
  webhookEventId: string (FK)
  eventType: string
  action: string
  result: enum (SUCCESS, FAILED, SKIPPED)
  duration: integer (ms)
  createdAt: datetime
}
```

---

## 6. API Specification

### Payment Webhook Endpoint
```
POST /api/v1/webhooks/payment

Headers:
- X-Webhook-Signature: {signature}
- X-Gateway: {TPayGate}
- Content-Type: application/json

Request:
{
  "gatewayTransactionId": "string",
  "eventType": "PAYMENT_SUCCESS",
  "paymentData": {
    "amount": decimal,
    "currency": "VND",
    "paymentMethod": "QR",
    "status": "COMPLETED",
    "transactionTime": "datetime"
  },
  "timestamp": "datetime",
  "signature": "string"
}

Response (200 OK):
{
  "status": "PROCESSED",
  "webhookEventId": "string",
  "paymentId": "string",
  "message": "Webhook processed successfully"
}

Response (403 Forbidden):
{
  "status": "REJECTED",
  "error": "INVALID_SIGNATURE",
  "message": "Webhook signature verification failed"
}

Response (409 Conflict - Duplicate):
{
  "status": "DUPLICATE",
  "webhookEventId": "string",
  "message": "Webhook already processed"
}
```

### Webhook Retry Queue (Internal)
```
POST /api/v1/webhooks/retry

Request:
{
  "webhookEventId": "string"
}

Response:
{
  "status": "QUEUED_FOR_RETRY",
  "retryAt": "datetime"
}
```

### Get Webhook Status
```
GET /api/v1/webhooks/{webhookEventId}

Response:
{
  "id": "string",
  "gatewayTransactionId": "string",
  "eventType": "PAYMENT_SUCCESS",
  "status": "PROCESSED",
  "processingAttempts": 1,
  "receivedAt": "datetime",
  "processedAt": "datetime"
}
```

---

## 7. Integration with TPayGate

### Webhook Signature
TPayGate sẽ gửi signature trong header: `X-Webhook-Signature`

Signature algorithm:
```
signature = HMAC-SHA256(secret_key, payload_json)
```

Verification:
```python
expected_signature = hmac_sha256(WEBHOOK_SECRET, request_body)
if expected_signature == received_signature:
    # Valid
else:
    # Invalid, return 403
```

### Retry Strategy
TPayGate retry policy:
- Retry 3 times
- Exponential backoff: 1s, 2s, 4s
- Nếu vẫn fail: manual investigation

### Payload Format
```json
{
  "gatewayTransactionId": "TPG_1234567890",
  "eventType": "PAYMENT_SUCCESS",
  "paymentData": {
    "amount": 100000,
    "currency": "VND",
    "paymentMethod": "QR",
    "status": "COMPLETED",
    "transactionTime": "2026-06-28T10:30:00Z"
  },
  "timestamp": "2026-06-28T10:30:05Z",
  "signature": "abc123..."
}
```

---

## 8. Event Processing Flow

### Payment Success Flow
```
Webhook RECEIVED
↓
Verify Signature
↓
Check Duplicate (gatewayTransactionId + eventType)
↓
If DUPLICATE → Return 200 (already processed)
↓
If NEW:
  ↓
  Update Payment status to COMPLETED
  ↓
  Call Wallet Service to credit balance
  ↓
  Create Ledger Entry
  ↓
  Trigger Invoice Creation
  ↓
  Publish PaymentSucceeded Event
  ↓
  Return 200 OK
```

### Payment Failed Flow
```
Webhook RECEIVED
↓
Verify Signature
↓
Check Duplicate
↓
If NEW:
  ↓
  Update Payment status to FAILED
  ↓
  Create Ledger Entry (for record)
  ↓
  Publish PaymentFailed Event
  ↓
  Return 200 OK
```

### Payment Timeout Flow
```
Webhook RECEIVED
↓
Verify Signature
↓
Check Duplicate
↓
If NEW:
  ↓
  Update Payment status to TIMEOUT
  ↓
  Create Ledger Entry (for record)
  ↓
  Publish PaymentExpired Event
  ↓
  Return 200 OK
```

---

## 9. Events

### Publish
- **WebhookReceived** - Khi webhook được nhận
- **WebhookProcessed** - Khi webhook được xử lý thành công
- **WebhookFailed** - Khi webhook xử lý thất bại
- **DuplicateWebhookDetected** - Khi phát hiện webhook trùng

### Consume
- None

---

## 10. Permission Matrix

| Actor | Receive Webhook | View Status | Retry |
|-------|----------------|-------------|-------|
| Platform Admin | N/A | ✓ | ✓ |
| Finance | N/A | ✓ | N/A |
| Tenant Owner | N/A | N/A | N/A |
| TPayGate | ✓ | N/A | N/A |
| System | N/A | ✓ | ✓ (auto) |

---

## 11. Non-functional Requirements

### Security
- Signature verification for every webhook
- IP whitelist (TPayGate IPs only)
- Rate limiting per gateway

### Availability
- Webhook endpoint: 99.9% SLA
- Must handle burst of webhooks
- Async processing for heavy load

### Performance
- Webhook acknowledgment < 100ms (return 200 quickly)
- Processing can be async
- Support 1,000 webhooks per second

### Monitoring
- Alert on signature verification failures
- Alert on processing failures
- Track webhook processing time
- Track duplicate rate

---

## 12. Acceptance Criteria

### Definition of Done
- [ ] Webhook endpoint hoạt động
- [ ] Signature verification đúng
- [ ] Idempotency hoạt động
- [ ] Payment success → Wallet credited
- [ ] Payment success → Invoice triggered
- [ ] Payment fail → Status updated
- [ ] Ledger entry được tạo
- [ ] Events được publish
- [ ] API documentation hoàn chỉnh
- [ ] Unit test coverage > 80%
- [ ] Integration test với TPayGate

### Edge Cases
- [ ] Xử lý webhook signature invalid
- [ ] Xử lý webhook duplicate
- [ ] Xử lý payment not found
- [ ] Xử lý wallet credit fail (rollback)
- [ ] Xử lý invoice creation fail (compensate)
- [ ] Xử lý webhook payload invalid
- [ ] Xử lý concurrency (same webhook multiple times)
- [ ] Xử lý webhook retry queue

### Security Tests
- [ ] Test webhook without signature (403)
- [ ] Test webhook with wrong signature (403)
- [ ] Test webhook from untrusted IP (403)
- [ ] Test replay attack (duplicate detection)

---

## 13. Cross-References

- **Depends On:**
  - payment-service.md (handles payment webhooks)

- **Dependent Services:**
  - wallet-service.md (credit wallet on success)
  - ledger-service.md (log webhook processing)
  - invoice-integration.md (trigger invoice on success)

- **Related:**
  - BR-003: Invoice timing rule
  - BR-004: Webhook idempotency rule
