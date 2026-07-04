# Event Conventions

**Purpose**: Standard event conventions and patterns used across all WION microservices.

**Scope**: These conventions apply to **all services** in the WION ecosystem. Do not duplicate these in service-specific documentation.

---

## Event Naming Conventions

### Domain Events

**Format**: `{Aggregate}{StateChange}`

**Examples**:
- ✅ `WalletCreated` - New wallet created
- ✅ `BalanceChanged` - Balance updated
- ✅ `PaymentSucceeded` - Payment completed successfully
- ✅ `PaymentFailed` - Payment failed
- ✅ `InsufficientBalance` - Balance insufficient for operation

**Rules**:
- ✅ Use **past tense** (Created, Succeeded, Failed)
- ✅ Use **positive naming** (Succeeded, not DidNotFail)
- ✅ Be **specific** (PaymentSucceeded, not PaymentUpdated)
- ❌ Avoid generic names (StatusChanged, DataUpdated)

---

### Integration Events

**Format**: `{Service}{Entity}{Action}`

**Examples**:
- ✅ `BillingPaymentCompleted` - Payment completed in Billing Service
- ✅ `BillingInvoiceIssued` - Invoice issued in Billing Service
- ✅ `TenantUserCreated` - User created in Tenant Service

**Rules**:
- ✅ Include **service name** prefix
- ✅ Use **past tense**
- ✅ Be **specific** about what happened

---

## Event Payload Structure

### Standard Event Structure

```typescript
interface DomainEvent {
  eventType: string;        // Event name (e.g., "WalletCreated")
  eventId: string;          // Unique event ID
  aggregateId: string;       // Aggregate root ID
  aggregateType: string;    // Aggregate type (e.g., "Wallet")
  occurredAt: Date;         // When event happened
  version: number;          // Event version for evolution
  data: EventData;          // Event-specific data
  metadata?: EventMetadata; // Optional metadata
}

interface EventMetadata {
  correlationId?: string;   // For distributed tracing
  causationId?: string;    // Event that caused this event
  tenantId?: string;        // Tenant context
  userId?: string;          // User context
}
```

---

### Event Data Principles

**Rule 1**: Events contain **minimal data**

**Example**:
```typescript
✅ CORRECT:
interface PaymentSucceeded {
  eventType: "PaymentSucceeded";
  aggregateId: string;      // paymentId
  occurredAt: Date;
  data: {
    amount: number;
    currency: string;
    completedAt: Date;
  };
}

❌ WRONG:
interface PaymentSucceeded {
  data: {
    payment: Payment;  // Too much data
    wallet: Wallet;   // Crosses aggregate boundary
  };
}
```

---

**Rule 2**: Events contain **IDs**, not objects

**Example**:
```typescript
✅ CORRECT:
interface InvoiceCreated {
  data: {
    paymentId: string;    // Reference by ID
    walletId: string;     // Reference by ID
    invoiceNumber: string;
  };
}

❌ WRONG:
interface InvoiceCreated {
  data: {
    payment: Payment;    // Object - wrong
    wallet: Wallet;      // Object - wrong
  };
}
```

---

**Rule 3**: Events are **immutable**

**Example**:
```typescript
✅ CORRECT:
interface DomainEvent {
  readonly eventType: string;
  readonly eventId: string;
  readonly occurredAt: Date;
  readonly data: EventData;
}

❌ WRONG:
interface DomainEvent {
  eventType: string;  // Mutable
  occurredAt: Date;   // Mutable
}
```

---

## Event Publishing Conventions

### When to Publish Events

**Publish events for**:
- ✅ Aggregate state changes
- ✅ Business rule violations
- ✅ Significant business events
- ✅ Integration needs

**Do NOT publish events for**:
- ❌ Internal state transitions (use lifecycle)
- ❌ Validation successes (expected behavior)
- ❌ Technical operations (logging, monitoring)

---

### Event Publishing Flow

**Standard pattern**:
```typescript
// 1. Perform domain operation
wallet.debit(amount);

// 2. Publish event if state changed
if (wallet.hasUncommittedEvents()) {
  const events = wallet.getUncommittedEvents();
  await eventPublisher.publishBatch(events);
  wallet.markEventsAsCommitted();
}
```

---

## Event Versioning

### Event Evolution Strategy

**Rule**: Events are **immutable** - never modify existing event schema

**Versioning approach**:
```typescript
// Version 1
interface PaymentSucceeded {
  eventType: "PaymentSucceeded";
  version: 1;
  data: {
    amount: number;
    currency: string;
  };
}

// Version 2 (new field added)
interface PaymentSucceeded {
  eventType: "PaymentSucceeded";
  version: 2;
  data: {
    amount: number;
    currency: string;
    paymentMethod?: string;  // Optional new field
  };
}
```

**Backward compatibility**:
- ✅ Add optional fields (new consumers handle missing data)
- ✅ Create new event type for breaking changes
- ❌ Never remove required fields
- ❌ Never change field types

---

## Event Delivery Conventions

### Delivery Guarantees

**Standard**: **At-least-once delivery**

**Implementation**:
- ✅ Use message queue with persistence
- ✅ Implement retry mechanism
- ✅ Use dead letter queue for failures
- ✅ Make event handlers **idempotent**

---

### Retry Strategy

**Standard retry pattern**:
```typescript
const retryConfig = {
  maxRetries: 3,
  backoffStrategy: 'exponential',
  initialDelay: 1000,  // 1 second
  maxDelay: 60000,     // 1 minute
  deadLetterQueue: true
};
```

**Retry logic**:
- ✅ Retry on transient failures (network, timeout)
- ✅ No retry on permanent failures (validation, not found)
- ✅ Exponential backoff between retries
- ✅ Move to dead letter queue after max retries

---

