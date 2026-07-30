# Invoice Domain

## Purpose
Map payments to e-invoices through provider integration (WiInvoice, VNPT, Viettel, etc.) for subscription billing, platform services, and customer billing operations.

## Business Context

### Wion Billing Platform Invoice Types
**Wion Billing Platform** handles e-invoices for:
- **Subscription billing**: Recurring charges for platform services
- **Payment completion**: Automatic invoice generation after successful payments
- **Platform services**: API usage, storage, compute, AI services
- **Customer billing**: Direct customer invoicing
- **Credit transactions**: Wallet top-ups and credit purchases

### Integration Pattern
Based on analysis of billing-management service, e-invoice integration follows:
- **Provider abstraction**: Multiple provider support behind unified interface
- **Authentication**: HMAC-SHA256 signature-based authentication
- **Configuration**: Tenant-specific provider configurations
- **Webhook handling**: Status updates via provider webhooks
- **Background processing**: Async invoice generation and status sync
- **No retry logic**: Failed invoices are marked immediately without automatic retry

## Implementation Status

✅ **Core Invoice Reference IMPLEMENTED** - Basic invoice reference mapping
✅ **E-Invoice Domain Enhanced** - Full lifecycle with provider integration (2026-07-29)
✅ **Retry Logic Removed** - Simplified error handling without automatic retry (2026-07-29)

**⚠️ TDD Violation Note**: This domain was implemented using a code-first approach instead of proper Test-Driven Development. While comprehensive tests exist, they were written after the implementation rather than driving the design.

**E-Invoice Enhancement (2026-07-29)**: The E-Invoice enhancements (status lifecycle, provider settings, business rules) also followed the same code-first approach, violating TDD principles. Tests should have been written first to drive the domain design.

**Future domains MUST follow proper TDD workflow**: See [TDD Playbook](../../../templates/tdd-playbook.md) for the correct Outside-In TDD approach.

**See**: [ADR: TDD Violation Fix](../adr/tdd-violation-fix.md) for details about the violation and remediation strategy.

## Related Documents
- [Invoice Aggregate](./aggregate.md)
- [Invoice Model](./model.md)
- [Invoice Business Rules](./business-rules.md)
- [Invoice Lifecycle](./lifecycle.md)
- [Domain Events](./domain-events.md)
- [E-Invoice Integration](./electronic-invoice-integration.md)
