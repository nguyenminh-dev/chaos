# Credit Transaction Lifecycle

## Lifecycle States

```
[Not Created]
      ↓
  [PENDING]
      ↓
[COMPLETED]  [FAILED]  [REVERSED]
  (success)   (error)   (rollback)
```

## State Transitions

| From | To | Trigger |
|------|-----|---------|
| PENDING | COMPLETED | Service success |
| PENDING | FAILED | Service failure |
| PENDING | REVERSED | Rollback |

## Related Documents
- [Credit Transaction Aggregate](./aggregate.md)
- [Credit Transaction Business Rules](./business-rules.md)
