# Consume Credit Success Example

## Operation
Consume credits from wallet for service usage.

## Request

### HTTP Method
```
POST /api/v1/credit/consume
```

### Request Headers
```http
Authorization: Bearer test-api-key-12345
Content-Type: application/json
X-Idempotency-Key: unique-key-123
X-Tenant-Id: tenant-12345
```

### Request Body
```json
{
  "tenantId": "tenant-12345",
  "amount": 1000,
  "service": "AI_GENERATION",
  "referenceId": "req-12345",
  "idempotencyKey": "unique-key-123",
  "metadata": {
    "resourceType": "TOKEN",
    "quantity": 1000,
    "model": "gpt-4",
    "promptTokens": 500,
    "completionTokens": 500
  }
}
```

---

## Response

### HTTP Status
```
200 OK
```

### Response Headers
```http
Content-Type: application/json
X-Request-Id: req-abc-123
X-Response-Time: 45ms
```

### Response Body
```json
{
  "transactionId": "txn-abc-123",
  "status": "COMPLETED",
  "amount": 1000,
  "remainingBalance": 99000,
  "reservedBalance": 0,
  "totalBalance": 99000,
  "currency": "VND",
  "createdAt": "2026-07-05T10:00:00Z",
  "completedAt": "2026-07-05T10:00:00Z"
}
```

---

## Domain Events Published

### CreditConsumed Event
```json
{
  "eventType": "CreditConsumed",
  "eventId": "evt-xyz-789",
  "timestamp": "2026-07-05T10:00:00Z",
  "data": {
    "transactionId": "txn-abc-123",
    "tenantId": "tenant-12345",
    "amount": 1000,
    "service": "AI_GENERATION",
    "referenceId": "req-12345",
    "remainingBalance": 99000
  }
}
```

### BalanceChanged Event
```json
{
  "eventType": "BalanceChanged",
  "eventId": "evt-balance-123",
  "timestamp": "2026-07-05T10:00:00Z",
  "data": {
    "tenantId": "tenant-12345",
    "previousBalance": 100000,
    "newBalance": 99000,
    "changeAmount": -1000,
    "reason": "CREDIT_CONSUMED"
  }
}
```

---

## Business Rules Enforced

1. **Non-negative Balance**: `availableBalance >= 0`
   - Enforced by: Wallet.debit() method
   - Documentation: [Wallet Business Rules](../../domains/wallet/business-rules.md#br-w-002)

2. **Idempotency**: Unique idempotency key required
   - Enforced by: CreditTransaction aggregate
   - Documentation: [Credit Transaction Business Rules](../../domains/credit-transaction/business-rules.md#br-c-004)

---

## Related Documentation

- [Consume Credit Use Case](../../application/use-cases/consume-credit.md)
- [Wallet Domain](../../domains/wallet/aggregate.md)
- [Credit Transaction Domain](../../domains/credit-transaction/aggregate.md)
- [API Documentation](../api/index.md#consume-credit)

---

## Integration Example

### Node.js
```javascript
const consumeCredit = async (tenantId, amount, service, referenceId) => {
  const response = await fetch('http://localhost:3000/api/v1/credit/consume', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${API_KEY}`,
      'X-Idempotency-Key': uuidv4()
    },
    body: JSON.stringify({
      tenantId,
      amount,
      service,
      referenceId,
      idempotencyKey: uuidv4()
    })
  });

  if (!response.ok) {
    throw new Error(`Credit consumption failed: ${response.statusText}`);
  }

  return response.json();
};

// Usage
const result = await consumeCredit(
  'tenant-12345',
  1000,
  'AI_GENERATION',
  'req-12345'
);
console.log(`Transaction ${result.transactionId} completed`);
```

### Python
```python
import requests
import uuid

def consume_credit(tenant_id: int, amount: int, service: str, reference_id: str):
    url = "http://localhost:3000/api/v1/credit/consume"
    headers = {
        "Content-Type": "application/json",
        "Authorization": f"Bearer {API_KEY}",
        "X-Idempotency-Key": str(uuid.uuid4())
    }
    payload = {
        "tenantId": tenant_id,
        "amount": amount,
        "service": service,
        "referenceId": reference_id,
        "idempotencyKey": str(uuid.uuid4())
    }
    
    response = requests.post(url, json=payload, headers=headers)
    response.raise_for_status()
    return response.json()

# Usage
result = consume_credit(
    tenant_id="tenant-12345",
    amount=1000,
    service="AI_GENERATION",
    reference_id="req-12345"
)
print(f"Transaction {result['transactionId']} completed")
```

---

**Last Updated**: 2026-07-05
**Example Version**: 1.0
