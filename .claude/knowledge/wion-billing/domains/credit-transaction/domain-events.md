# Credit Transaction Domain Events

## Events

### CreditConsumed
**Published**: When credits consumed

**Payload**:
```json
{
  "eventType": "CreditConsumed",
  "transactionId": "txn-12345",
  "userId": "tenant-12345",
  "amount": 1000,
  "service": "AI_GENERATION",
  "referenceId": "req-12345"
}
```

---

### CreditRefunded
**Published**: When credits refunded

**Payload**:
```json
{
  "eventType": "CreditRefunded",
  "transactionId": "txn-refund-12345",
  "originalTransactionId": "txn-12345",
  "userId": "tenant-12345",
  "amount": 1000,
  "reason": "SERVICE_FAILURE"
}
```

---

### BalanceAdjusted
**Published**: On admin balance adjustment

**Payload**:
```json
{
  "eventType": "BalanceAdjusted",
  "transactionId": "txn-adjust-12345",
  "userId": "tenant-12345",
  "amount": 50000,
  "reason": "PROMOTION_GRANT"
}
```

## Related Documents
- [Credit Transaction Aggregate](./aggregate.md)
