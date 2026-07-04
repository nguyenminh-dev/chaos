# Credit Transaction Business Rules

## Core Business Rules

### BR-C-001: Balance Validation Before Consume
**Rule**: Validate sufficient balance before consuming credits

**Enforcement**: Check wallet balance before transaction

**Related**: [Wallet Business Rules](../wallet/business-rules.md)

---

### BR-C-002: All Operations Create Ledger Entries
**Rule**: Every credit operation must create corresponding ledger entries

**Enforcement**: Ledger entry created atomically with transaction

**Related**: [Ledger Business Rules](../ledger/business-rules.md)

---

### BR-C-003: Reference ID Uniqueness Per Source
**Rule**: Reference ID must be unique per source system

**Enforcement**: Database unique constraint on referenceId

---

### BR-C-004: Idempotency Key Required
**Rule**: All mutation operations require idempotency key

**Purpose**: Prevent duplicate transaction processing

**Enforcement**: Unique constraint on idempotencyKey

---

### BR-C-005: Refund Cannot Exceed Original
**Rule**: Refund amount cannot exceed original transaction amount

**Formal Definition**: `refundAmount <= originalTransactionAmount`

**Enforcement**: Validated on refund creation

## Related Documents
- [Credit Transaction Aggregate](./aggregate.md)
- [Credit Transaction Model](./model.md)
