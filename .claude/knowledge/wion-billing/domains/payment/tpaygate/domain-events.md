# T-PayGate Domain Events

## Overview
T-PayGate integration domain publishes events for bank connection lifecycle, payment bill processing, and gateway integration monitoring. These events track interactions with the T-PayGate payment gateway provider, which abstracts multiple banking partners.

**Important**: T-PayGate is a payment gateway provider/aggregator, not a payment provider itself. Events reflect gateway operations, while actual payment processing is handled by banking partners through T-PayGate.

## Event Format

### Base Event Schema
```json
{
  "eventId": "guid",
  "eventType": "EventName",
  "timestamp": "2026-07-30T10:30:00Z",
  "tenantId": "tenant-123",
  "correlationId": "txn-456",
  "data": { /* event-specific data */ },
  "metadata": {
    "source": "TpgBankConnection",
    "version": "1.0"
  }
}
```

---

## Published Events

### BankConnectionConnected

**Trigger**: Bank connection successfully established with Virtual Account allocated.

**Purpose**: Notify system that payment gateway is ready for bill creation.

**Consumers**: Monitoring Service, Configuration Service

**Schema**:
```json
{
  "configBankId": "abc-uuid",
  "bankCode": "VCB",
  "vaNumber": "9XYZ200309052356",
  "accountNo": "108800888060",
  "merchantName": "Cửa hàng A",
  "connectedAt": "2026-07-30T10:30:00Z"
}
```

**Business Impact**: Enables bill creation for this bank connection.

---

### BankConnectionDisconnected

**Trigger**: Bank connection terminated (soft delete).

**Purpose**: Notify system that payment gateway is no longer available.

**Consumers**: Monitoring Service, Configuration Service

**Schema**:
```json
{
  "configBankId": "abc-uuid",
  "bankCode": "VCB",
  "disconnectedAt": "2026-07-30T10:30:00Z",
  "reason": "USER_REQUESTED" // or "BANK_REVOKED", "FRAUD_DETECTED"
}
```

**Business Impact**: Prevents new bill creation, existing bills unaffected.

---

### BankOtpRequired

**Trigger**: Bank connection requires OTP verification.

**Purpose**: Alert UI systems to show OTP input form.

**Consumers**: UI Service, Monitoring Service

**Schema**:
```json
{
  "configBankId": "abc-uuid",
  "bankCode": "VCB",
  "otpMethod": "SMS", // or "EMAIL", "APP"
  "expiresAt": "2026-07-30T10:35:00Z",
  "maxRetries": 3
}
```

**Business Impact**: Requires user interaction to complete connection.

---

### TpgBillCreated

**Trigger**: Payment bill created with QR code generated.

**Purpose**: Notify system of new payment request.

**Consumers**: Monitoring Service, Analytics Service

**Schema**:
```json
{
  "billCode": "B202607300001",
  "refTransactionId": "ORDER-2026-0001",
  "tenantId": "tenant-123",
  "amount": 250000,
  "currency": "VND",
  "description": "Thanh toan don hang 0001",
  "qrGenerated": true,
  "expiredAt": "2026-07-31T10:30:00Z",
  "configBankId": "abc-uuid"
}
```

**Business Impact**: Payment bill ready for customer to scan.

---

### TpgBillScanned

**Trigger**: Customer scanned QR code, payment initiated.

**Purpose**: Notify system that payment is in progress.

**Consumers**: Monitoring Service, UI Service

**Schema**:
```json
{
  "billCode": "B202607300001",
  "refTransactionId": "ORDER-2026-0001",
  "tenantId": "tenant-123",
  "scannedAt": "2026-07-30T10:35:00Z"
}
```

**Business Impact**: Bill status transition to WAITING_PAYMENT.

---

### TpgBillPaid

**Trigger**: Payment completed successfully.

**Purpose**: Trigger invoice creation, wallet credit, ledger recording.

**Consumers**: Invoice Service, Wallet Service, Ledger Service, Monitoring Service

**Schema**:
```json
{
  "billCode": "B202607300001",
  "refTransactionId": "ORDER-2026-0001",
  "tenantId": "tenant-123",
  "amount": 250000,
  "currency": "VND",
  "paidAt": "2026-07-30T10:40:00Z",
  "paymentMethod": "BANK_TRANSFER",
  "actualAccount": "1234567890123",
  "configBankId": "abc-uuid"
}
```

