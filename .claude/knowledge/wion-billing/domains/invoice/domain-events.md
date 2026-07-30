# Invoice Domain Events

## Events

### InvoiceIssued
**Published**: When invoice issued successfully

**Payload**:
```json
{
  "eventType": "InvoiceIssued",
  "invoiceId": "inv-12345",
  "paymentId": "pay-12345",
  "invoiceNumber": "INV20260704001",
  "amount": 100000,
  "issuedAt": "2026-07-04T10:06:00Z"
}
```

---

### InvoiceFailed
**Published**: When invoice creation failed

**Payload**:
```json
{
  "eventType": "InvoiceFailed",
  "paymentId": "pay-12345",
  "reason": "INVOICE_HUB_UNAVAILABLE",
  "retryCount": 3,
  "queuedForManual": true
}
```

## Related Documents
- [Invoice Aggregate](./aggregate.md)
- [Invoice After Payment Policy](../../policies/invoice-after-payment.md)
