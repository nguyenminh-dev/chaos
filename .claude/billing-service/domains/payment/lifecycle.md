# Payment Lifecycle

## Lifecycle States

```
[Not Created]
      ↓
   [PENDING]
      ↓
 [PROCESSING]
      ↓
   [COMPLETED]     [FAILED]     [CANCELLED]
      (success)      (failure)    (expired)
```

## State Definitions

### PENDING
**Entry**: Payment created

**Operations**: Allowed to transition to PROCESSING or CANCELLED

**Duration**: Until payment is processed or expires

---

### PROCESSING
**Entry**: Payment being processed

**Operations**: Transitions to COMPLETED or FAILED

**Duration**: Until gateway responds

---

### COMPLETED
**Entry**: Payment successful

**Operations**: No modifications allowed (immutable)

**Duration**: Final state

**Events**: Publishes `PaymentSucceeded`

---

### FAILED
**Entry**: Payment failed

**Operations**: No modifications allowed

**Duration**: Final state

**Events**: Publishes `PaymentFailed`

---

### CANCELLED
**Entry**: Payment expired (15 minutes)

**Operations**: No modifications allowed

**Duration**: Final state

**Events**: Publishes `PaymentExpired`

## State Transitions

| From | To | Trigger |
|------|-----|---------|
| PENDING | PROCESSING | Webhook received |
| PENDING | CANCELLED | 15-minute expiration |
| PROCESSING | COMPLETED | Payment success |
| PROCESSING | FAILED | Payment failure |

## Related Documents
- [Payment Aggregate](./aggregate.md)
- [Payment Business Rules](./business-rules.md)
- [Payment Domain Events](./domain-events.md)
