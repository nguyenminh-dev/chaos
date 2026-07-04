# Ledger Aggregate

## Purpose
Maintain double-entry accounting for audit trail.

## Aggregate Root
**`LedgerEntry`**

## Business Invariants
- **Σ Debit = Σ Credit** (double-entry accounting rule)
- Ledger entries are immutable (append-only)
- All financial operations must create ledger entries

## Domain Events
- `TransactionCreated` - Ledger transaction created
- `TransactionCompleted` - Transaction completed
- `TransactionFailed` - Transaction failed

## Transaction Boundary
**Single ledger entry creation**

## Related Documents
- [Ledger Model](./model.md)
- [Ledger Business Rules](./business-rules.md)
