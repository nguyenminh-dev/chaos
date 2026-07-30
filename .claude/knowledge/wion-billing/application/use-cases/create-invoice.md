# Create Invoice Use Case

## Business Goal
Automatically issue invoices on successful payments through Invoice Hub integration.

## Actor
System (triggered by payment success event)

## Trigger
PaymentSucceeded event received

## Preconditions
- Payment succeeded
- Invoice Hub is available

## Domain References

### Aggregates Involved
- [Invoice Aggregate](../domains/invoice/aggregate.md) - Invoice creation
- [Ledger Aggregate](../domains/ledger/aggregate.md) - Accounting entries

### Business Rules Enforced
- [Invoice business rules](../domains/invoice/business-rules.md) - Retry policy (BR-I-003, BR-I-004)

### Policies Applied
This use case **implements** the [Invoice After Payment Policy](../policies/invoice-after-payment.md)

## Main Flow
1. Receive PaymentSucceeded event
2. Load Payment aggregate
3. Create InvoiceReference aggregate
4. Validate invoice data completeness
5. Call Invoice Hub API to create invoice
6. Update invoice reference with invoice number and URL
7. Create ledger entries
8. Publish `InvoiceIssued` event
9. Return success

## Alternative Flow
- **Invoice Hub unavailable** → Queue for retry
- **Invoice Hub error** → Queue for retry with backoff

## Failure Flow
- **3 retry attempts failed** → Add to manual processing queue
- Publish `InvoiceFailed` event

## Postconditions
- Invoice created in Invoice Hub
- Invoice reference updated
- Ledger entries created

## Acceptance Criteria
- ✓ Creates invoice in Invoice Hub
- ✓ Stores invoice reference
- ✓ Retries 3 times with backoff
- ✓ Queues for manual processing on final failure
- ✓ Publishes InvoiceIssued or InvoiceFailed event

## Related APIs
- `GET /api/v1/invoices/payment/{paymentId}` - Get invoice by payment
- `GET /api/v1/invoices/{invoiceNumber}` - Get invoice by number

## Related Events
- `InvoiceIssued` - Published on successful invoice creation
- `InvoiceFailed` - Published after 3 failed attempts

## Related Documents
- [Invoice Domain](../domains/invoice/)
- [Ledger Domain](../domains/ledger/)
- [Invoice After Payment Policy](../policies/invoice-after-payment.md)
