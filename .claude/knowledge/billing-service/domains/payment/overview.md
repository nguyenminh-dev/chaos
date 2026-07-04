# Payment Domain

## Purpose
The Payment domain manages payment transactions from initiation through completion. It handles QR code payments via TPayGate integration, payment tracking, and webhook processing.

## Scope
The Payment domain encompasses:
- Payment initiation and QR code generation
- Payment status tracking
- Webhook processing for payment callbacks
- Payment expiration handling

## Bounded Context
Payment operates within the **Financial Operations** bounded context, handling all payment-related operations for the Billing Service.

## Ubiquitous Language
- **Payment**: Financial transaction for wallet topup
- **QR Code**: Payment method for scanning and paying
- **Gateway**: External payment system (TPayGate)
- **Webhook**: Callback from payment gateway
- **Expiration**: 15-minute timeout for payments

## Related Documents
- [Payment Aggregate](./aggregate.md) - Complete Aggregate definition
- [Payment Model](./model.md) - Entities and Value Objects
- [Payment Business Rules](./business-rules.md) - Business invariants
- [Payment Lifecycle](./lifecycle.md) - State transitions
- [Payment Domain Events](./domain-events.md) - Events published
