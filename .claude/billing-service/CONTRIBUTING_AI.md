# Contributing to Billing Service Documentation

This document provides guidelines for maintaining the Billing Service Knowledge Base following Domain-Driven Design (DDD) principles. **All contributors (AI agents and developers) MUST follow these guidelines** to prevent knowledge duplication.

---

## Core Principles

### 1. Single Source of Truth (SSOT)
**Every concept exists in exactly ONE place.**

| Concept | Storage Location | NEVER In |
|---------|-----------------|----------|
| Aggregate Definition | `domains/{aggregate}/aggregate.md` | Use Cases, API docs, multiple files |
| Business Rules | `domains/{aggregate}/business-rules.md` | Use Cases, API docs, database constraints |
| Lifecycle | `domains/{aggregate}/lifecycle.md` | Use Cases, diagrams |
| Domain Events | `domains/{aggregate}/domain-events.md` | API docs, event-index |
| Entities & Value Objects | `domains/{aggregate}/model.md` | Separate entities.md/value-objects.md files |
| Use Case | `application/use-cases/{use-case}.md` | Multiple places |
| Cross-Aggregate Logic | `policies/{policy}.md` | Any Aggregate |

### 2. Domain is Primary
- Business knowledge belongs in `domains/`
- Application layer (`application/use-cases/`) REFERENCES domain, NEVER duplicates it
- API layer (`api/`) REFERENCES domain concepts
- Infrastructure documents technical details, NOT business rules

### 3. Reference, Don't Duplicate
- **ALWAYS** use relative markdown links to reference domain knowledge
- **NEVER** copy-paste business rules from domain documents
- **NEVER** redefine Aggregate concepts in Use Cases or API docs

---

## Where to Document What

### Adding a New Aggregate

**When**: A new domain concept is introduced (e.g., "Subscription" aggregate)

**Create these files in `domains/{aggregate}/`**:
1. `overview.md` - Domain purpose and scope
2. `aggregate.md` - Aggregate Root definition
3. `model.md` - Entities + Value Objects combined (NOT separate files)
4. `business-rules.md` - Business invariants
5. `lifecycle.md` - State transitions
6. `domain-events.md` - Events published
7. `repositories.md` - Repository interfaces (if applicable)

**Template**:
```markdown
# {Domain} Domain

## Purpose
{What this domain handles}

## Scope
{What's included}

## Related Documents
- [{Domain} Aggregate](./aggregate.md)
- [{Domain} Model](./model.md)
- [{Domain} Business Rules](./business-rules.md)
```

---

### Modifying an Existing Aggregate

**When**: Aggregate structure, business rules, or events change

**Files to UPDATE** (in `domains/{aggregate}/`):
- `aggregate.md` - If Aggregate Root changes
- `model.md` - If Entities or Value Objects change
- `business-rules.md` - If business rules change
- `lifecycle.md` - If state transitions change
- `domain-events.md` - If events change
- `repositories.md` - If repository interfaces change

**Files to NEVER update**:
- ❌ Use Cases (they automatically get updated via references)
- ❌ API docs (they reference domain concepts)

**Example**: Changing Wallet Balance Validation
```markdown
# ✅ CORRECT
Edit: domains/wallet/business-rules.md
Add: BR-W-006: Minimum balance requirement

# ❌ WRONG
Edit: application/use-cases/consume-credit.md
❌ Don't re-document the rule here
```

---

### Adding a New Business Rule

**When**: New business invariant is discovered

**Where**: `domains/{aggregate}/business-rules.md`

**Format**:
```markdown
### BR-{DOMAIN}-{NUMBER}: {Rule Name}
**Rule**: {Human-readable description}

**Formal Definition**: {Mathematical/logical definition}

**Enforcement Points**: {Where enforced}

**Violation Handling**: {What happens on violation}

**Purpose**: {Why this rule exists}
```

**NEVER** duplicate business rules in:
- ❌ Use Cases
- ❌ API docs
- ❌ Database constraints (those are technical, not business)

---

### Adding a New Use Case

**When**: New application workflow is needed

**Create**: `application/use-cases/{use-case-name}.md`

