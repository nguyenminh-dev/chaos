# Wallet Service

## Metadata
- **Version:** 1.0
- **Status:** Draft
- **Author:** WION Platform Team
- **Dependencies:** None (foundation service)

---

## 1. Business Background

### Context
Wallet Service là nền tảng cơ bản của Billing Platform, quản lý số dư và tài sản cho toàn bộ hệ sinh thái WION.

Wallet Service được tích hợp bởi:
- WION POS
- WION FnB
- WION SPA
- WIPIX
- AI Services
- Platform Services

---

## 2. Business Concepts

### Wallet
Một Tenant có một ví.

Wallet lưu:
- Balance (Số dư khả dụng)
- Reserved Balance (Số dư đang khóa)
- Currency (Loại tiền tệ)

### Asset
Wallet có thể chứa nhiều loại tài sản.

Ví dụ:
- Wi Credit
- Promotion Credit
- Gift Credit
- Trial Credit
- AI Token (Future)

### Wi Credit
Wi Credit là đơn vị thanh toán nội bộ của WION Platform.

Nguyên tắc:
- 1 VNĐ = 1 Wi Credit (có thể cấu hình)
- Không quy đổi ngược
- Không chuyển ra tiền mặt
- Có thể tiêu thụ tại mọi sản phẩm WION

---

## 3. Functional Requirements

### FR-09: Kiểm tra số dư

**Actor:** Tenant Owner, Application, SDK

**Precondition:**
- Wallet đã tồn tại

**Flow:**
1. Application/SDK gọi API check balance
2. System trả về:
   - Available Balance
   - Reserved Balance
   - Total Balance
   - Asset breakdown

**Postcondition:**
- Balance được trả về chính xác

---

## 4. Business Rules

### BR-001: Balance Validation
Balance không được âm.

Mọi giao dịch trừ credit phải kiểm tra:
- Available Balance >= Amount
- Nếu không đủ: từ chối giao dịch

### BR-005: ReferenceId Uniqueness
ReferenceId phải duy nhất theo Source.

Để tránh trùng lặp giao dịch:
- ReferenceId + Source phải unique
- System phải reject nếu trùng lặp

---

## 5. Domain Model

### Wallet
```
Wallet {
  tenantId: string (PK)
  balance: decimal
  reservedBalance: decimal
  currency: string
  createdAt: datetime
  updatedAt: datetime
  version: integer
}
```

### WalletAsset
```
WalletAsset {
  id: string (PK)
  tenantId: string (FK)
  assetType: enum (WI_CREDIT, PROMOTION, GIFT, TRIAL, AI_TOKEN)
  balance: decimal
  reservedBalance: decimal
  metadata: json
  createdAt: datetime
  updatedAt: datetime
  expiresAt: datetime (nullable)
}
```

---

## 6. API Specification

### Get Wallet Balance
```
GET /api/v1/wallets/{tenantId}/balance

Response:
{
  "tenantId": "string",
  "availableBalance": decimal,
  "reservedBalance": decimal,
  "totalBalance": decimal,
  "currency": "VND",
  "assets": [
    {
      "assetType": "WI_CREDIT",
      "balance": decimal,
      "reservedBalance": decimal
    }
  ]
}
```

### Get Wallet by Tenant
```
GET /api/v1/wallets/{tenantId}

Response:
{
  "tenantId": "string",
  "balance": decimal,
  "reservedBalance": decimal,
  "currency": "string",
  "createdAt": "datetime",
  "updatedAt": "datetime"
}
```

### Create Wallet (Internal)
```
POST /api/v1/wallets

Request:
{
  "tenantId": "string",
  "currency": "VND"
}

Response:
{
  "tenantId": "string",
  "balance": 0,
  "reservedBalance": 0,
  "currency": "VND",
  "createdAt": "datetime"
}
```

---

## 7. Events

### Publish
- **WalletCreated** - Khi wallet được tạo mới
- **BalanceChanged** - Khi balance thay đổi

### Consume
- **TenantDeleted** - Xóa wallet khi tenant bị xóa

---

## 8. Permission Matrix

| Actor | View Balance | Create Wallet | Update Balance |
|-------|-------------|---------------|----------------|
| Platform Admin | ✓ | ✓ | ✓ (internal) |
| Tenant Owner | ✓ (own) | N/A | N/A |
| Finance | ✓ (all) | N/A | N/A |
| Application | ✓ (authorized) | N/A | N/A |
| SDK | ✓ (authorized) | N/A | N/A |

---

## 9. Non-functional Requirements

### Consistency
- Balance update phải atomic
- Sử dụng optimistic locking (version field)

### Performance
- API response time < 100ms (P95)
- Support 10,000 RPS

### Security
- Tenant chỉ được xem wallet của chính mình
- Finance/Platform Admin có thể xem all

---

## 10. Acceptance Criteria

### Definition of Done
- [ ] Wallet được tạo khi tenant đăng ký
- [ ] Get balance API trả về đúng số dư
- [ ] Balance không bao giờ âm
- [ ] Support multi asset type
- [ ] Audit log cho tất cả operations
- [ ] API documentation hoàn chỉnh
- [ ] Unit test coverage > 80%
- [ ] Integration test với dependent services

### Edge Cases
- [ ] Xử lý khi wallet không tồn tại
- [ ] Xử lý concurrent balance updates
- [ ] Xử lý asset expire
- [ ] Xử lý tenant deletion

---

## 11. Cross-References

- **Dependent Services:**
  - payment-service.md (credit wallet after topup)
  - credit-service.md (consume/check balance)
  - ledger-service.md (log balance changes)

- **Related Use Cases:**
  - UC-05: Check Balance
