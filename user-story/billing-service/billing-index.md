# Billing Platform - User Stories Index

## Overview

Billing Platform là hệ thống thanh toán thống nhất cho toàn bộ hệ sinh thái WION, bao gồm:
- WION POS
- WION FnB
- WION SPA
- WIPIX
- AI Services
- Platform Services

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                      Billing Platform                         │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │   Wallet     │  │   Payment    │  │    Credit    │      │
│  │   Service    │  │   Service    │  │   Service    │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│         │                 │                 │                │
│         └─────────────────┼─────────────────┘                │
│                           │                                  │
│  ┌──────────────┐  ┌──────┴──────┐  ┌──────────────┐       │
│  │   Ledger     │  │   Webhook   │  │   Invoice    │       │
│  │   Service    │  │   Handler   │  │ Integration  │       │
│  └──────────────┘  └─────────────┘  └──────────────┘       │
│                                                               │
├─────────────────────────────────────────────────────────────┤
│                      Client SDK                               │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │   WION POS   │  │  WION FnB    │  │  AI Services │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

---

## User Stories

### 1. [Wallet Service](./wallet-service.md) 🔥 Foundation

**Status:** Draft
**Priority:** Critical (P0)
**Dependencies:** None

**Description:**
Quản lý ví điện tử nội bộ và số dư cho từng tenant.

**Key Features:**
- Wallet creation & management
- Balance tracking (available, reserved)
- Multi-asset support (Wi Credit, Promotion, Gift, Trial)
- Balance check API

**Links:** [View Details](./wallet-service.md)

---

### 2. [Ledger Service](./ledger-service.md) 📊 Audit

**Status:** Draft
**Priority:** Critical (P0)
**Dependencies:** None

**Description:**
Hệ thống sổ cái ghi nhận mọi giao dịch với double-entry accounting.

**Key Features:**
- Double-entry transaction logging
- Transaction history & audit
- Idempotency support
- Transaction search & filter

**Links:** [View Details](./ledger-service.md)

---

### 3. [Payment Service](./payment-service.md) 💳 Integration

**Status:** Draft
**Priority:** High (P1)
**Dependencies:** wallet-service.md

**Description:**
Tích hợp với TPayGate để xử lý nạp tiền qua QR code.

**Key Features:**
- Topup initiation
- QR payment flow
- Payment tracking
- Payment history

**Links:** [View Details](./payment-service.md)

---

### 4. [Credit Service](./credit-service.md) ⚡ Operations

**Status:** Draft
**Priority:** High (P1)
**Dependencies:** wallet-service.md, ledger-service.md

**Description:**
Quản lý việc consume, refund và adjustment credit.

**Key Features:**
- Consume credit API
- Refund credit API
- Balance adjustment API
- Consumption tracking

**Links:** [View Details](./credit-service.md)

---

### 5. [Webhook Handler](./webhook-handler.md) 🔔 Callbacks

**Status:** Draft
**Priority:** High (P1)
**Dependencies:** payment-service.md

**Description:**
Xử lý webhook callbacks từ TPayGate.

**Key Features:**
- Webhook signature verification
- Idempotent webhook processing
- Auto credit wallet on success
- Trigger invoice creation

**Links:** [View Details](./webhook-handler.md)

---

### 6. [Invoice Integration](./invoice-integration.md) 📄 Invoicing

**Status:** Draft
**Priority:** Medium (P2)
**Dependencies:** payment-service.md

**Description:**
Tích hợp với Invoice Hub để xuất hóa đơn điện tử.

**Key Features:**
- Invoice creation API
- Payment → Invoice mapping
- Retry mechanism
- Invoice queue management

**Links:** [View Details](./invoice-integration.md)

---

### 7. [Client SDK](./client-sdk.md) 🛠️ Developer Tools

**Status:** Draft
**Priority:** High (P1)
**Dependencies:** wallet-service.md, credit-service.md

**Description:**
Client SDK để các application tích hợp với Billing Platform.

**Key Features:**
- Simple API interface
- Built-in retry logic
- Error handling
- Logging & monitoring
- TypeScript support

**Links:** [View Details](./client-sdk.md)

---

## Dependency Graph