**Required Sections**:
```markdown
# {Use Case Name} Use Case

## Business Goal
{What this use case achieves}

## Actor
{Who triggers this}

## Trigger
{What starts this}

## Preconditions
{What must be true}

## Domain References

### Aggregates Involved
- [{Aggregate}](../domains/{aggregate}/aggregate.md) - {Purpose}

### Business Rules Enforced
- [{Aggregate} business rules](../domains/{aggregate}/business-rules.md) - {Which rules}

### Policies Applied
- [{Policy}](../policies/{policy}.md) - {Cross-aggregate logic}

## Main Flow
{Step-by-step process}

## Alternative Flow
{Edge cases}

## Failure Flow
{Error scenarios}

## Postconditions
{What changed}

## Acceptance Criteria
{Verification checklist}
```

**Key Rule**: Use Cases MUST reference Aggregates, NEVER duplicate domain knowledge.

---

### Adding a New Cross-Aggregate Policy

**When**: Business logic spans multiple Aggregates

**Create**: `policies/{policy-name}.md`

**Required Sections**:
```markdown
# {Policy Name} Policy

## Purpose
{What this policy coordinates}

## Trigger
{Event or condition}

## Participating Aggregates
- [{Aggregate}](../domains/{aggregate}/aggregate.md)
- [{Aggregate}](../domains/{aggregate}/aggregate.md)

## Domain Events
### Consumes
- {Event} from {Aggregate}

### Publishes
- {Event}

## Flow
{Step-by-step coordination}

## Failure Handling
{Compensation and recovery}
```

**Examples**:
- Invoice creation after payment → `policies/invoice-after-payment.md`
- Refund coordination → `policies/refund-policy.md`
- Multi-step checkout → `policies/checkout-flow.md`

---

### Documenting a New API

**When**: New endpoint is added

**Update**: `api/index.md`

**Content**:
```markdown
## {API Name}
### Endpoint
`METHOD /api/v1/{path}`

### Purpose
{What this API does}

### Domain Concepts
This API operates on the [{Aggregate}](../domains/{aggregate}/aggregate.md).

### Business Rules
For detailed rules, see [{Aggregate} business rules](../domains/{aggregate}/business-rules.md).

### Events Published
Successful calls publish:
- [{Event}](../domains/{aggregate}/domain-events.md)
```

**NEVER** document business rules in API docs - always reference domain documents.

---

## Change Scenarios

### Scenario 1: Aggregate Changes

**Example**: Wallet Aggregate adds "Credit Limit" feature

**Files to UPDATE**:
1. `domains/wallet/aggregate.md` - Add credit limit to Aggregate definition
2. `domains/wallet/model.md` - Add creditLimit attribute to Wallet entity
3. `domains/wallet/business-rules.md` - Add BR-W-006: Credit limit validation

**Files that REFERENCE automatically update**:
- ✅ Use Cases (no changes needed, they reference via links)
- ✅ API docs (no changes needed)

**Files to NEVER touch**:
- ❌ Don't update Use Cases
- ❌ Don't update API docs
- ❌ Don't duplicate in multiple places

---

### Scenario 2: Use Case Changes

**Example**: "Consume Credit" use case adds retry logic

**Files to UPDATE**:
1. `application/use-cases/consume-credit.md` - Add retry flow

**Files to NEVER update**:
- ❌ Don't update domain documents (no business rule change)

---

### Scenario 3: New Business Rule Discovered

**Example**: "Wallet balance cannot exceed credit limit"

**Files to UPDATE**:
1. `domains/wallet/business-rules.md` - Add BR-W-007

**Files to NEVER update**:
- ❌ Don't update Use Cases (they reference business-rules.md)
- ❌ Don't update API docs (they reference business-rules.md)

---

### Scenario 4: API Changes

**Example**: New endpoint `POST /api/v1/wallets/{id}/freeze`

**Files to UPDATE**:
1. `api/index.md` - Add API documentation

**What to include**:
- Endpoint details
- Links to domain concepts
- Links to business rules (via reference, not duplication)

**What NOT to include**:
- ❌ Business rule definitions (reference domain docs instead)
- ❌ Aggregate definitions (reference domain docs instead)

---

## Anti-Patterns to Avoid

### ❌ Anti-Pattern 1: Duplicating Business Rules

**WRONG**:
```markdown
# application/use-cases/consume-credit.md

## Business Rules
- Balance must be sufficient  ❌ WRONG
- Idempotency key required    ❌ WRONG
```

**CORRECT**:
```markdown
# application/use-cases/consume-credit.md

## Domain References

### Business Rules Enforced
- [Wallet business rules](../domains/wallet/business-rules.md) - Includes balance validation and idempotency
```

---

### ❌ Anti-Pattern 2: Defining Aggregates in Multiple Places

