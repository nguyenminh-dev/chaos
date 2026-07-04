# Handle Webhook Use Case

## Business Goal
Process payment gateway callbacks securely with signature verification and idempotency.

## Actor
TPayGate (Payment Gateway)

## Trigger
Webhook received from TPayGate

## Preconditions
- Webhook signature is valid
- Payment exists

## Domain References

### Aggregates Involved
- [Payment Aggregate](../domains/payment/aggregate.md) - Status update
- [Wallet Aggregate](../domains/wallet/aggregate.md) - Credit on success
- [Ledger Aggregate](../domains/ledger/aggregate.md) - Accounting entries
- [Invoice Aggregate](../domains/invoice/aggregate.md) - Triggered on success

### Business Rules Enforced
- [Payment business rules](../domains/payment/business-rules.md) - Status transitions (BR-P-004)
- [Wallet business rules](../domains/wallet/business-rules.md) - Non-negative balance

### Policies Applied
- [Invoice After Payment Policy](../policies/invoice-after-payment.md) - Payment → Invoice flow

## Main Flow
1. Receive webhook with payload and signature
2. Verify HMAC-SHA256 signature
3. Check for duplicate (by gatewayTransactionId)
4. Load Payment aggregate
5. Update Payment status based on event
6. If PAYMENT_SUCCESS:
   - Credit wallet balance
   - Create ledger entries
   - Trigger invoice creation (async via [Invoice After Payment Policy](../policies/invoice-after-payment.md))
   - Publish `PaymentSucceeded` event
7. If PAYMENT_FAILED or TIMEOUT:
   - Publish `PaymentFailed` or `PaymentExpired` event
8. Log webhook event
9. Return 200 OK

## Alternative Flow
- **Invalid signature** → Return 401
- **Duplicate webhook** → Return 200 (idempotent)
- **Payment not found** → Return 404

## Failure Flow
- **Database error** → Retry webhook processing (3 attempts)
- **Ledger error** → Retry webhook processing

## Postconditions
- Payment status updated
- Wallet credited (on success)
- Ledger entries created
- Invoice triggered
- Webhook logged

## Acceptance Criteria
- ✓ Verifies HMAC-SHA256 signature
- ✓ Rejects invalid signatures (401)
- ✓ Handles duplicate webhooks idempotently
- ✓ Credits wallet on payment success
- ✓ Triggers invoice creation
- ✓ Returns 200 within 1 second

## Related APIs
- `POST /api/v1/webhooks/payment` - Webhook endpoint

## Related Events
- `WebhookReceived` - Published on receipt
- `WebhookProcessed` - Published after processing
- `DuplicateWebhookDetected` - Published on duplicate detection

## Related Documents
- [Payment Domain](../domains/payment/)
- [Wallet Domain](../domains/wallet/)
- [Ledger Domain](../domains/ledger/)
- [Invoice Domain](../domains/invoice/)