```
Phase 1 (Foundation):
┌─────────────────┐
│  Wallet Service │ ◄─── Foundation (no dependencies)
└─────────────────┘
┌─────────────────┐
│  Ledger Service │ ◄─── Foundation (no dependencies)
└─────────────────┘

Phase 2 (Core):
┌─────────────────┐
│ Payment Service │ ◄─── Depends on: Wallet
└─────────────────┘
┌─────────────────┐
│  Credit Service │ ◄─── Depends on: Wallet, Ledger
└─────────────────┘
┌─────────────────────┐
│ Webhook Handler     │ ◄─── Depends on: Payment
└─────────────────────┘

Phase 3 (Integration):
┌──────────────────────┐
│ Invoice Integration  │ ◄─── Depends on: Payment
└──────────────────────┘
┌─────────────────┐
│  Client SDK     │ ◄─── Depends on: Wallet, Credit
└─────────────────┘
```

---

## Implementation Order

### Sprint 1 (Week 1-2)
1. ✅ **Wallet Service** - Foundation
2. ✅ **Ledger Service** - Audit foundation

### Sprint 2 (Week 3-4)
3. ✅ **Payment Service** - Topup functionality
4. ✅ **Credit Service** - Core consumption

### Sprint 3 (Week 5-6)
5. ✅ **Webhook Handler** - Payment callbacks
6. ✅ **Invoice Integration** - Invoicing

### Sprint 4 (Week 7-8)
7. ✅ **Client SDK** - Developer tools
8. ✅ **Integration Testing** - End-to-end tests

---

## Cross-Reference Matrix

| Service | Depends On | Used By |
|---------|------------|---------|
| Wallet Service | None | Payment, Credit, SDK |
| Ledger Service | None | Payment, Credit, Webhook, Invoice |
| Payment Service | Wallet | Webhook, Invoice |
| Credit Service | Wallet, Ledger | SDK |
| Webhook Handler | Payment | - |
| Invoice Integration | Payment, Ledger | - |
| Client SDK | Wallet, Credit | All Applications |

---

## Event Flow Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                     Payment Success Flow                      │
└─────────────────────────────────────────────────────────────┘

User scan QR
    ↓
TPayGate Process
    ↓
TPayGate Webhook ────────→ Webhook Handler
                               ↓
                          Verify Signature
                               ↓
                          Check Duplicate
                               ↓
                    ┌──────────┴──────────┐
                    ↓                     ↓
            Update Payment          Credit Wallet
            Status                   (Balance +)
                    ↓                     ↓
                    └──────────┬──────────┘
                               ↓
                        Create Ledger Entry
                               ↓
                         Publish Events:
                         - PaymentSucceeded
                         - BalanceChanged
                               ↓
                    Trigger Invoice Creation
                               ↓
                    Invoice Hub (Issue Invoice)
                               ↓
                         Store Invoice Ref
                               ↓
                    Publish InvoiceIssued