**Business Impact**: Triggers downstream fulfillment processes.

---

### TpgBillExpired

**Trigger**: Bill expiry time reached without payment.

**Purpose**: Notify system of payment timeout.

**Consumers**: Monitoring Service, Analytics Service

**Schema**:
```json
{
  "billCode": "B202607300001",
  "refTransactionId": "ORDER-2026-0001",
  "tenantId": "tenant-123",
  "expiredAt": "2026-07-31T10:30:00Z",
  "reason": "TIMEOUT"
}
```

**Business Impact**: Payment not completed, customer may need new bill.

---

### TpgBillCanceled

**Trigger**: Merchant cancelled bill before payment.

**Purpose**: Notify system of bill cancellation.

**Consumers**: Monitoring Service, Analytics Service

**Schema**:
```json
{
  "billCode": "B202607300001",
  "refTransactionId": "ORDER-2026-0001",
  "tenantId": "tenant-123",
  "canceledAt": "2026-07-30T10:32:00Z",
  "reason": "MERCHANT_REQUESTED"
}
```

**Business Impact**: Bill cancelled, customer cannot pay.

---

### TpgPaymentNotificationReceived

**Trigger**: Webhook received from T-PayGate.

**Purpose**: Monitoring webhook delivery rate.

**Consumers**: Monitoring Service

**Schema**:
```json
{
  "billCode": "B202607300001",
  "webhookVersion": "v1", // or "v2" when available
  "receivedAt": "2026-07-30T10:40:00Z",
  "payloadHash": "sha256-abc123"
}
```

**Business Impact**: Monitoring webhook delivery reliability.

---

### TpgPaymentNotificationProcessed

**Trigger**: Payment notification successfully processed.

**Purpose**: Confirm payment applied to bill.

**Consumers**: Monitoring Service, Ledger Service

**Schema**:
```json
{
  "billCode": "B202607300001",
  "refTransactionId": "ORDER-2026-0001",
  "amount": 250000,
  "processedAt": "2026-07-30T10:40:05Z",
  "processingDurationMs": 5000,
  "idempotencyCheck": "PASSED"
}
```

**Business Impact**: Payment successfully recorded.

---

### TpgDuplicatePaymentNotificationDetected

**Trigger**: Duplicate webhook received.

**Purpose**: Alert on duplicate delivery, confirm idempotency.

**Consumers**: Monitoring Service

**Schema**:
```json
{
  "billCode": "B202607300001",
  "duplicateReceivedAt": "2026-07-30T10:41:00Z",
  "originalProcessedAt": "2026-07-30T10:40:05Z",
  "duplicateCount": 1
}
```

**Business Impact**: Confirms idempotency working, no action needed.

---

### TpgPaymentNotificationFailed

**Trigger**: Webhook processing failed.

**Purpose**: Alert on payment processing errors.

**Consumers**: Monitoring Service, Alert Service

**Schema**:
```json
{
  "billCode": "B202607300001",
  "failedAt": "2026-07-30T10:40:00Z",
  "failureReason": "AMOUNT_MISMATCH", // or "BILL_NOT_FOUND", "VALIDATION_ERROR"
  "expectedAmount": 250000,
  "actualAmount": 240000,
  "requiresManualIntervention": true
}
```

**Business Impact**: Payment not applied, requires investigation.

---

### TpgOAuthTokenRefreshed

**Trigger**: OAuth token successfully refreshed.

**Purpose**: Monitor token refresh operations.

**Consumers**: Monitoring Service

**Schema**:
```json
{
  "tenantId": "tenant-123",
  "refreshedAt": "2026-07-30T10:00:00Z",
  "expiresAt": "2026-07-30T11:00:00Z",
  "refreshMethod": "PROACTIVE" // or "REACTIVE"
}
```

**Business Impact**: Confirms authentication renewal.

---

### TpgOAuthTokenRefreshFailed

**Trigger**: OAuth token refresh failed.

**Purpose**: Alert on authentication failures.

**Consumers**: Monitoring Service, Alert Service

