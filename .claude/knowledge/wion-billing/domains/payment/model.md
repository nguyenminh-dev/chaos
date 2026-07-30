# Payment Model

Entities and Value Objects for the Payment Aggregate.

## Entities

### Payment (Aggregate Root)
**Attributes**:
- `id: string` - Payment ID
- `userId: string` - Tenant ID
- `amount: decimal` - Payment amount
- `currency: string` - Currency (VND)
- `status: PaymentStatus` - Payment status
- `paymentMethod: PaymentMethod` - Payment method
- `gatewayTransactionId: string` - Gateway transaction ID
- `gatewayReference: string` - Gateway reference
- `metadata: json` - Additional data
- `createdAt: datetime` - Creation timestamp
- `completedAt: datetime?` - Completion timestamp
- `expiresAt: datetime?` - Expiration timestamp

**Operations**:
- `initiate()` - Create payment
- `complete()` - Mark as completed
- `fail(reason)` - Mark as failed
- `expire()` - Cancel expired payment
- `isExpired()` - Check expiration

### PaymentWebhook
**Purpose**: Webhook event tracking

**Attributes**:
- `id: string` - Webhook ID
- `gatewayTransactionId: string` - Gateway transaction ID
- `eventType: string` - Event type
- `payload: json` - Webhook payload
- `signature: string` - HMAC signature
- `status: string` - Processing status
- `processingAttempts: int` - Retry count
- `receivedAt: datetime` - Receipt timestamp
- `processedAt: datetime?` - Processing timestamp
- `errorMessage: string` - Error details

## Value Objects

### PaymentAmount
**Attributes**: `amount: decimal`, `currency: Currency`

**Invariant**: `amount > 0`

### PaymentStatus
**Values**: `PENDING`, `PROCESSING`, `COMPLETED`, `FAILED`, `CANCELLED`

### PaymentMethod
**Values**: `QR`, `BANK_TRANSFER`, `CREDIT_CARD`, `E_WALLET`

### GatewayReference
**Attributes**: `gatewayId: string`, `reference: string`

## Related Documents
- [Payment Aggregate](./aggregate.md)
- [Payment Business Rules](./business-rules.md)
