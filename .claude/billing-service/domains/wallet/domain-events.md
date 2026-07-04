# Wallet Domain Events

This document describes all domain events published by the Wallet Aggregate.

## Event Overview

Wallet Aggregate publishes events to notify other parts of the system about state changes. These events enable eventual consistency across bounded contexts.

---

## WalletCreated

**Purpose**: Notify that a new wallet has been created for a tenant

**Published By**: Wallet Aggregate on creation

**Payload**:
```json
{
  "eventType": "WalletCreated",
  "tenantId": "tenant-12345",
  "walletId": "wallet-12345",
  "currency": "VND",
  "createdAt": "2026-07-04T10:00:00Z"
}
```

**Trigger**: Tenant registration triggers wallet creation

**Consumers**:
- Ledger Service - Create initial ledger entry
- Analytics Service - Track wallet creation
- Monitoring Service - Log new wallet

**Guarantee**: At-least-once delivery

---

## BalanceChanged

**Purpose**: Notify that wallet balance has changed

**Published By**: Wallet Aggregate on credit/debit operations

**Payload**:
```json
{
  "eventType": "BalanceChanged",
  "tenantId": "tenant-12345",
  "assetType": "WI_CREDIT",
  "oldBalance": 100000,
  "newBalance": 95000,
  "changeAmount": -5000,
  "changeType": "DEBIT",
  "referenceId": "txn-12345",
  "timestamp": "2026-07-04T10:05:00Z"
}
```

**Change Types**:
- `CREDIT` - Balance increased
- `DEBIT` - Balance decreased
- `RESERVE` - Funds transferred to reserved
- `RELEASE` - Reserved funds released back

**Trigger**:
- `credit(amount)` operation
- `debit(amount)` operation
- `reserve(amount)` operation
- `releaseReserve(amount)` operation

**Consumers**:
- Ledger Service - Create ledger entries
- Credit Service - Update balance tracking
- Analytics Service - Track balance changes
- Notification Service - Notify user of significant changes

**Guarantee**: At-least-once delivery

---

## InsufficientBalance

**Purpose**: Notify that a balance operation failed due to insufficient funds

**Published By**: Wallet Aggregate on failed debit validation

**Payload**:
```json
{
  "eventType": "InsufficientBalance",
  "tenantId": "tenant-12345",
  "requestedAmount": 50000,
  "availableBalance": 30000,
  "shortage": 20000,
  "referenceId": "txn-12345",
  "timestamp": "2026-07-04T10:05:00Z"
}
```

**Trigger**:
- Debit operation when `availableBalance < requestedAmount`
- Reserve operation when `availableBalance < requestedAmount`

**Consumers**:
- Monitoring Service - Alert on insufficient balance
- Notification Service - Notify user to top up
- Analytics Service - Track failed transactions

**Guarantee**: At-least-once delivery

---

## Event Delivery

### Reliability
- **Guarantee**: At-least-once delivery
- **Retry**: 3 attempts with exponential backoff (5s, 30s, 60s)
- **Dead Letter Queue**: After 3 failed attempts

### Ordering
- **Per Tenant**: Events for the same tenant are ordered
- **Causal Order**: Maintains cause-effect relationship

### Persistence
- **Retention**: 7 days in message queue
- **Storage**: Event log in database for audit

### Performance
- **Publish Latency**: < 50ms (P95)
- **Throughput**: 10,000 events/second

---

## Event Schema

All Wallet events follow this schema:

```json
{
  "eventId": "unique-event-id",
  "eventType": "WalletCreated | BalanceChanged | InsufficientBalance",
  "tenantId": "tenant-identifier",
  "timestamp": "ISO-8601-timestamp",
  "correlationId": "correlation-id",
  "metadata": {
    "additional properties"
  }
}
```

---

## Event Handlers

### Subscribing to Wallet Events

To subscribe to Wallet events:

```typescript
// Example event handler
class WalletEventHandler {
  async handleBalanceChanged(event: BalanceChangedEvent) {
    // 1. Validate event
    // 2. Process event
    // 3. Acknowledge message
  }

  async handleInsufficientBalance(event: InsufficientBalanceEvent) {
    // 1. Alert monitoring
    // 2. Notify user
    // 3. Acknowledge message
  }
}
```

### Idempotency

Event handlers MUST be idempotent:
- Use `eventId` to detect duplicate events
- Store processed event IDs
- Return success for duplicate events

---

## Related Documents
- [Wallet Aggregate](./aggregate.md) - Aggregate definition
- [Wallet Lifecycle](./lifecycle.md) - State transitions that trigger events
- [Wallet Business Rules](./business-rules.md) - Rules that may trigger InsufficientBalance event
