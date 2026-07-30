# Payment Domain Events

## Events

### PaymentCreated
**Published**: When payment initiated

**Payload**:
```json
{
  "eventType": "PaymentCreated",
  "paymentId": "pay-12345",
  "userId": "tenant-12345",
  "amount": 100000,
  "status": "PENDING"
}
```

---

### PaymentSucceeded
**Published**: When payment completed successfully

**Payload**:
```json
{
  "eventType": "PaymentSucceeded",
  "paymentId": "pay-12345",
  "userId": "tenant-12345",
  "amount": 100000,
  "completedAt": "2026-07-04T10:05:00Z"
}
```

**Consumers**:
- Invoice Service - Create invoice
- Wallet Service - Credit wallet
- Ledger Service - Create ledger entries

---

### PaymentFailed
**Published**: When payment failed

**Payload**:
```json
{
  "eventType": "PaymentFailed",
  "paymentId": "pay-12345",
  "userId": "tenant-12345",
  "reason": "TIMEOUT",
  "failedAt": "2026-07-04T10:05:00Z"
}
```

---

### PaymentExpired
**Published**: When payment timed out

**Payload**:
```json
{
  "eventType": "PaymentExpired",
  "paymentId": "pay-12345",
  "userId": "tenant-12345",
  "expiredAt": "2026-07-04T10:15:00Z"
}
```

## Event Delivery
- **Guarantee**: At-least-once
- **Retry**: 3 attempts with exponential backoff
- **Ordering**: Per payment

## Related Documents
- [Payment Aggregate](./aggregate.md)
- [Payment Lifecycle](./lifecycle.md)
- [Payment Business Rules](./business-rules.md)