**WRONG**:
- Defining Wallet Aggregate in both `domains/wallet/aggregate.md` AND a feature doc

**CORRECT**:
- Define Wallet Aggregate ONCE in `domains/wallet/aggregate.md`
- All other documents reference it via links

---

### ❌ Anti-Pattern 3: Creating Separate Files for Entities and Value Objects

**WRONG**:
```
domains/wallet/
├── entities.md
└── value-objects.md
```

**CORRECT**:
```
domains/wallet/
└── model.md  # Combined unless >300 lines
```

---

### ❌ Anti-Pattern 4: Placing Business Rules in Wrong Layer

**WRONG**:
- Business rules in Use Cases
- Business rules in API docs
- Business rules in database constraints (beyond technical validation)

**CORRECT**:
- Business rules ONLY in `domains/{aggregate}/business-rules.md`

---

### ❌ Anti-Pattern 5: Cross-Aggregate Logic Inside an Aggregate

**WRONG**:
```markdown
# domains/payment/business-rules.md

### BR-P-006: Invoice After Payment
Invoice must be created after payment succeeds  ❌ WRONG (cross-aggregate)
```

**CORRECT**:
```markdown
# policies/invoice-after-payment.md

## Trigger
PaymentSucceeded event from Payment Aggregate

## Participating Aggregates
- Payment (source)
- Invoice (target)
```

---

## Validation Checklist

Before finalizing any documentation change, verify:

### For Domain Changes
- [ ] Business knowledge exists in exactly ONE place
- [ ] No duplication across documents
- [ ] Use Cases reference (not duplicate) domain knowledge
- [ ] API docs reference (not duplicate) domain concepts
- [ ] All links use relative paths
- [ ] No broken links

### For Application Changes
- [ ] Use Case references Aggregates
- [ ] Use Case references Business Rules
- [ ] Use Case references Policies (if cross-aggregate)
- [ ] No business logic implemented in Use Case

### For API Changes
- [ ] API docs reference domain concepts
- [ ] API docs reference business rules (via links)
- [ ] No business rules defined in API docs
- [ ] Event links point to domain events

---

## File Modification Matrix

| Change Type | Files to Update | Files to Leave Alone |
|--------------|-----------------|---------------------|
| **New Aggregate** | Create 7 files in `domains/{aggregate}/` | All existing files |
| **Aggregate Structure Change** | `domains/{aggregate}/aggregate.md`, `model.md` | Use Cases, API docs (auto-update via refs) |
| **Business Rule Change** | `domains/{aggregate}/business-rules.md` | Use Cases, API docs (auto-update via refs) |
| **New Use Case** | Create 1 file in `application/use-cases/` | Domain files |
| **Use Case Logic Change** | `application/use-cases/{use-case}.md` | Domain files |
| **New API** | `api/index.md` | Domain files |
| **API Contract Change** | `api/index.md` | Domain files |
| **Cross-Aggregate Logic** | Create 1 file in `policies/` | Domain files |
| **Domain Event Change** | `domains/{aggregate}/domain-events.md` | Use Cases, event-index |

---

## Quick Reference

### "Where do I document...?"

| Question | Answer |
|----------|--------|
| "A new Aggregate?" | Create in `domains/{aggregate}/` |
| "A business rule?" | Add to `domains/{aggregate}/business-rules.md` |
| "A new use case?" | Create in `application/use-cases/` |
| "A new API endpoint?" | Add to `api/index.md` |
| "Logic spanning multiple Aggregates?" | Create in `policies/` |
| "Entity or Value Object?" | Add to `domains/{aggregate}/model.md` |
| "Lifecycle or state transitions?" | Add to `domains/{aggregate}/lifecycle.md` |
| "Domain events?" | Add to `domains/{aggregate}/domain-events.md` |

### "What do I update when...?"

| Scenario | Files to Update | Files to Avoid |
|----------|-----------------|--------------|
| "Aggregate changes" | `domains/{aggregate}/aggregate.md`, `model.md`, `business-rules.md` | Use Cases, API docs |
| "Business rule changes" | `domains/{aggregate}/business-rules.md` | Use Cases, API docs |
| "Use case changes" | `application/use-cases/{use-case}.md` | Domain files |
| "API changes" | `api/index.md` | Domain files |
| "New domain event" | `domains/{aggregate}/domain-events.md` | event-index |

---

## Examples

### Example 1: Adding "Credit Limit" Feature

**Step 1**: Update domain documents
```markdown
# domains/wallet/model.md
Add creditLimit attribute to Wallet entity

# domains/wallet/business-rules.md
Add BR-W-007: Credit limit validation
```

