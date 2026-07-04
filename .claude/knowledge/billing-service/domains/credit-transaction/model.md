# Credit Transaction Model

## Entities

### CreditTransaction (Aggregate Root)
**Attributes**:
- `id: string` - Transaction ID
- `tenantId: string` - Tenant ID
- `type: TransactionType` - CONSUME, REFUND, ADJUSTMENT
- `status: TransactionStatus` - PENDING, COMPLETED, FAILED, REVERSED
- `amount: decimal` - Transaction amount
- `currency: string` - Currency (VND)
- `service: string` - Service consuming credits
- `referenceId: string` - External reference ID
- `originalTransactionId: string` - Original transaction (for refunds)
- `idempotencyKey: string` - Unique operation key
- `reason: string` - Transaction reason
- `metadata: json` - Additional data
- `createdAt: datetime` - Creation timestamp
- `completedAt: datetime?` - Completion timestamp
- `createdBy: string` - Creator

**Operations**:
- `consume(amount)` - Consume credits
- `refund(amount)` - Refund credits

## Value Objects

### TransactionAmount
**Attributes**: `amount: decimal`, `currency: Currency`

**Invariant**: `amount > 0`

### TransactionType
**Values**: `CONSUME`, `REFUND`, `ADJUSTMENT`

### TransactionStatus
**Values**: `PENDING`, `COMPLETED`, `FAILED`, `REVERSED`

### IdempotencyKey
**Attributes**: `key: string`

**Purpose**: Ensure idempotent operations

## Related Documents
- [Credit Transaction Aggregate](./aggregate.md)
- [Credit Transaction Business Rules](./business-rules.md)
