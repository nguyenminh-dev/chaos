# Invoice After Payment Policy

## Purpose
Coordinate invoice creation with payment success, ensuring invoices are issued only after successful payments.

## Trigger
`PaymentSucceeded` domain event from Payment Aggregate

## Participating Aggregates
- [Payment Aggregate](../domains/payment/aggregate.md) - Source of trigger event
- [Invoice Aggregate](../domains/invoice/aggregate.md) - Creates invoice
- [Ledger Aggregate](../domains/ledger/aggregate.md) - Creates ledger entries

## Domain Events

### Consumes
- `PaymentSucceeded` from Payment Aggregate

### Publishes
- `InvoiceIssued` on successful invoice creation
- `InvoiceFailed` on failed invoice creation

## Flow

```
[PaymentSucceeded Event]
      ↓
1. Validate invoice data completeness
      ↓
2. Call Invoice Hub API
      ↓
3. Store invoice reference
      ↓
4. Create ledger entries
      ↓
5. Publish InvoiceIssued event
```

## Invoice Data Requirements

### Required Fields
- Buyer name
- Buyer address
- Tax code (required for companies)
- Payment amount
- Payment date
- Description

### Validation
- All required fields must be present
- Amount must match payment amount
- Tax amount calculated automatically

## Failure Handling

### Retry Strategy
- **Attempt 1**: Immediate
- **Attempt 2**: 30 seconds delay
- **Attempt 3**: 60 seconds delay
- **After 3 failures**: Queue for manual processing

### Manual Processing Queue
- Failed invoices added to `invoice_queue` table
- Finance team processes via dashboard
- Retry after manual correction

## Compensation

### On Payment Refund
If payment is refunded after invoice issued:
- Mark invoice as CANCELLED
- Create credit note in Invoice Hub
- Update ledger entries

### On Invoice Hub Unavailability
- Queue invoice for retry
- Do not fail payment operation
- Continue retrying independently

## Business Rules

### One Invoice Per Payment
**Rule**: Each payment can have only one invoice

**Enforcement**: Unique constraint on `paymentId` in invoice_reference table

### Invoice Number Uniqueness
**Rule**: Invoice number must be unique across system

**Enforcement**: Unique constraint on `invoice_number` field

## Related Documents
- [Payment Aggregate](../domains/payment/aggregate.md) - Payment events
- [Invoice Aggregate](../domains/invoice/aggregate.md) - Invoice lifecycle
- [Invoice Business Rules](../domains/invoice/business-rules.md) - Invoice rules