**Step 2**: Create or update Use Case (if new workflow)
```markdown
# application/use-cases/check-credit-limit.md

## Domain References
- [Wallet business rules](../domains/wallet/business-rules.md) - BR-W-007
```

**Step 3**: Update API docs (if new API)
```markdown
# api/index.md
## Check Credit Limit API
References: [Wallet Aggregate](../domains/wallet/aggregate.md)
```

---

### Example 2: Modifying "Consume Credit" Use Case

**Step 1**: Update Use Case
```markdown
# application/use-cases/consume-credit.md
Add new retry flow to Main Flow section
```

**Step 2**: DO NOT update domain files
- ❌ Don't touch `domains/wallet/business-rules.md` (no rule change)
- ❌ Don't touch `domains/wallet/aggregate.md` (no structure change)

---

## Documentation Granularity Rules

### DO Create Separate Files When:
- New Aggregate is introduced (7+ files in `domains/{aggregate}/`)
- New Use Case is added (1 file in `application/use-cases/`)
- New cross-aggregate policy (1 file in `policies/`)

### DO Combine Into Single File When:
- Entities + Value Objects → `model.md` (NOT separate files)
- Unless file exceeds 300 lines

### NEVER Create:
- ❌ Separate `entities.md` and `value-objects.md` files (use `model.md`)
- ❌ Feature-based documentation directories
- ❌ Duplicate definitions across multiple files

---

## Links and References

### Relative Path Patterns

```markdown
<!-- Within same domain -->
[Business Rules](./business-rules.md)
[Model](./model.md)

<!-- From Use Case to Domain -->
[Wallet Aggregate](../domains/wallet/aggregate.md)
[Wallet Business Rules](../domains/wallet/business-rules.md)

<!-- From API to Domain -->
[Payment Domain](../domains/payment/aggregate.md)

<!-- From Policy to Domain -->
[Payment Aggregate](../domains/payment/aggregate.md)
[Invoice Aggregate](../domains/invoice/aggregate.md)
```

### Reference Validation

After making changes:
1. Search for the business rule name - should appear ONLY in `business-rules.md`
2. Search for Aggregate name - should appear ONLY in `aggregate.md` (except references)
3. Check all links resolve correctly
4. Verify no duplication across documents

---

## Code Review Checklist

When reviewing documentation changes, verify:

### Domain Changes
- [ ] No business rules duplicated outside `domains/`
- [ ] All Aggregates have complete documentation
- [ ] Business rules have formal definitions
- [ ] Domain events are documented

### Application Changes
- [ ] Use Cases reference Aggregates (not duplicate)
- [ ] No business logic in Use Cases
- [ ] Cross-aggregate logic in `policies/`

### API Changes
- [ ] API docs reference domain concepts
- [ ] No business rules defined in API docs
- [ ] Events link to domain events

### Structure
- [ ] No separate `entities.md` and `value-objects.md` files
- [ ] Model files combined unless >300 lines
- [ ] No feature-based directories

---

## Common Mistakes to Avoid

1. **❌ Copying business rules from domain to use case**
   - **✅ Reference the business-rules.md file instead**

2. **❌ Defining Aggregates in multiple places**
   - **✅ Define once in domain/aggregate.md**

3. **❌ Creating entities.md and value-objects.md separately**
   - **✅ Combine into model.md**

4. **❌ Putting cross-aggregate logic in an Aggregate**
   - **✅ Create a policy in policies/**

5. **❌ Documenting business rules in API docs**
   - **✅ Reference domain business-rules.md**

6. **❌ Updating Use Cases when Aggregate changes**
   - **✅ Let references handle updates automatically**

7. **❌ Documenting implementation details in domain**
   - **✅ Keep domain focused on business logic**

---

## Summary

### Golden Rules
1. **One concept, one place** - No duplication
2. **Domain is primary** - Business knowledge in `domains/`
3. **Reference, don't duplicate** - Use markdown links
4. **Policies for coordination** - Cross-aggregate logic in `policies/`
5. **Model granularity** - Combine unless >300 lines

### Maintenance Workflow
1. Identify what type of change: Domain, Application, API, or Policy
2. Update the correct file(s) based on the matrix above
3. Add references to related documents
4. Validate no duplication introduced
5. Verify all links resolve

By following these guidelines, we maintain a clean, DDD-compliant Knowledge Base with zero duplication and clear separation of concerns.