```

---

## API Endpoints Summary

### Wallet APIs
- `GET /api/v1/wallets/{tenantId}` - Get wallet info
- `GET /api/v1/wallets/{tenantId}/balance` - Get balance
- `POST /api/v1/wallets` - Create wallet (internal)

### Payment APIs
- `POST /api/v1/payments/topup` - Initiate topup
- `GET /api/v1/payments/{paymentId}` - Get payment status
- `GET /api/v1/payments` - Get payment history
- `POST /api/v1/payments/webhook` - Payment webhook

### Credit APIs
- `POST /api/v1/credit/consume` - Consume credit
- `POST /api/v1/credit/refund` - Refund credit
- `POST /api/v1/credit/adjust` - Adjust balance
- `GET /api/v1/credit/consumption` - Get consumption history

### Ledger APIs
- `GET /api/v1/ledger/transactions` - Get transaction history
- `GET /api/v1/ledger/transactions/{transactionId}` - Get transaction detail
- `POST /api/v1/ledger/transactions` - Create transaction (internal)

### Invoice APIs
- `POST /api/v1/invoices/create` - Create invoice
- `GET /api/v1/invoices/payment/{paymentId}` - Get invoice by payment
- `GET /api/v1/invoices/{invoiceNumber}` - Get invoice by number
- `GET /api/v1/invoices` - Get invoice history

---

## Events Summary

### Published Events
- `WalletCreated` - Wallet được tạo
- `BalanceChanged` - Balance thay đổi
- `PaymentCreated` - Payment được tạo
- `PaymentSucceeded` - Payment thành công
- `PaymentFailed` - Payment thất bại
- `PaymentExpired` - Payment timeout
- `CreditConsumed` - Credit được consume
- `CreditRefunded` - Credit được refund
- `BalanceAdjusted` - Balance được điều chỉnh
- `InsufficientBalance` - Balance không đủ
- `TransactionCreated` - Transaction được tạo
- `TransactionCompleted` - Transaction hoàn thành
- `TransactionFailed` - Transaction thất bại
- `InvoiceIssued` - Invoice được issue
- `InvoiceFailed` - Invoice creation thất bại
- `WebhookReceived` - Webhook được nhận
- `WebhookProcessed` - Webhook được xử lý
- `WebhookFailed` - Webhook xử lý thất bại

### Subscribed Events
- `TenantDeleted` - Xóa wallet khi tenant bị xóa

---

## Non-Functional Requirements

### Performance
- Wallet API: < 100ms (P95)
- Credit API: < 100ms (P95)
- Payment API: < 500ms (P95)
- Webhook processing: < 1s (P95)
- Support 10,000 concurrent users

### Availability
- All APIs: 99.9% SLA
- Webhook endpoint: 99.9% SLA
- Graceful degradation

### Security
- API key authentication
- Webhook signature verification
- Tenant isolation
- Encryption in transit (HTTPS)
- Audit logging

### Scalability
- Horizontal scaling ready
- Database sharding strategy
- Caching layer (Redis)
- Message queue (RabbitMQ/Kafka)

---

## Business Rules Summary

| Rule ID | Description | Service |
|---------|-------------|---------|
| BR-001 | Balance không được âm | Wallet |
| BR-002 | Mọi Consume phải tạo Ledger | Credit |
| BR-003 | Invoice chỉ tạo khi Payment thành công | Invoice |
| BR-004 | Webhook phải Idempotent | Webhook |
| BR-005 | ReferenceId phải duy nhất | Credit |
| BR-006 | Webhook signature phải valid | Webhook |
| BR-007 | Invoice data phải complete | Invoice |
| BR-008 | Invoice creation phải retry | Invoice |

---

## Use Cases Summary

| UC ID | Use Case | Primary Service |
|-------|----------|-----------------|
| UC-01 | Topup Credit | Payment |
| UC-02 | Consume AI Credit | Credit |
| UC-03 | Refund | Credit |
| UC-04 | Issue Invoice | Invoice |
| UC-05 | Check Balance | Wallet |
| UC-06 | Transaction History | Ledger |

---

## Glossary

| Term | Definition |
|------|------------|
| **Wi Credit** | Đơn vị thanh toán nội bộ (1 VNĐ = 1 Wi Credit) |
| **Wallet** | Ví điện tử nội bộ của tenant |
| **Ledger** | Sổ cái ghi nhận mọi giao dịch |
| **Transaction** | Giao dịch thay đổi số dư |
| **Idempotency** | Tính chất một request được thực hiện nhiều lần nhưng chỉ có tác dụng một lần |
| **ReferenceId** | ID của giao dịch từ external system |
| **Tenant** | Khách hàng sử dụng Platform |
| **TPayGate** | Payment Gateway partner |
| **Invoice Hub** | Hệ thống xuất hóa đơn điện tử |

---

## Version History

| Version | Date | Changes | Author |
|---------|------|---------|--------|
| 1.0 | 2026-06-28 | Initial split from billing-service.md | WION Platform Team |

---

## Next Steps

1. **Review User Stories**: Team review each user story
2. **Estimate Effort**: Story point estimation for each story
3. **Sprint Planning**: Assign stories to sprints
4. **Start Development**: Begin with Wallet & Ledger (Phase 1)

---

## Contacts

- **Platform Team**: platform@wion.vn
- **Tech Lead**: tech-lead@wion.vn
- **Product Owner**: po@wion.vn

---

*Last updated: 2026-06-28*
