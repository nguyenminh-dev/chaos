# T-PayGate Domain Aggregates

## Overview
T-PayGate integration consists of three core aggregates that manage the payment lifecycle from bank connection through payment completion.

## Aggregates

### 1. BankConnection Aggregate

**Purpose**: Manage bank connection lifecycle and virtual account allocation.

**Aggregate Root**: `BankConnection`

**Entities**: None (simple aggregate)

**Value Objects**:
- `ConfigBankId` - Unique connection identifier
- `BankCode` - Bank identifier (NAPAS standard)
- `VaNumber` - Virtual Account number for payments
- `MerchantCredentials` - Encrypted bank credentials

#### BankConnection Lifecycle

```
INITIATED ──(OTP required)──> PENDING_OTP ──(OTP verified)──> CONNECTED
    │                                                               │
    │                           (No OTP required)                  │
    └──────────────────────────────────────────────────────────────┘
                                                                    │
                                                                    └───(Disconnect)──> DISCONNECTED
```

#### Business Invariants
1. **One connection per merchant-bank pair** per environment
2. **Virtual Account uniqueness** per connection
3. **Soft delete on disconnect** - preserve records for reconciliation
4. **OTP required** for most banks (bank-specific)

#### Domain Events
- `BankConnectionInitiated` - Connection started
- `BankOtpRequired` - OTP verification needed
- `BankConnectionConnected` - Connection established with VA
- `BankConnectionDisconnected` - Connection terminated

#### Repository
- `ITpgBankConnectionRepository`
  - `FindByConfigIdAsync(ConfigBankId)`
  - `FindByTenantIdAndBankCodeAsync(tenantId, bankCode)`
  - `ListActiveConnectionsAsync(tenantId)`

---

### 2. Bill Aggregate

**Purpose**: Manage payment bill creation, QR generation, and payment status tracking.

**Aggregate Root**: `Bill`

**Entities**: None (simple aggregate)

**Value Objects**:
- `BillCode` - T-PayGate bill identifier
- `RefTransactionId` - Merchant's transaction reference
- `QrData` - QR code content and image
- `Amount` - Payment amount in VND

#### Bill Lifecycle

```
CREATED ──(Customer scans QR)──> WAITING_PAYMENT ──(Payment received)──> PAID
    │                                      │
    │                                      └───(Timeout)──> EXPIRED
    │
    └───(Merchant cancels)───────────────────────────────────────> CANCELED
```

#### Business Invariants
1. **refTransactionId uniqueness** per merchant (24h cache)
2. **Amount immutability** - once created, amount cannot change
3. **Expiry enforcement** - default 24 hours, configurable per tenant
4. **Final state protection** - PAID/EXPIRED/CANNOT be modified

#### Domain Events
- `BillCreated` - Bill generated with QR code
- `BillScanned` - Customer scanned QR (WAITING_PAYMENT)
- `BillPaid` - Payment completed
- `BillExpired` - Payment deadline passed
- `BillCanceled` - Merchant cancelled bill

#### Repository
- `ITpgBillRepository`
  - `FindByBillCodeAsync(BillCode)`
  - `FindByRefTransactionIdAsync(RefTransactionId)`
  - `FindActiveBillsAsync(tenantId)`
  - `FindExpiringBillsAsync(DateTimeOffset)`

---

### 3. PaymentNotification Aggregate

**Purpose**: Manage webhook processing, idempotency, and payment notification tracking.

**Aggregate Root**: `PaymentNotification`

**Entities**: None (simple aggregate)

**Value Objects**:
- `NotificationId` - Unique notification identifier
- `BillCode` - Associated bill identifier
- `PaymentTime` - Bank payment timestamp
- `ActualAccount` - Customer's bank account

#### Webhook Processing Lifecycle

```
RECEIVED ──(Idempotency check)──> PROCESSING ──(Success)──> PROCESSED
    │                                         │
    │                                         └───(Failure)──> FAILED
    │
    └───(Duplicate)─────────────────────────────────────────> DUPLICATE
```

#### Business Invariants
1. **Idempotency by billCode** - prevent duplicate payment processing
2. **Amount validation** - webhook amount must match bill amount
3. **Processing timeout** - webhook must respond < 10s (PROD)
4. **Retry tracking** - log all webhook attempts

#### Domain Events
- `PaymentNotificationReceived` - Webhook received from T-PayGate
- `PaymentNotificationProcessed` - Payment notification processed successfully
- `DuplicatePaymentNotificationDetected` - Duplicate webhook detected
- `PaymentNotificationFailed` - Webhook processing failed

