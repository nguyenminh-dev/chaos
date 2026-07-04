# Billing Service - Database

## Architecture Context

### Database as Infrastructure Detail

**Critical Principle**: Database is an Infrastructure concern, NOT a Domain concern.

**Business Rules MUST NOT be in**:
❌ Database stored procedures
❌ Database triggers
❌ Database constraints beyond data integrity
❌ Database functions containing business logic

**Business Rules MUST be in**:
✓ Domain Aggregates (enforce invariants)
✓ Domain Value Objects (validation)
✓ Domain Services (cross-aggregate rules)

**Database Role**:
- Persist Aggregates
- Enforce data integrity (NOT business logic)
- Provide retrieval capabilities
- Maintain audit trails

---

### Aggregate Persistence

**One Table per Aggregate Root**:
- `wallet` table → Wallet Aggregate
- `payment` table → Payment Aggregate
- `credit_transaction` table → CreditTransaction Aggregate
- `ledger` table → LedgerEntry Aggregate
- `invoice_reference` table → InvoiceReference Aggregate

**Value Objects**:
- Embedded in root entity table (JSON columns)
- Separate table only if lifecycle differs

**Optimistic Locking**:
- `version` column on all root entities
- Prevent concurrent modification conflicts

---

## Database Schema

## Database Schema

### wallet
**Purpose**: Store wallet information for each tenant

```sql
CREATE TABLE wallet (
  tenant_id VARCHAR(255) PRIMARY KEY,
  balance DECIMAL(19,4) NOT NULL DEFAULT 0,
  reserved_balance DECIMAL(19,4) NOT NULL DEFAULT 0,
  currency VARCHAR(3) NOT NULL DEFAULT 'VND',
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  version INT NOT NULL DEFAULT 0,
  is_deleted BOOLEAN NOT NULL DEFAULT FALSE,

  CONSTRAINT chk_balance_nonnegative CHECK (balance >= 0),
  CONSTRAINT chk_reserved_nonnegative CHECK (reserved_balance >= 0)
);

CREATE INDEX idx_created_at ON wallet(created_at);
CREATE INDEX idx_is_deleted ON wallet(is_deleted);
```

---

### wallet_asset
**Purpose**: Store multiple asset types per wallet

```sql
CREATE TABLE wallet_asset (
  id VARCHAR(255) PRIMARY KEY,
  tenant_id VARCHAR(255) NOT NULL,
  asset_type ENUM('WI_CREDIT', 'PROMOTION', 'GIFT', 'TRIAL', 'AI_TOKEN') NOT NULL,
  balance DECIMAL(19,4) NOT NULL DEFAULT 0,
  reserved_balance DECIMAL(19,4) NOT NULL DEFAULT 0,
  metadata JSON,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  expires_at TIMESTAMP NULL,
  is_deleted BOOLEAN NOT NULL DEFAULT FALSE,

  CONSTRAINT fk_wallet FOREIGN KEY (tenant_id) REFERENCES wallet(tenant_id) ON DELETE CASCADE,
  CONSTRAINT uq_tenant_asset UNIQUE (tenant_id, asset_type),
  CONSTRAINT chk_asset_balance_nonnegative CHECK (balance >= 0)
);

CREATE INDEX idx_tenant_asset ON wallet_asset(tenant_id, asset_type);
CREATE INDEX idx_expires_at ON wallet_asset(expires_at);
```

---

### payment
**Purpose**: Track payment transactions

```sql
CREATE TABLE payment (
  id VARCHAR(255) PRIMARY KEY,
  tenant_id VARCHAR(255) NOT NULL,
  amount DECIMAL(19,4) NOT NULL,
  currency VARCHAR(3) NOT NULL,
  status ENUM('PENDING', 'PROCESSING', 'COMPLETED', 'FAILED', 'CANCELLED') NOT NULL,
  payment_method ENUM('QR', 'BANK_TRANSFER', 'CREDIT_CARD', 'E_WALLET') NOT NULL,
  gateway_transaction_id VARCHAR(255),
  gateway_reference VARCHAR(255),
  metadata JSON,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  completed_at TIMESTAMP NULL,
  expires_at TIMESTAMP NULL
);

CREATE INDEX idx_tenant_id ON payment(tenant_id);
CREATE INDEX idx_status ON payment(status);
CREATE INDEX idx_gateway_transaction_id ON payment(gateway_transaction_id);
CREATE INDEX idx_created_at ON payment(created_at);
```

---

### credit_transaction
**Purpose**: Track credit consumption and refunds

