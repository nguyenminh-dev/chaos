# Common Implementation Pitfalls

This directory contains common implementation mistakes and business rule violations that developers and AI agents should avoid.

## Purpose

Document:
- ❌ Common mistakes developers make
- ❌ Business rules that are frequently violated
- ❌ Implementation anti-patterns
- ❌ Things AI agents should never do
- ❌ Subtle bugs that are hard to catch

## Target Audience

- **New Developers**: Learn from others' mistakes
- **AI Agents**: Understand what NOT to do
- **Code Reviewers**: Checklist of common issues
- **Architects**: Design better training materials

---

## Pitfall Categories

### Domain Layer Pitfalls
- Violating Aggregate boundaries
- Duplicating business rules
- Placing logic in wrong layer

### Application Layer Pitfalls
- Business logic in use cases
- Missing domain references
- Improper transaction boundaries

### API Layer Pitfalls
- Business rules in controllers
- Missing idempotency
- Incorrect error handling

### Database Pitfalls
- Business rules in constraints
- Missing indexes
- N+1 query problems

### Integration Pitfalls
- Missing webhook verification
- Incorrect event publishing
- Failure handling gaps

---

## Available Pitfalls

### Domain Pitfalls
- [Placing Business Rules in Wrong Layer](./domain/business-rules-in-wrong-layer.md)
- [Duplicating Business Rules Across Files](./domain/duplicating-business-rules.md)
- [Violating Aggregate Boundaries](./domain/violating-aggregate-boundaries.md)
- [Defining Aggregates in Multiple Places](./domain/defining-aggregates-multiple-places.md)

### Application Pitfalls
- [Business Logic in Use Cases](./application/business-logic-in-use-cases.md)
- [Missing Domain References](./application/missing-domain-references.md)
- [Cross-Aggregate Transactions](./application/cross-aggregate-transactions.md)

### API Pitfalls
- [Business Rules in Controllers](./api/business-rules-in-controllers.md)
- [Missing Idempotency Keys](./api/missing-idempotency.md)
- [Returning Wrong HTTP Status](./api/wrong-http-status.md)

### Database Pitfalls
- [Business Rules in Database Constraints](./database/business-rules-in-constraints.md)
- [Missing Transaction Rollback](./database/missing-rollback.md)
- [Concurrency Race Conditions](./database/concurrency-race-conditions.md)

### Integration Pitfalls
- [Missing Webhook Signature Verification](./integration/missing-webhook-verification.md)
- [Non-Idempotent Event Handlers](./integration/non-idempotent-handlers.md)
- [Missing DLQ for Failed Events](./integration/missing-dlq.md)

---

## Using Pitfalls Documentation

### For Code Reviews
Use pitfalls as a checklist:
```markdown
## Code Review Checklist

### Domain Layer
- [ ] No business rules duplicated outside `domains/`
- [ ] Aggregates have proper boundaries
- [ ] No cross-aggregate transactions

### Application Layer
- [ ] Use cases reference domain (not duplicate)
- [ ] No business logic in use cases
- [ ] Proper error handling
```

### For AI Agents
Load pitfalls before making changes:
```bash
# Load pitfalls
load(knowledge/billing-service/pitfalls/)

# Apply to task
"Review changes for common pitfalls"
```

### For Onboarding
New developers should:
1. Read relevant pitfalls before coding
2. Reference during code review
3. Add new pitfalls when discovered

---

## Pitfall Template

```markdown
# {Pitfall Name}

## What It Looks Like
{Code example of the pitfall}

## Why It's Wrong
{Explanation of the problem}

## Consequences
{What happens when you make this mistake}

## How to Fix It
{Correct implementation}

## How to Detect It
{Code review checklist, automated detection}

## Related Pitfalls
- [Links to related mistakes}

## Related Documentation
- [Correct approach documentation]
```

---

## Detection Strategies

### Code Review Detection
- Manual review checklist
- PR template with pitfalls section

### Automated Detection
- Linting rules
- Static analysis
- Test cases that catch pitfalls

### Runtime Detection
- Monitoring alerts
- Error logging
- Metrics for common mistakes

---

## Adding New Pitfalls

### When to Add
Add a new pitfall when:
- ✅ Same mistake made by 2+ developers
- ✅ AI agent makes the mistake repeatedly
- ✅ Code review catches same issue frequently
- ✅ Production incident caused by the mistake

### How to Add
1. Create markdown file in appropriate category
2. Use pitfall template
3. Include real code examples (anonymized)
4. Link to correct documentation
5. Update README index

---

## Related Documentation

- [CONTRIBUTING_AI.md](../CONTRIBUTING_AI.md) - Contribution guidelines
- [Anti-Patterns in CONTRIBUTING_AI](../CONTRIBUTING_AI.md#anti-patterns-to-avoid) - Detailed anti-patterns
- [Shared DDD Conventions](../../shared/ddd-conventions.md) - Correct patterns
- [Domain Documentation](../domains/) - Proper domain modeling

---

## Pitfalls vs. Anti-Patterns

| Aspect | Pitfalls | Anti-Patterns |
|--------|----------|---------------|
| **Location** | `pitfalls/` | `CONTRIBUTING_AI.md` |
| **Scope** | Implementation mistakes | Documentation mistakes |
| **Focus** | Code-level issues | Documentation-level issues |
| **Audience** | Developers, AI agents | Documenters, AI agents |
| **Examples** | Business rule in controller | Duplicating business rules in docs |

---

**Last Updated**: 2026-07-05
**Maintained By**: Billing Service Team