**Schema**:
```json
{
  "tenantId": "tenant-123",
  "failedAt": "2026-07-30T10:00:00Z",
  "failureReason": "INVALID_CREDENTIALS", // or "RATE_LIMIT", "SERVICE_UNAVAILABLE"
  "retryAttempts": 3,
  "requiresManualIntervention": true
}
```

**Business Impact**: API access blocked, payment processing unavailable.

---

### TpgWebhookDeliveryFailed

**Trigger**: All webhook retry attempts exhausted.

**Purpose**: Alert on webhook delivery failures.

**Consumers**: Monitoring Service, Alert Service

**Schema**:
```json
{
  "billCode": "B202607300001",
  "deliveryFailedAt": "2026-07-30T10:50:00Z",
  "totalAttempts": 3,
  "lastAttemptAt": "2026-07-30T10:45:00Z",
  "failureReason": "TIMEOUT", // or "CONNECTION_ERROR", "HTTP_5XX"
  "fallbackPollingTriggered": true
}
```

**Business Impact**: Payment notification lost, fallback polling initiated.

---

## Subscribed Events

### TenantDeleted
- **Source**: Tenant Service
- **Handler**: `TenantDeletedHandler`
- **Action**: Soft delete all T-PayGate data for tenant
  - Set `isConnected = false` for all bank connections
  - Cancel all active bills
  - Archive payment notifications
  - Do NOT hard delete (reconciliation requirement)

---

## Event Delivery Guarantees

### Reliability
- **At-Least-Once Delivery**: Guaranteed via RabbitMQ
- **Ordering**: Maintained per tenant
- **Persistence**: 7-day retention in message queue

### Retry Mechanism
- **Attempts**: 3 retry attempts
- **Backoff**: Exponential (1s, 2s, 4s)
- **Dead Letter Queue**: Failed events moved to DLQ

### Performance
- **Publish Latency**: < 50ms (P95)
- **Throughput**: 10,000 events/second
- **Monitoring**: Real-time delivery monitoring

---

## Event Versioning

### Current Version: 1.0
All events currently at version 1.0

### Backward Compatibility
- **Additive Changes**: New fields added with null default
- **Non-Breaking**: Old consumers continue working
- **Breaking Changes**: New event version with migration period

### Version Strategy
- **v1 → v2 Migration**: Minimum 6-month notice
- **Dual Publishing**: Publish both versions during migration
- **Consumer Migration**: Update consumers before v1 sunset

---

## Event-Driven Integration Points

### Upstream (T-PayGate → Billing Service)
- **Webhook Delivery**: HTTP POST to webhook endpoint
- **Internal Events**: Domain events published on webhook processing

### Downstream (Billing Service → Other Services)
- **RabbitMQ**: Event bus for cross-service communication
- **Event Handlers**: Application layer handles events

### Event Handlers
```csharp
// Example: BillPaid handler
public class BillPaidHandler : IEventHandler<TpgBillPaid>
{
    public async Task HandleAsync(TpgBillPaid @event)
    {
        // 1. Update ledger
        await _ledgerService.RecordPaymentAsync(@event);
        
        // 2. Generate invoice
        await _invoiceService.GenerateInvoiceAsync(@event);
        
        // 3. Update wallet (if applicable)
        await _walletService.CreditBalanceAsync(@event);
    }
}
```

---

## Monitoring and Alerting

### Key Metrics
- **Event Publish Rate**: Events/second per event type
- **Event Delivery Success Rate**: % delivered successfully
- **Event Processing Latency**: Time from publish to handler completion
- **Dead Letter Queue Size**: Number of failed events

### Alerting Rules
- **Critical**: Event delivery failure rate > 5%
- **High**: Event processing latency > 5s (P95)
- **Medium**: DLQ size > 100 events
- **Low**: Event publish rate anomalies

### Dashboards
- Real-time event streaming monitor
- Event delivery success rate trends
- Event processing latency heatmaps
- DLQ aging analysis

---

## Related Documents
- [T-PayGate Domain Overview](./overview.md) - Domain context
- [T-PayGate Aggregates](./aggregates.md) - Event publishers
- [Billing Service Event Catalog](../../events/README.md) - Service-wide events
- [T-PayGate Webhook Handling](../../application/tpaygate/webhook-handler.md) - Event publishing logic