#### Repository
- `ITpgPaymentNotificationRepository`
  - `FindByBillCodeAsync(BillCode)`
  - `IsProcessedAsync(BillCode)` - Idempotency check
  - `LogAttemptAsync(NotificationId, attemptNumber)`

---

## Aggregate Interactions

### Bank Connection → Bill Creation

```
BankConnection (CONNECTED)
    ↓ provides configBankId and vaNumber
Bill.CREATED
    ↓ generates QR code
Customer scans QR → Bill.WAITING_PAYMENT
```

### Bill → Payment Notification

```
Bill.WAITING_PAYMENT
    ↓ customer pays via bank
PaymentNotification.RECEIVED
    ↓ idempotency check
PaymentNotification.PROCESSED
    ↓ updates bill status
Bill.PAID
```

### Cross-Aggregate Consistency

**Consistency Boundary**: Each aggregate maintains its own consistency.

**Eventual Consistency**: 
- Payment processing updates Bill status via domain events
- Bank connection state changes don't affect existing bills
- Webhook processing is independent of Bill state (idempotent)

---

## Data Persistence

### Schema Design

#### bank_connections
```sql
CREATE TABLE bank_connections (
    config_bank_id VARCHAR(36) PRIMARY KEY,
    tenant_id VARCHAR(50) NOT NULL,
    bank_code VARCHAR(10) NOT NULL,
    account_no VARCHAR(50) NOT NULL,
    account_name VARCHAR(200) NOT NULL,
    merchant_name VARCHAR(200) NOT NULL,
    va_number VARCHAR(50) NOT NULL UNIQUE,
    is_connected BOOLEAN DEFAULT TRUE,
    phone VARCHAR(20),
    email VARCHAR(100),
    date_created DATETIME2 NOT NULL,
    date_connected DATETIME2,
    date_disconnected DATETIME2,
    INDEX idx_tenant_bank (tenant_id, bank_code),
    INDEX idx_tenant_active (tenant_id, is_connected)
);
```

#### tpg_bills
```sql
CREATE TABLE tpg_bills (
    bill_code VARCHAR(50) PRIMARY KEY,
    ref_transaction_id VARCHAR(100) NOT NULL,
    tenant_id VARCHAR(50) NOT NULL,
    config_bank_id VARCHAR(36) NOT NULL,
    amount DECIMAL(18,0) NOT NULL,
    description VARCHAR(500),
    qr_content TEXT NOT NULL,
    qr_image_base64 TEXT,
    status VARCHAR(20) NOT NULL, -- CREATED, WAITING_PAYMENT, PAID, EXPIRED, CANCELED
    created_at DATETIME2 NOT NULL,
    expired_at DATETIME2 NOT NULL,
    paid_at DATETIME2,
    INDEX idx_ref_transaction (ref_transaction_id),
    INDEX idx_tenant_status (tenant_id, status),
    INDEX idx_expiry (expired_at),
    FOREIGN KEY (config_bank_id) REFERENCES bank_connections(config_bank_id)
);
```

#### payment_notifications
```sql
CREATE TABLE payment_notifications (
    id BIGINT PRIMARY KEY IDENTITY(1,1),
    bill_code VARCHAR(50) NOT NULL,
    amount DECIMAL(18,0) NOT NULL,
    virtual_account VARCHAR(50),
    actual_account VARCHAR(50),
    payment_time DATETIME2 NOT NULL,
    received_at DATETIME2 NOT NULL,
    processed_at DATETIME2,
    processing_status VARCHAR(20) NOT NULL, -- RECEIVED, PROCESSING, PROCESSED, FAILED, DUPLICATE
    attempt_number INT DEFAULT 1,
    INDEX idx_bill_code (bill_code),
    INDEX idx_status (processing_status),
    INDEX idx_received (received_at),
    UNIQUE INDEX idx_idempotency (bill_code)
);
```

---

## Repository Implementation Notes

### Connection Pooling
- Separate connection pool for T-PayGate API calls
- Configure timeout: 10 seconds for bill operations, 15 seconds for connect
- Implement circuit breaker per aggregate

### Transaction Boundaries
- **BankConnection**: Single connection per transaction
- **Bill**: Create + query in single transaction
- **PaymentNotification**: Webhook receive (immediate) + processing (background) separate

### Caching Strategy
- **BankConnection**: Cache connected configs (TTL: 1 hour)
- **Bill**: No caching (real-time status)
- **PaymentNotification**: Cache processed notifications (TTL: 24 hours)

---

## Related Documents
- [T-PayGate Domain Overview](./overview.md) - Domain context
- [T-PayGate Business Rules](./business-rules.md) - Business rules and invariants
- [T-PayGate Domain Model](./model.md) - Detailed class design
- [T-PayGate Domain Events](./domain-events.md) - Event definitions
