# Ledger Service

## Metadata
- **Version:** 1.0
- **Status:** Draft
- **Author:** WION Platform Team
- **Dependencies:** None (can be built in parallel with wallet)

---

## 1. Business Background

### Context
Ledger Service là nguồn dữ liệu chuẩn phục vụ:
- Audit: Truy vết mọi giao dịch
- Đối soát: So khớp số liệu với external systems
- Truy vết: Debug và dispute resolution

Ledger sử dụng Double-entry Accounting:
- Mọi giao dịch có Debit và Credit
- Tổng Debit = Tổng Credit
- Không bao giờ có balance âm

---

## 2. Business Concepts

### Transaction
Mọi thay đổi số dư đều phải sinh Transaction.

Không tồn tại việc cập nhật Balance trực tiếp.

### Ledger Entry
Mỗi Ledger Entry thể hiện:
- Transaction ID
- Account (Debit hoặc Credit)
- Amount
- Reference (external transaction ID)
- Metadata

### Double-Entry
Mọi giao dịch phải balance:
- Σ Debit = Σ Credit
- Nếu không balance: reject transaction

---

## 3. Functional Requirements

### FR-07: Lịch sử giao dịch

**Actor:** Tenant Owner, Finance, Platform Admin

**Precondition:**
- Wallet đã có giao dịch

**Flow:**
1. User gọi API get transaction history
2. System trả về:
   - List transactions
   - Pagination
   - Filter by date range, type, status

**Postcondition:**
- Transaction history hoàn chỉnh và chính xác

---

## 4. Business Rules

### BR-002: Ledger Requirement
Mọi Consume phải tạo Ledger.

Không exception:
- Credit consumption → Ledger entry
- Credit refund → Ledger entry
- Balance adjustment → Ledger entry
- Payment → Ledger entry

### BR-004: Idempotency
Transaction creation phải idempotent.

Để tránh duplicate:
- Client phải cung cấp unique idempotency key
- System cache result của request với key
- Retry với cùng key = return cached result

---

## 5. Domain Model

### Ledger
```
Ledger {
  id: string (PK)
  tenantId: string (indexed)
  transactionId: string (indexed)
  transactionType: enum (PAYMENT, CONSUME, REFUND, ADJUSTMENT, TRANSFER)
  debitAccount: string
  creditAccount: string
  amount: decimal
  currency: string
  referenceType: enum (PAYMENT_GATEWAY, INVOICE, MANUAL)
  referenceId: string (indexed)
  metadata: json
  createdAt: datetime
  createdBy: string
}
```

### Transaction
```
Transaction {
  id: string (PK)
  tenantId: string (indexed)
  type: enum (PAYMENT, CONSUME, REFUND, ADJUSTMENT, TRANSFER)
  status: enum (PENDING, COMPLETED, FAILED, REVERSED)
  amount: decimal
  currency: string
  idempotencyKey: string (indexed, unique)
  metadata: json
  createdAt: datetime
  completedAt: datetime
  entries: Ledger[]
}
```

---

## 6. API Specification

### Get Transaction History
```
GET /api/v1/ledger/transactions?tenantId={tenantId}&from={from}&to={to}&page={page}&size={size}

Query Parameters:
- tenantId (required)
- from (optional): ISO date
- to (optional): ISO date
- type (optional): Filter by transaction type
- status (optional): Filter by status
- page (default: 0)
- size (default: 20, max: 100)

Response:
{
  "transactions": [
    {
      "id": "string",
      "type": "CONSUME",
      "status": "COMPLETED",
      "amount": decimal,
      "currency": "VND",
      "createdAt": "datetime",
      "metadata": {}
    }
  ],
  "pagination": {
    "page": 0,
    "size": 20,
    "total": 150,
    "totalPages": 8
  }
}
```

### Get Transaction Detail
```
GET /api/v1/ledger/transactions/{transactionId}

Response:
{
  "id": "string",
  "tenantId": "string",
  "type": "CONSUME",
  "status": "COMPLETED",
  "amount": decimal,
  "currency": "VND",
  "metadata": {},
  "entries": [
    {
      "id": "string",
      "debitAccount": "WALLET:{tenantId}",
      "creditAccount": "CONSUME:AI",
      "amount": decimal,
      "referenceType": "MANUAL",
      "referenceId": "string"
    }
  ],
  "createdAt": "datetime",
  "completedAt": "datetime"
}
```

### Create Transaction (Internal)
```
POST /api/v1/ledger/transactions

Headers:
- X-Idempotency-Key: {unique-key}

Request:
{
  "tenantId": "string",
  "type": "CONSUME",
  "entries": [
    {
      "debitAccount": "WALLET:{tenantId}",
      "creditAccount": "CONSUME:AI",
      "amount": decimal,
      "referenceType": "MANUAL",
      "referenceId": "string"
    }
  ],
  "metadata": {}
}

Response:
{
  "id": "string",
  "status": "COMPLETED",
  "amount": decimal,
  "entries": [...],
  "createdAt": "datetime"
}
```

---

## 7. Events

### Publish
- **TransactionCreated** - Khi transaction được tạo
- **TransactionCompleted** - Khi transaction hoàn thành
- **TransactionFailed** - Khi transaction thất bại

---

## 8. Permission Matrix

| Actor | View History | View Detail | Create Transaction |
|-------|-------------|-------------|-------------------|
| Platform Admin | ✓ (all) | ✓ (all) | ✓ (internal) |
| Finance | ✓ (all) | ✓ (all) | ✓ (adjustment) |
| Tenant Owner | ✓ (own) | ✓ (own) | N/A |
| Application | N/A | N/A | ✓ (internal) |
| SDK | N/A | N/A | ✓ (internal) |

---

## 9. Non-functional Requirements

### Audit
- Mọi transaction phải log
- Không cho phép delete
- Soft delete only (for legal compliance)

### Performance
- Query by tenant + date range < 200ms (P95)
- Support 5,000 write TPS
- Retention: 7 years (legal requirement)

### Consistency
- ACID for transaction creation
- Either all entries created or none

---

## 10. Acceptance Criteria

### Definition of Done
- [ ] Transaction API hoạt động đúng
- [ ] Double-entry balancing được enforce
- [ ] Idempotency hoạt động đúng
- [ ] History pagination hoạt động
- [ ] Filter by date range/type hoạt động
- [ ] Audit log hoàn chỉnh
- [ ] API documentation hoàn chỉnh
- [ ] Unit test coverage > 80%
- [ ] Integration test với dependent services

### Edge Cases
- [ ] Xử lý debit != credit
- [ ] Xử lý duplicate idempotency key
- [ ] Xử lý transaction revert
- [ ] Xử lý concurrent transaction creation
- [ ] Xử lý massive historical data query

---

## 11. Cross-References

- **Dependent On:** None
- **Dependent Services:**
  - wallet-service.md (log wallet balance changes)
  - credit-service.md (log credit operations)
  - payment-service.md (log payment transactions)
  - invoice-integration.md (log invoice mappings)

- **Related Use Cases:**
  - UC-06: Transaction History
