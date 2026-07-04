# Payment Aggregate

## Purpose
Track payment transactions from initiation to completion with support for QR code payments.

## Aggregate Root
**`Payment`** - The root entity that provides access to the Payment Aggregate

## Entities

### Payment (Aggregate Root)
**Purpose**: Payment transaction tracking

**Key Operations**:
- `initiate(amount, method)` - Create payment
- `complete()` - Mark payment as completed
- `fail(reason)` - Mark payment as failed
- `expire()` - Cancel expired payment

## Value Objects
See [Payment Model](./model.md) for detailed definitions:
- `PaymentAmount` - Amount with currency
- `PaymentStatus` - Status enum
- `PaymentMethod` - Method type enum
- `GatewayReference` - External gateway reference

## Business Invariants
See [Payment Business Rules](./business-rules.md):
- Payment amount must be positive
- Payment status must transition through valid states
- Completed payments cannot be modified
- Expired payments auto-cancel

## Lifecycle
See [Payment Lifecycle](./lifecycle.md):
- Created when payment initiated
- Updated on webhook callback
- Completed/Failed/Cancelled based on outcome

## Domain Events
See [Payment Domain Events](./domain-events.md):
- `PaymentCreated` - Payment initiated
- `PaymentSucceeded` - Payment completed
- `PaymentFailed` - Payment failed
- `PaymentExpired` - Payment timed out

## Repositories
- `IPaymentRepository` - Load/save payments
- `IPaymentWebhookRepository` - Load/save webhook events

## Specifications
- `PendingPaymentSpec` - Query pending payments
- `ExpiredPaymentSpec` - Query expired payments for cleanup

## Transaction Boundary
**Single payment per transaction**

## Related Documents
- [Payment Model](./model.md)
- [Payment Business Rules](./business-rules.md)
- [Payment Lifecycle](./lifecycle.md)
- [Payment Domain Events](./domain-events.md)
