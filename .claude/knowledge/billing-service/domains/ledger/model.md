# Ledger Model

## Entities

### LedgerEntry (Aggregate Root)
**Attributes**:
- `id: string` - Entry ID
- `tenantId: string` - Tenant ID
- `transactionId: string` - Source transaction ID
- `transactionType: TransactionType` - PAYMENT, CONSUME, REFUND, ADJUSTMENT, TRANSFER
- `debitAccount: string` - Debit account identifier
- `creditAccount: string` - Credit account identifier
- `amount: decimal` - Entry amount
- `currency: string` - Currency (VND)
- `referenceType: ReferenceType` - PAYMENT_GATEWAY, INVOICE, MANUAL
- `referenceId: string` - Reference ID
- `metadata: json` - Additional data
- `createdAt: datetime` - Creation timestamp
- `createdBy: string` - Creator

**Operations**:
- `create(debitAccount, creditAccount, amount)` - Create entry

**Immutability**: Never modified once created (append-only)

## Value Objects

### Account
**Attributes**: `accountId: string`, `accountType: string`

**Purpose**: Debit or credit account identifier

### EntryAmount
**Attributes**: `amount: decimal`, `currency: Currency`

**Invariant**: `amount > 0`

### TransactionReference
**Attributes**: `referenceType: string`, `referenceId: string`

**Purpose**: Link to source transaction

## Related Documents
- [Ledger Aggregate](./aggregate.md)
- [Ledger Business Rules](./business-rules.md)
