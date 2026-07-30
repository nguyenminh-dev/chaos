# Credit Transaction Aggregate

## Purpose
Handle credit consumption and refunds with idempotency.

## Aggregate Root
**`CreditTransaction`**

## Business Invariants
- Transaction amount must be positive
- Idempotency key must be unique
- Refund cannot exceed original transaction amount

## Domain Events
- `CreditConsumed` - Credit consumed by service
- `CreditRefunded` - Credit refunded
- `BalanceAdjusted` - Admin adjustment

## Transaction Boundary
**Single credit transaction per transaction**

## Related Documents
- [Credit Transaction Model](./model.md)
- [Credit Transaction Business Rules](./business-rules.md)
