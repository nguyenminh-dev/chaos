# Ledger Domain Events

## Events

### TransactionCreated
**Published**: When ledger transaction created

**Payload**:
```json
{
  "eventType": "TransactionCreated",
  "entryId": "ledger-12345",
  "userId": "tenant-12345",
  "transactionId": "txn-12345",
  "debitAccount": "wallet:available",
  "creditAccount": "service:ai_generation",
  "amount": 1000,
  "referenceId": "ref-12345"
}
```

---

### TransactionCompleted
**Published**: When transaction completed

---

### TransactionFailed
**Published**: When transaction failed

## Related Documents
- [Ledger Aggregate](./aggregate.md)