```sql
CREATE TABLE credit_transaction (
  id VARCHAR(255) PRIMARY KEY,
  tenant_id VARCHAR(255) NOT NULL,
  type ENUM('CONSUME', 'REFUND', 'ADJUSTMENT') NOT NULL,
  status ENUM('PENDING', 'COMPLETED', 'FAILED', 'REVERSED') NOT NULL,
  amount DECIMAL(19,4) NOT NULL,
  currency VARCHAR(3) NOT NULL,
  service VARCHAR(100) NOT NULL,
  reference_id VARCHAR(255) NOT NULL,
  original_transaction_id VARCHAR(255),
  idempotency_key VARCHAR(255) UNIQUE,
  reason VARCHAR(500),
  metadata JSON,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  completed_at TIMESTAMP NULL,
  created_by VARCHAR(255)
);

CREATE INDEX idx_tenant_id ON credit_transaction(tenant_id);
CREATE INDEX idx_reference_id ON credit_transaction(reference_id);
CREATE INDEX idx_idempotency_key ON credit_transaction(idempotency_key);
CREATE INDEX idx_service ON credit_transaction(service);
CREATE INDEX idx_created_at ON credit_transaction(created_at);
```

---

### ledger
**Purpose**: Double-entry accounting records

```sql
CREATE TABLE ledger (
  id VARCHAR(255) PRIMARY KEY,
  tenant_id VARCHAR(255) NOT NULL,
  transaction_id VARCHAR(255) NOT NULL,
  transaction_type ENUM('PAYMENT', 'CONSUME', 'REFUND', 'ADJUSTMENT', 'TRANSFER') NOT NULL,
  debit_account VARCHAR(255) NOT NULL,
  credit_account VARCHAR(255) NOT NULL,
  amount DECIMAL(19,4) NOT NULL,
  currency VARCHAR(3) NOT NULL,
  reference_type ENUM('PAYMENT_GATEWAY', 'INVOICE', 'MANUAL') NOT NULL,
  reference_id VARCHAR(255) NOT NULL,
  metadata JSON,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  created_by VARCHAR(255) NOT NULL
);

CREATE INDEX idx_tenant_id ON ledger(tenant_id);
CREATE INDEX idx_transaction_id ON ledger(transaction_id);
CREATE INDEX idx_reference_id ON ledger(reference_id);
CREATE INDEX idx_created_at ON ledger(created_at);
```

---

### invoice_reference
**Purpose**: Map payments to invoices

```sql
CREATE TABLE invoice_reference (
  id VARCHAR(255) PRIMARY KEY,
  payment_id VARCHAR(255) NOT NULL,
  tenant_id VARCHAR(255) NOT NULL,
  invoice_number VARCHAR(255) NOT NULL,
  invoice_hub_id VARCHAR(255) NOT NULL,
  invoice_url TEXT,
  invoice_type ENUM('BAN_HANG', 'PHIEN_GIA_DICH', 'KHAC') NOT NULL,
  status ENUM('PENDING', 'ISSUED', 'FAILED', 'CANCELLED') NOT NULL,
  amount DECIMAL(19,4) NOT NULL,
  tax_amount DECIMAL(19,4) NOT NULL,
  total_amount DECIMAL(19,4) NOT NULL,
  issued_at TIMESTAMP NULL,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  metadata JSON
);

CREATE INDEX idx_payment_id ON invoice_reference(payment_id);
CREATE INDEX idx_tenant_id ON invoice_reference(tenant_id);
CREATE INDEX idx_invoice_number ON invoice_reference(invoice_number);
CREATE INDEX idx_status ON invoice_reference(status);
```

---

### webhook_event
**Purpose**: Log webhook processing

```sql
CREATE TABLE webhook_event (
  id VARCHAR(255) PRIMARY KEY,
  gateway_transaction_id VARCHAR(255) NOT NULL,
  event_type ENUM('PAYMENT_SUCCESS', 'PAYMENT_FAILED', 'PAYMENT_TIMEOUT') NOT NULL,
  payload JSON NOT NULL,
  signature VARCHAR(255) NOT NULL,
  status ENUM('RECEIVED', 'PROCESSED', 'FAILED', 'DUPLICATE') NOT NULL,
  processing_attempts INT NOT NULL DEFAULT 0,
  received_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  processed_at TIMESTAMP NULL,
  error_message TEXT
);

CREATE INDEX idx_gateway_transaction_id ON webhook_event(gateway_transaction_id);
CREATE INDEX idx_status ON webhook_event(status);
CREATE INDEX idx_received_at ON webhook_event(received_at);
```

---

### invoice_queue
**Purpose**: Queue for failed invoice creation

