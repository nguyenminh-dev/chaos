# Business Workflows

This directory contains end-to-end business workflows that span multiple operations and Aggregates.

## Purpose

Workflows document complex business processes that involve:
- Multiple steps or operations
- Cross-aggregate coordination
- State transitions over time
- Integration with external systems

## Difference from Use Cases

| Aspect | Use Cases | Workflows |
|--------|-----------|-----------|
| **Location** | `application/use-cases/` | `flows/` |
| **Scope** | Single operation | Multi-step process |
| **Focus** | Actor + Trigger + Flow | Complete business process |
| **Granularity** | Fine-grained | Coarse-grained |
| **Examples** | Consume Credit | Complete Order-to-Invoice |

## When to Create a Workflow

Create a workflow when:
- ✅ Business process has 3+ distinct steps
- ✅ Multiple aggregates participate
- ✅ External system integration is involved
- ✅ State changes over time (async)
- ✅ Business stakeholders need to understand the full flow

**Don't create** when:
- ❌ Simple CRUD operation
- ❌ Single aggregate operation
- ❌ Already covered by use case

## Workflow Template

Each workflow should include:

```markdown
# {Workflow Name}

## Business Purpose
{Why this workflow exists}

## Actors
{Who participates}

## Prerequisites
{What must be true before starting}

## Workflow Steps
1. {Step 1}
2. {Step 2}
...

## Error Handling
{What happens on failure}

## Related Use Cases
- [{Use Case}](../application/use-cases/{use-case}.md)

## Related Aggregates
- [{Aggregate}](../domains/{aggregate}/aggregate.md)
```

## Available Workflows

- [Payment Topup to Invoice Flow](./payment-topup-to-invoice.md) - Complete payment processing flow
- [Credit Consumption with Retry](./credit-consumption-with-retry.md) - Credit consumption with failure handling

## Related Documentation

- [Use Cases](../application/use-cases/) - Individual operations
- [Policies](../policies/) - Cross-aggregate logic
- [Domains](../domains/) - Business knowledge
