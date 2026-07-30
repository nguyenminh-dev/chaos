# Payment Service

## Metadata
- **Version:** 1.0
- **Status:** Draft
- **Author:** WION Platform Team
- **Dependencies:** wallet-service.md (needs wallet to credit)

---

## 1. Business Background

### Context
Payment Service là cầu nối giữa WION Platform và Payment Gateway (TPayGate).

Payment Service KHÔNG thay thế Payment Gateway.
Payment Service chỉ:
- Tích hợp với TPayGate
- Quản lý payment flow
- Credit wallet sau khi payment thành công
- Webhook handling

---

## 2. Business Concepts

### Payment Flow
Portal → Billing → TPayGate → QR → Webhook → Billing → Wallet → Invoice Hub

### Payment Methods
- QR Code (Primary)
- Bank Transfer (Future)
- Credit Card (Future)
- E-wallet (Future)

---

## 3. Functional Requirements

### FR-01: Nạp Credit (Topup)

**Actor:** Tenant Owner

**Precondition:**
- User đã login
- Wallet đã tồn tại

**Main Flow:**
1. User nhập amount muốn topup
2. System validate amount (min, max)
3. System tạo Payment Request với TPayGate
4. TPayGate trả về QR Code
5. System hiển thị QR cho user
6. User scan QR và thanh toán
7. TPayGate gửi webhook về Billing
8. System credit wallet
9. System trigger invoice creation

**Alternative Flow:**
- Payment thất bại → User thử lại
- Payment timeout → System mark as failed

**Postcondition:**
- Wallet balance tăng
- Ledger được tạo
- Invoice được issue

---

### FR-02: Thanh toán QR

**Actor:** Tenant Owner

**Precondition:**
- Payment Request đã được tạo

**Flow:**
1. System hiển thị QR Code từ TPayGate
2. User scan QR với banking app
3. User confirm payment
4. TPayGate process payment
5. TPayGate send webhook

**Postcondition:**
- Payment được ghi nhận
- Webhook được xử lý

---

## 4. Business Rules

### BR-003: Invoice Timing
Invoice chỉ được tạo khi Payment thành công.

Sequence:
1. Payment thành công → Credit wallet → Create Invoice
2. Nếu payment thất bại → Không credit, không invoice

### BR-004: Webhook Idempotency
Webhook phải idempotent.

TPayGate có thể gửi webhook nhiều lần:
- System phải detect duplicate webhook
- Process chỉ một lần
- Return 200 OK cho duplicate

---

## 5. Domain Model

### Payment
```
Payment {
  id: string (PK)
  tenantId: string (indexed)
  amount: decimal
  currency: string
  status: enum (PENDING, PROCESSING, COMPLETED, FAILED, CANCELLED)
  paymentMethod: enum (QR, BANK_TRANSFER, CREDIT_CARD, E_WALLET)
  gatewayTransactionId: string (indexed)
  gatewayReference: string
  metadata: json
  createdAt: datetime
  completedAt: datetime
  expiresAt: datetime
}
```

### PaymentMethod
```
PaymentMethod {
  id: string (PK)
  tenantId: string
  type: enum (QR, BANK_TRANSFER, CREDIT_CARD, E_WALLET)
  provider: string
  metadata: json
  isActive: boolean
  createdAt: datetime
}
```

### PaymentTransaction
```
PaymentTransaction {
  id: string (PK)
  paymentId: string (FK)
  type: enum (REQUEST, WEBHOOK, CREDIT, INVOICE)
  status: enum (PENDING, SUCCESS, FAILED)
  request: json
  response: json
  createdAt: datetime
  processedAt: datetime
}
```

---

## 6. API Specification

### Initiate Topup
```
POST /api/v1/payments/topup

Request:
{
  "amount": decimal,
  "currency": "VND",
  "paymentMethod": "QR",
  "returnUrl": "string",
  "metadata": {}
}

Response:
{
  "paymentId": "string",
  "status": "PENDING",
  "amount": decimal,
  "qrCode": "string",
  "expiresAt": "datetime"
}
```

### Get Payment Status
```
GET /api/v1/payments/{paymentId}

Response:
{
  "paymentId": "string",
  "tenantId": "string",
  "status": "COMPLETED",
  "amount": decimal,
  "paymentMethod": "QR",
  "createdAt": "datetime",
  "completedAt": "datetime"
}
```