```sql
CREATE TABLE invoice_queue (
  id VARCHAR(255) PRIMARY KEY,
  payment_id VARCHAR(255) NOT NULL,
  priority INT NOT NULL DEFAULT 0,
  status ENUM('PENDING', 'PROCESSING', 'COMPLETED', 'FAILED') NOT NULL,
  retry_count INT NOT NULL DEFAULT 0,
  next_retry_at TIMESTAMP NULL,
  payload JSON NOT NULL,
  error_message TEXT,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_payment_id ON invoice_queue(payment_id);
CREATE INDEX idx_status ON invoice_queue(status);
CREATE INDEX idx_next_retry_at ON invoice_queue(next_retry_at);
```

---

## Key Operations

### Balance Update (Atomic with Optimistic Locking)
```sql
UPDATE wallet
SET balance = balance + ?,
    reserved_balance = reserved_balance + ?,
    version = version + 1,
    updated_at = CURRENT_TIMESTAMP
WHERE tenant_id = ?
  AND version = ?
  AND balance + ? >= 0;
```

**Returns**: Number of rows affected (0 if version mismatch)

---

### Create Ledger Entry (Double-Entry)
```sql
INSERT INTO ledger (id, tenant_id, transaction_id, transaction_type,
                   debit_account, credit_account, amount, currency,
                   reference_type, reference_id, metadata, created_at, created_by)
VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, CURRENT_TIMESTAMP, ?);
```

**Constraint**: Must be paired with corresponding credit/debit entries

---

### Payment Creation
```sql
INSERT INTO payment (id, tenant_id, amount, currency, status, payment_method,
                    gateway_transaction_id, created_at, expires_at)
VALUES (?, ?, ?, ?, 'PENDING', ?, ?, CURRENT_TIMESTAMP, ?);
```

---

## Database Constraints

### Data Integrity Constraints (Infrastructure Layer)
These constraints protect DATA INTEGRITY, not business rules:

**Check Constraints**:
- `balance >= 0` - Prevent data corruption (negative balance)
- `reserved_balance >= 0` - Prevent data corruption
- `amount > 0` - Prevent invalid amounts

**Unique Constraints**:
- `tenant_id` (wallet) - One wallet per tenant (data integrity)
- `(tenant_id, asset_type)` (wallet_asset) - One asset per type
- `idempotency_key` (credit_transaction) - Unique operation keys

**Foreign Keys**:
- `wallet_asset.tenant_id` → `wallet.tenant_id` (CASCADE delete)
- All payment/invoice references

---

### Business Invariants (Domain Layer)
These rules are enforced in Domain Aggregates, NOT database:

**Wallet Aggregate Invariants**:
- `balance >= 0` - Enforced in `Wallet.debit()` method
- `reserved_balance >= 0` - Enforced in `Wallet.reserve()` method
- `balance + reserved_balance >= total` - Enforced in Wallet

**Payment Aggregate Invariants**:
- `status` transitions are valid - Enforced in `Payment.transitionTo()`
- Completed payments cannot be modified - Enforced in Payment methods

**Ledger Aggregate Invariants**:
- `Σ debit = Σ credit` - Enforced in `LedgerEntry.validateDoubleEntry()`
- Entries are immutable - No update methods on LedgerEntry

**Key Point**: Database constraints are DEFENSE in depth, not PRIMARY business rule enforcement.

---

## Performance Optimization

### Index Strategy
- Primary keys on all tables
- Foreign keys indexed
- Composite indexes for common query patterns
- Covering indexes for frequent queries

### Connection Pooling
- **Pool Size**: 20 connections
- **Timeout**: 30 seconds
- **Idle Timeout**: 10 minutes

### Query Optimization
- Use prepared statements
- Enable query cache
- Read replicas for balance queries
- Write to master for mutations

---

## Data Retention

### Retention Policy
- **Active wallets**: Indefinite
- **Ledger data**: 7 years (legal compliance)
- **Payment records**: 7 years
- **Invoice references**: 10 years
- **Webhook logs**: 1 year
- **Soft deleted data**: 7 years

### Cleanup Strategy
- Daily job to mark expired assets
- Weekly job to archive old webhook logs
- Monthly job to archive soft deleted records

---

## Backup & Recovery

### Backup Strategy
- **Full Backup**: Daily at 2:00 AM UTC
- **Incremental Backup**: Every 4 hours
- **Retention**: 30 days

### Recovery
- Point-in-time recovery enabled
- RTO: 1 hour
- RPO: 5 minutes

---

## Related Documentation
- [Service Documentation](./service.md)
- [Dependency Documentation](./dependency.md)
