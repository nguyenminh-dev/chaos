# Payment Domain

## Purpose
The Payment domain manages payment transactions from initiation through completion. It handles QR code payments via TPayGate integration, payment tracking, and webhook processing.

## Implementation Status

✅ **IMPLEMENTED** - Fully implemented with comprehensive tests

**⚠️ TDD Violation Note**: This domain was implemented using a code-first approach instead of proper Test-Driven Development. While comprehensive tests exist, they were written after the implementation rather than driving the design.

**Future domains MUST follow proper TDD workflow**: See [TDD Playbook](../../../templates/tdd-playbook.md) for the correct Outside-In TDD approach.

**See**: [ADR: TDD Violation Fix](../adr/tdd-violation-fix.md) for details about the violation and remediation strategy.

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
