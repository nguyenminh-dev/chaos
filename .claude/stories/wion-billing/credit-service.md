# Credit Service

## Metadata
- **Version:** 1.0
- **Status:** Draft
- **Author:** WION Platform Team
- **Dependencies:** wallet-service.md, ledger-service.md

---

## 1. Business Background

### Context
Credit Service quản lý việc tiêu consume, refund và adjustment credit cho toàn bộ Platform Services.

Các sản phẩm sử dụng Credit Service:
- WION AI (AI Generation, OCR)
- WION Communication (SMS)
- WION Storage
- Marketplace
- Add-ons

---

## 2. Business Concepts

### Transaction Concept
Mọi thay đổi số dư đều phải sinh Transaction.

Không tồn tại việc cập nhật Balance trực tiếp.

### Wi Credit
Wi Credit là đơn vị thanh toán nội bộ của WION Platform.

Nguyên tắc:
- 1 VNĐ = 1 Wi Credit (có thể cấu hình)
- Không quy đổi ngược
- Không chuyển ra tiền mặt
- Có thể tiêu thụ tại mọi sản phẩm WION

### Transaction Types
- **Consume**: Trừ credit khi sử dụng service
- **Refund**: Hoàn credit khi refund/thất bại
- **Adjustment**: Điều chỉnh bởi admin/finance

---

## 3. Functional Requirements

### FR-05: Trừ Credit (Consume Credit)

**Actor:** Application, SDK

**Precondition:**
- Wallet có đủ balance
- Service được authorized

**Main Flow:**
1. Application/SDK gọi consume API
2. System validate:
   - Tenant exists
   - Sufficient balance
   - Idempotency key
3. System reserve amount
4. System process service request
5. If success:
   - Confirm transaction
   - Deduct balance
   - Create ledger entry
6. If failure:
   - Release reserve
   - Refund amount

**Postcondition:**
- Balance được cập nhật
- Ledger được tạo
- Audit log được ghi

**API:**
```
POST /api/v1/credit/consume

Request:
{
  "tenantId": "string",
  "amount": decimal,
  "service": "AI_GENERATION",
  "referenceId": "string",
  "idempotencyKey": "string",
  "metadata": {}
}
```

---

### FR-06: Hoàn Credit (Refund Credit)

**Actor:** Application, SDK, Finance

**Precondition:**
- Original transaction exists
- Transaction chưa được refund

**Main Flow:**
1. System validate original transaction
2. System check refund eligibility:
   - Within refund period
   - Not already refunded
   - Sufficient refundable amount
3. System refund credit:
   - Add balance back
   - Create refund transaction
   - Link to original transaction
4. System create ledger entry

**Postcondition:**
- Balance được hoàn
- Refund transaction được tạo
- Ledger được cập nhật

**API:**
```
POST /api/v1/credit/refund

Request:
{
  "tenantId": "string",
  "originalTransactionId": "string",
  "amount": decimal,
  "reason": "string",
  "referenceId": "string",
  "metadata": {}
}
```

---

### FR-04: Cộng Credit (Adjustment)

**Actor:** Platform Admin, Finance

**Precondition:**
- Actor có quyền adjustment

**Main Flow:**
1. Admin/Finance tạo adjustment request
2. System validate:
   - Actor permission
   - Adjustment reason required
   - Amount valid
3. System update balance:
   - Add or deduct amount
   - Create adjustment transaction
   - Require approval if amount > threshold

**Postcondition:**
- Balance được điều chỉnh
- Adjustment transaction được tạo
- Audit log ghi người thực hiện

**API:**
```
POST /api/v1/credit/adjust

Request:
{
  "tenantId": "string",
  "amount": decimal,
  "reason": "string",
  "adjustedBy": "string",
  "metadata": {}
}
```

---

## 4. Business Rules

### BR-001: Balance Validation
Balance không được âm.

Mọi giao dịch trừ credit phải kiểm tra:
- Available Balance >= Amount
- Nếu không đủ: reject

### BR-002: Ledger Requirement
Mọi Consume phải tạo Ledger.

Không exception:
- Credit consumption → Ledger entry
- Credit refund → Ledger entry
- Balance adjustment → Ledger entry

### BR-005: ReferenceId Uniqueness
ReferenceId phải duy nhất theo Source.

Để tránh trùng lặp:
- ReferenceId + Service phải unique
- System phải reject nếu trùng lặp

---

## 5. Domain Model

