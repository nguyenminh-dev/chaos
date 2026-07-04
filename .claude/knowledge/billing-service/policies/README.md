# Policies - Cross-Aggregate Business Logic

This directory contains policies that orchestrate business logic across multiple aggregates in the Billing Service.

## Purpose
Policies implement business rules that cannot be contained within a single aggregate boundary. They coordinate multiple aggregates while maintaining consistency.

## Key Principles

### Policy vs Use Case
- **Use Cases**: Application layer, user-triggered workflows
- **Policies**: Business logic triggered by domain events

### Transaction Boundaries
- Policies maintain aggregate boundaries
- Each aggregate operation is independent
- Eventual consistency between aggregates

### Event-Driven
- Policies subscribe to domain events
- Execute business logic asynchronously
- Publish new domain events if needed

## Policies

### [Invoice After Payment Policy](./invoice-after-payment.md)
**Trigger**: `PaymentSucceeded` event

**Purpose**: Automatically generate electronic invoices when payments succeed

**Aggregates Involved**:
- Payment (event source)
- Invoice (target)

**Business Logic**:
1. Receive PaymentSucceeded event
2. Create InvoiceReference aggregate
3. Call Invoice Hub API
4. Update invoice status
5. Publish InvoiceIssued or InvoiceFailed event

**Failure Handling**:
- 3 retries with exponential backoff
- Manual processing queue on final failure

---

### [Refund Policy](./refund-policy.md)
**Trigger**: Refund request

**Purpose**: Handle credit refunds while maintaining ledger consistency

**Aggregates Involved**:
- CreditTransaction (refund orchestration)
- Wallet (balance restoration)
- Ledger (double-entry accounting)

**Business Logic**:
1. Validate refund eligibility
2. Check original transaction amount
3. Create CreditTransaction (type: REFUND)
4. Restore wallet balance
5. Create ledger entries
6. Publish CreditRefunded event

**Failure Handling**:
- Idempotent refunds
- Audit trail maintenance

## Policy Characteristics

### Coordination
- Multiple aggregates involved
- Orchestrates domain operations
- Maintains business invariants

### Consistency
- Eventual consistency between aggregates
- No distributed transactions
- Idempotent operations

### Reliability
- Event-driven execution
- Retry mechanisms
- Dead letter queue handling

## Future Policies
As the Billing Service evolves, additional policies may be added:
- Payment reconciliation policy
- Low balance alert policy
- Invoice retry policy
- Ledger consolidation policy
