# Invoice Lifecycle

## Lifecycle States

```
[Not Created]
      ↓
  [PENDING]
      ↓
 [ISSUED]  [FAILED]
  (success) (retry)
      ↓
 [CANCELLED] (on refund)
```

## State Transitions

| From | To | Trigger |
|------|-----|---------|
| PENDING | ISSUED | Invoice Hub success |
| PENDING | FAILED | Invoice Hub failure (retry) |
| PENDING | CANCELLED | Payment refund |
| FAILED | ISSUED | Retry success |

## Related Documents
- [Invoice Aggregate](./aggregate.md)
- [Invoice Business Rules](./business-rules.md)