### CreditTransaction
```
CreditTransaction {
  id: string (PK)
  tenantId: string (indexed)
  type: enum (CONSUME, REFUND, ADJUSTMENT, TRANSFER)
  status: enum (PENDING, COMPLETED, FAILED, REVERSED)
  amount: decimal
  currency: string
  service: string
  referenceId: string (indexed)
  originalTransactionId: string (nullable, for refund)
  idempotencyKey: string (indexed, unique)
  reason: string (for adjustment/refund)
  metadata: json
  createdAt: datetime
  completedAt: datetime
  createdBy: string (for adjustment)
}
```

### ConsumptionRecord
```
ConsumptionRecord {
  id: string (PK)
  tenantId: string (indexed)
  service: string
  resourceType: string
  quantity: decimal
  unit: string
  unitPrice: decimal
  amount: decimal
  periodStart: datetime
  periodEnd: datetime
  metadata: json
  createdAt: datetime
}
```

---

## 6. API Specification

### Consume Credit
```
POST /api/v1/credit/consume

Request:
{
  "tenantId": "string",
  "amount": decimal,
  "service": "AI_GENERATION",
  "referenceId": "string",
  "idempotencyKey": "string",
  "metadata": {
    "resourceType": "TOKEN",
    "quantity": 1000
  }
}

Response:
{
  "transactionId": "string",
  "status": "COMPLETED",
  "amount": decimal,
  "remainingBalance": decimal,
  "createdAt": "datetime"
}
```

### Refund Credit
```
POST /api/v1/credit/refund

Request:
{
  "tenantId": "string",
  "originalTransactionId": "string",
  "amount": decimal,
  "reason": "SERVICE_FAILURE",
  "referenceId": "string",
  "metadata": {}
}

Response:
{
  "transactionId": "string",
  "status": "COMPLETED",
  "refundAmount": decimal,
  "newBalance": decimal,
  "createdAt": "datetime"
}
```

### Adjust Balance
```
POST /api/v1/credit/adjust

Request:
{
  "tenantId": "string",
  "amount": decimal,
  "reason": "PROMOTION_GRANT",
  "adjustedBy": "admin@wion.vn",
  "metadata": {
    "campaign": "WELCOME_BONUS"
  }
}

Response:
{
  "transactionId": "string",
  "status": "COMPLETED",
  "adjustmentAmount": decimal,
  "newBalance": decimal,
  "createdAt": "datetime"
}
```

### Get Consumption History
```
GET /api/v1/credit/consumption?tenantId={tenantId}&service={service}&from={from}&to={to}&page={page}&size={size}

Response:
{
  "consumptions": [
    {
      "id": "string",
      "service": "AI_GENERATION",
      "amount": decimal,
      "createdAt": "datetime"
    }
  ],
  "pagination": {...}
}
```

---

## 7. Events

### Publish
- **CreditConsumed** - Khi credit được consume
- **CreditRefunded** - Khi credit được refund
- **BalanceAdjusted** - Khi balance được adjustment
- **InsufficientBalance** - Khi balance không đủ

### Consume
- None

---

## 8. Permission Matrix

| Actor | Consume | Refund | Adjust | View History |
|-------|---------|--------|--------|--------------|
| Platform Admin | N/A | ✓ | ✓ | ✓ (all) |
| Finance | N/A | ✓ | ✓ | ✓ (all) |
| Tenant Owner | N/A | N/A | N/A | ✓ (own) |
| Application | ✓ | ✓ (own service) | N/A | N/A |
| SDK | ✓ | ✓ (own service) | N/A | N/A |

---

## 9. Non-functional Requirements

### Performance
- Consume API: < 100ms (P95)
- Refund API: < 200ms (P95)
- Support 10,000 consume TPS

### Consistency
- Atomic balance update
- Ledger creation must succeed
- If ledger fails → rollback balance

### Security
- Rate limiting per tenant
- Service-level authorization
- Adjustment requires approval

---

## 10. Acceptance Criteria

### Definition of Done
- [ ] Consume API hoạt động đúng
- [ ] Refund API hoạt động đúng
- [ ] Adjust API hoạt động đúng
- [ ] Balance không bao giờ âm
- [ ] Ledger entry được tạo cho mọi transaction
- [ ] Idempotency hoạt động
- [ ] API documentation hoàn chỉnh
- [ ] Unit test coverage > 80%
- [ ] Integration test với dependent services

### Edge Cases
- [ ] Xử lý insufficient balance
- [ ] Xử lý refund exceed original amount
- [ ] Xử lý duplicate idempotency key
- [ ] Xử lý concurrent consume
- [ ] Xử lý refund transaction not found
- [ ] Xử lý adjustment approval threshold

---

## 11. Cross-References

- **Depends On:**
  - wallet-service.md (check and update balance)
  - ledger-service.md (create ledger entries)

- **Dependent Services:**
  - client-sdk.md (SDK wraps consume API)

- **Related Use Cases:**
  - UC-02: Consume AI Credit
  - UC-03: Refund
