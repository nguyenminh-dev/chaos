# Payment Business Rules

## Core Business Rules

### BR-P-001: Invoice Only on Successful Payment
**Rule**: Invoice is created only when payment succeeds

**Enforcement**: Invoice creation triggered by `PaymentSucceeded` event

**Related**: [Invoice After Payment Policy](../../policies/invoice-after-payment.md)

---

### BR-P-002: Payment Amount Validation
**Rule**: Payment amount must be positive

**Formal Definition**: `amount > 0`

**Enforcement**: Validated on payment creation

---

### BR-P-003: QR Code Expiration
**Rule**: QR code payments expire after 15 minutes

**Formal Definition**: `createdAt + 15 minutes`

**Enforcement**: Background job cancels expired payments

---

### BR-P-004: Payment Status Transitions
**Rule**: Payment status must follow valid state transitions

**Valid Transitions**:
- PENDING → PROCESSING
- PROCESSING → COMPLETED
- PROCESSING → FAILED
- PENDING → CANCELLED (on expiration)

**Invalid Transitions**:
- COMPLETED → any state (completed payments are immutable)
- FAILED → COMPLETED

---

### BR-P-005: Completed Payment Immutability
**Rule**: Completed payments cannot be modified

**Enforcement**: All update operations reject completed payments

---

## Related Documents
- [Payment Aggregate](./aggregate.md)
- [Payment Lifecycle](./lifecycle.md)
- [Payment Domain Events](./domain-events.md)