### Idempotency Requirements

**All event handlers must be idempotent**:

```typescript
✅ CORRECT:
class PaymentSucceededHandler {
  async handle(event: PaymentSucceeded) {
    // Check if already processed
    if (await this.isAlreadyProcessed(event.eventId)) {
      return;  // Skip duplicate
    }
    
    // Process event
    await this.createInvoice(event);
    
    // Mark as processed
    await this.markAsProcessed(event.eventId);
  }
}

❌ WRONG:
class PaymentSucceededHandler {
  async handle(event: PaymentSucceeded) {
    // Not idempotent - will create duplicate invoices
    await this.createInvoice(event);
  }
}
```

---

## Event Documentation Conventions

### Event Documentation Structure

**Domain Events** (`{Service}/domains/{domain}/domain-events.md`):
```markdown
## {EventName}

**Trigger**: {When this event is published}

**Payload**:
```typescript
interface {EventName} {
  eventType: "{EventName}";
  aggregateId: string;
  occurredAt: Date;
  data: {
    // Event-specific fields
  };
}
```

**Consumers**: {Who subscribes to this event}
**Example**: {Concrete example}
**Retry**: {Retry strategy}
**Idempotency**: {How idempotency is achieved}
```

**Event Catalog** (`{Service}/events/README.md`):
```markdown
## {EventName}
- **Trigger**: {When published}
- **Consumers**: {Services that subscribe}
- **Documentation**: [Detailed event docs](../domains/{domain}/domain-events.md)
```

---

## Cross-Service Events

### Event Ownership

**Rule**: Events are **owned by the service** that publishes them

**Example**:
```markdown
✅ CORRECT:
- `BillingPaymentSucceeded` - Published by Billing Service
- `BillingInvoiceIssued` - Published by Billing Service

❌ WRONG:
- `PaymentSucceeded` - Ambiguous (which service?)
```

---

### Event Cataloging

**All published events must be catalogued**:
- ✅ In service's `events/README.md`
- ✅ With trigger information
- ✅ With consumer information
- ✅ With documentation links

---

## Common Event Patterns

### Pattern 1: Lifecycle Events

**Events**: `{Aggregate}{StateChange}`

**Examples**:
- `WalletCreated`
- `WalletActivated`
- `WalletFrozen`
- `WalletDeleted`

**Use**: Tracking aggregate lifecycle

---

### Pattern 2: State Change Events

**Events**: `{Concept}{Changed}`

**Examples**:
- `BalanceChanged`
- `StatusChanged`
- `ContactInfoUpdated`

**Use**: Significant state changes

---

### Pattern 3: Business Events

**Events**: `{BusinessAction}{Result}`

**Examples**:
- `PaymentSucceeded`
- `PaymentFailed`
- `InsufficientBalance`
- `InvoiceIssued`

**Use**: Business operation results

---

### Pattern 4: Validation Events

**Events**: `{Validation}{Result}`

**Examples**:
- `InsufficientBalance`
- `DuplicateRequest`
- `InvalidSignature`

**Use**: Business rule violations

---

## Event Schema Examples

### Example 1: Wallet Created

```typescript
interface WalletCreated {
  eventType: "WalletCreated";
  eventId: string;
  aggregateId: string;      // walletId
  aggregateType: "Wallet";
  occurredAt: Date;
  version: 1;
  data: {
    tenantId: string;
    initialBalance: number;
    currency: string;
  };
  metadata: {
    correlationId: string;
    tenantId: string;
  };
}
```

---

### Example 2: Payment Succeeded

```typescript
interface PaymentSucceeded {
  eventType: "PaymentSucceeded";
  eventId: string;
  aggregateId: string;      // paymentId
  aggregateType: "Payment";
  occurredAt: Date;
  version: 1;
  data: {
    amount: number;
    currency: string;
    paymentMethod: string;
    completedAt: Date;
  };
  metadata: {
    correlationId: string;
    tenantId: string;
  };
}
```

---

### Example 3: Insufficient Balance

```typescript
interface InsufficientBalance {
  eventType: "InsufficientBalance";
  eventId: string;
  aggregateId: string;      // walletId
  aggregateType: "Wallet";
  occurredAt: Date;
  version: 1;
  data: {
    requestedAmount: number;
    availableBalance: number;
    currency: string;
    operation: string;      // "debit" or "reserve"
  };
  metadata: {
    correlationId: string;
    tenantId: string;
  };
}
```

---

## Quick Reference

### Event Naming

| Event Type | Format | Examples |
|------------|--------|----------|
| **Lifecycle** | `{Aggregate}{State}` | WalletCreated, WalletDeleted |
| **State Change** | `{Concept}{Changed}` | BalanceChanged, StatusChanged |
| **Business** | `{Action}{Result}` | PaymentSucceeded, PaymentFailed |
| **Validation** | `{Validation}{Result}` | InsufficientBalance, DuplicateRequest |

---

### Event Payload

| Component | Required? | Type | Purpose |
|------------|-----------|------|---------|
| `eventType` | ✅ Yes | string | Event name |
| `eventId` | ✅ Yes | string | Unique event ID |
| `aggregateId` | ✅ Yes | string | Aggregate root ID |
| `aggregateType` | ✅ Yes | string | Aggregate type |
| `occurredAt` | ✅ Yes | Date | When event occurred |
| `version` | ✅ Yes | number | Event version |
| `data` | ✅ Yes | object | Event-specific data |
| `metadata` | ⚠️ Optional | object | Correlation, tracing |

---

## Related Documentation

- [DDD Conventions](./ddd-conventions.md) - Domain-driven design patterns
- [API Conventions](./api-conventions.md) - API design patterns

---

**Convention Version**: 1.0  
**Last Updated**: 2026-07-05  
**Applies To**: All WION microservices
