# Refund Policy

## Purpose
Coordinate refund operations across multiple Aggregates to ensure consistent financial state.

## Trigger
Refund request from application or webhook

## Participating Aggregates
- [Wallet Aggregate](../domains/wallet/aggregate.md) - Credits wallet
- [Credit Transaction Aggregate](../domains/credit-transaction/aggregate.md) - Creates refund transaction
- [Ledger Aggregate](../domains/ledger/aggregate.md) - Creates ledger entries

## Domain Events

### Publishes
- `CreditRefunded` on successful refund
- `BalanceChanged` when wallet credited

## Flow

```
[Refund Request]
      ↓
1. Validate original transaction
      ↓
2. Check refund amount <= original
      ↓
3. Create CreditTransaction (type=REFUND)
      ↓
4. Credit wallet balance
      ↓
5. Create ledger entries
      ↓
6. Publish CreditRefunded event
```

## Validation Rules

### Refund Amount Limit
**Rule**: Refund amount cannot exceed original transaction amount

**Formal Definition**: `refundAmount <= originalTransactionAmount`

**Enforcement**: Validated before refund processing

### Original Transaction Validation
**Rule**: Original transaction must exist and be completed

**Enforcement**: Query CreditTransaction by originalTransactionId

## Failure Handling

### Insufficient Funds
If original transaction amount already consumed:
- Reject refund with error
- Publish `InsufficientBalance` event

### Original Transaction Not Found
- Reject refund with error
- Return 404 Not Found

## Compensation

### On Refund Failure
If wallet credit fails after transaction created:
- Reverse transaction (mark as REVERSED)
- Release any reserved funds
- Publish failure event

## Related Documents
- [Wallet Aggregate](../domains/wallet/aggregate.md) - Wallet operations
- [Credit Transaction Aggregate](../domains/credit-transaction/aggregate.md) - Refund tracking
- [Ledger Aggregate](../domains/ledger/aggregate.md) - Accounting entries
