# Consume Credit Use Case

## Business Goal
Enable products to consume credits with real-time balance updates.

## Actor
Application (WION POS, FnB, SPA, AI Services, Platform Services, etc.)

## Trigger
Service request requires credit payment

## Preconditions
- Wallet exists and is active
- Sufficient balance available

## Domain References

### Aggregates Involved
- [Wallet Aggregate](../domains/wallet/aggregate.md) - Balance check and update
- [Credit Transaction Aggregate](../domains/credit-transaction/aggregate.md) - Transaction tracking
- [Ledger Aggregate](../domains/ledger/aggregate.md) - Double-entry accounting

### Business Rules Enforced
- [Wallet business rules](../domains/wallet/business-rules.md) - Non-negative balance (BR-W-002)
- [Credit Transaction business rules](../domains/credit-transaction/business-rules.md) - Idempotency (BR-C-004)

### Policies Applied
- [Refund Policy](../policies/refund-policy.md) - If service fails

## Main Flow
1. Receive consume request with tenantId, amount, service, referenceId, idempotencyKey
2. Validate idempotencyKey (check for duplicate)
3. Load Wallet aggregate
4. Validate sufficient balance
5. Reserve amount (update reserved_balance)
6. Process service request (external)
7. On success: Confirm deduction (debit balance, release reserve)
8. Create Ledger entries (double-entry)
9. Publish `CreditConsumed` event
10. Return success with new balance

## Alternative Flow
- **Insufficient balance** → Publish `InsufficientBalance` event, return 400
- **Duplicate idempotencyKey** → Return cached result
- **Service failure** → Release reserve, return failure

## Failure Flow
- **Repository error** → Return 500, do not consume credit
- **Ledger creation error** → Rollback transaction, return 500

## Postconditions
- Balance deducted
- Ledger entries created
- Event published
- Idempotency maintained

## Acceptance Criteria
- ✓ Returns 400 if insufficient balance
- ✓ Returns cached result on duplicate idempotencyKey
- ✓ Processes 10,000 TPS
- ✓ Creates ledger entries atomically
- ✓ Publishes event on completion
- ✓ Releases reserve on service failure

## Related APIs
- `POST /api/v1/credit/consume` - Consume credits

## Related Events
- `CreditConsumed` - Published on success
- `InsufficientBalance` - Published on balance validation failure
- `BalanceChanged` - Published when wallet credited

## Related Documents
- [Wallet Domain](../domains/wallet/)
- [Credit Transaction Domain](../domains/credit-transaction/)
- [Ledger Domain](../domains/ledger/)
