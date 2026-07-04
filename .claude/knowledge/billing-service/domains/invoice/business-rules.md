# Invoice Business Rules

## Core Business Rules

### BR-I-001: One Invoice Per Payment
**Rule**: Each payment can have only one invoice

**Enforcement**: Unique constraint on paymentId

---

### BR-I-002: Invoice Data Completeness
**Rule**: Invoice data must be complete before issuance

**Required Fields**:
- Buyer name
- Buyer address
- Tax code (for companies)
- Payment amount
- Payment date
- Description

**Enforcement**: Validation before Invoice Hub call

---

### BR-I-003: Retry on Failure
**Rule**: Failed invoice creation must be retried (3 attempts)

**Retry Strategy**:
- Attempt 1: Immediate
- Attempt 2: 30 seconds delay
- Attempt 3: 60 seconds delay

**Enforcement**: Automatic retry with exponential backoff

---

### BR-I-004: Manual Queue After Retries
**Rule**: After 3 failed attempts, invoice queued for manual processing

**Purpose**: Ensure invoices are eventually issued

**Enforcement**: Background job manages manual queue

---

### BR-I-005: Invoice Only on Payment Success
**Rule**: Invoice created only when payment succeeds

**Related**: [Invoice After Payment Policy](../../policies/invoice-after-payment.md)

## Related Documents
- [Invoice Aggregate](./aggregate.md)
- [Invoice Model](./model.md)
