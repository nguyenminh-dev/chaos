# Process Payment Use Case

## Business Goal
Enable tenants to topup credits through QR code payments.

## Actor
Tenant/User

## Trigger
User initiates topup request

## Preconditions
- Wallet exists
- Amount is positive and within limits

## Domain References

### Aggregates Involved
- [Payment Aggregate](../domains/payment/aggregate.md) - Payment tracking
- [Wallet Aggregate](../domains/wallet/aggregate.md) - Wallet to be credited

### Business Rules Enforced
- [Payment business rules](../domains/payment/business-rules.md) - Amount validation (BR-P-002)

### Policies Applied
- [Invoice After Payment Policy](../policies/invoice-after-payment.md) - Triggered on success

## Main Flow
1. Receive payment request with tenantId, amount, paymentMethod
2. Validate request
3. Create Payment aggregate (status: PENDING)
4. Call TPayGate API to create payment
5. Store gateway reference
6. Publish `PaymentCreated` event
7. Return QR code to user
8. Wait for webhook callback (async)

## Alternative Flow
- **Invalid amount** → Return 400
- **TPayGate error** → Update Payment status to FAILED, return 500

## Failure Flow
- **TPayGate timeout** → Retry 3 times, then fail payment
- **Network error** → Return 503

## Postconditions
- Payment created in database
- QR code returned to user
- Waiting for webhook callback

## Acceptance Criteria
- ✓ Creates payment with PENDING status
- ✓ Returns QR code within 500ms
- ✓ Handles TPayGate errors gracefully
- ✓ Publishes PaymentCreated event

## Related APIs
- `POST /api/v1/payments/topup` - Initiate topup
- `GET /api/v1/payments/{paymentId}` - Get payment status

## Related Events
- `PaymentCreated` - Published when payment initiated
- `PaymentSucceeded` - Published on successful completion (via webhook)
- `PaymentFailed` - Published on payment failure

## Related Documents
- [Payment Domain](../domains/payment/)
- [Wallet Domain](../domains/wallet/)