### Get Payment History
```
GET /api/v1/payments?tenantId={tenantId}&from={from}&to={to}&status={status}&page={page}&size={size}

Response:
{
  "payments": [...],
  "pagination": {...}
}
```

### Payment Webhook
```
POST /api/v1/payments/webhook

Headers:
- X-Webhook-Signature: {signature}

Request:
{
  "gatewayTransactionId": "string",
  "status": "COMPLETED",
  "amount": decimal,
  "currency": "VND",
  "paymentMethod": "QR",
  "timestamp": "datetime",
  "signature": "string"
}

Response:
{
  "status": "PROCESSED",
  "paymentId": "string"
}
```

---

## 7. Integration with TPayGate

### Create Payment Request
```
POST {TPayGate}/api/v1/payments

Request:
{
  "amount": decimal,
  "currency": "VND",
  "method": "QR",
  "webhookUrl": "{Billing}/api/v1/payments/webhook",
  "merchantId": "string",
  "reference": "string"
}

Response:
{
  "transactionId": "string",
  "qrCode": "string",
  "expiresAt": "datetime"
}
```

---

## 8. User Journey

### Topup Flow Detail

#### Step 1: Portal
```
User click "Nạp tiền"
↓
Portal redirect to Billing
```

#### Step 2: Billing
```
Billing show topup form
↓
User enter amount
↓
Billing call Payment API
```

#### Step 3: TPayGate
```
Billing request TPayGate
↓
TPayGate return QR
↓
Billing display QR to user
```

#### Step 4: QR Payment
```
User scan QR
↓
User confirm payment
↓
TPayGate process
```

#### Step 5: Webhook
```
TPayGate send webhook
↓
Billing verify signature
↓
Billing update payment status
```

#### Step 6: Wallet
```
Billing credit wallet
↓
Ledger record transaction
```

#### Step 7: Invoice Hub
```
Billing trigger invoice
↓
Invoice Hub issue invoice
```

---

## 9. Events

### Publish
- **PaymentCreated** - Khi payment được tạo
- **PaymentSucceeded** - Khi payment thành công
- **PaymentFailed** - Khi payment thất bại
- **PaymentExpired** - Khi payment timeout

### Consume
- None

---

## 10. Permission Matrix

| Actor | Initiate Payment | View History | Webhook |
|-------|-----------------|--------------|----------|
| Platform Admin | ✓ | ✓ (all) | N/A |
| Finance | N/A | ✓ (all) | N/A |
| Tenant Owner | ✓ | ✓ (own) | N/A |
| Application | N/A | N/A | N/A |
| SDK | N/A | N/A | N/A |
| TPayGate | N/A | N/A | ✓ |

---

## 11. Non-functional Requirements

### Security
- Webhook signature verification
- Idempotency key for all requests
- Amount encryption in transit

### Performance
- Topup API response < 500ms (P95)
- Webhook processing < 1s (P95)
- Support 1,000 concurrent topups

### Availability
- Webhook endpoint: 99.9% SLA
- Retry webhook 3 times with exponential backoff

---

## 12. Acceptance Criteria

### Definition of Done
- [ ] Topup API hoạt động đúng
- [ ] QR code hiển thị đúng
- [ ] Webhook handling idempotent
- [ ] Wallet được credit sau payment thành công
- [ ] Ledger record payment transaction
- [ ] Invoice được trigger
- [ ] Payment history API hoạt động
- [ ] API documentation hoàn chỉnh
- [ ] Unit test coverage > 80%
- [ ] Integration test với TPayGate

### Edge Cases
- [ ] Xử lý payment timeout
- [ ] Xử lý webhook duplicate
- [ ] Xử lý webhook signature invalid
- [ ] Xử lý amount không match
- [ ] Xử lý payment thất bại sau khi đã credit (rollback)

---

## 13. Cross-References

- **Depends On:**
  - wallet-service.md (credit wallet)

- **Dependent Services:**
  - webhook-handler.md (handles payment webhooks)
  - invoice-integration.md (trigger invoice creation)
  - ledger-service.md (log payment transactions)

- **Related Use Cases:**
  - UC-01: Topup Credit
