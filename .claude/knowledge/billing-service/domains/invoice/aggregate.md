# Invoice Aggregate

## Purpose
Map payments to electronic invoices.

## Aggregate Root
**`InvoiceReference`**

## Business Invariants
- One invoice per payment
- Invoice number must be unique
- Failed invoice creation must be retried (3 attempts)

## Domain Events
- `InvoiceIssued` - Invoice issued successfully
- `InvoiceFailed` - Invoice creation failed

## Transaction Boundary
**Single invoice reference per transaction**

## Related Documents
- [Invoice Model](./model.md)
- [Invoice Business Rules](./business-rules.md)
