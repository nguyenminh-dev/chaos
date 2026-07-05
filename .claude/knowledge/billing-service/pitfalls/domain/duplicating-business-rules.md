# Duplicating Business Rules Across Files

## What It Looks Like

### ❌ WRONG: Business Rule Duplicated in Multiple Places

**File 1: `domains/wallet/business-rules.md`**
```markdown
### BR-W-002: Balance Cannot Be Negative
**Rule**: Wallet available balance must never be negative.
**Formal Definition**: availableBalance >= 0
**Enforcement Points**: debit() method
```

**File 2: `application/use-cases/consume-credit.md`**
```markdown
## Business Rules
- Balance must be non-negative  ❌ WRONG (duplicated)
- Sufficient balance required  ❌ WRONG (duplicated)
```

**File 3: `api/wallet-controller.ts`**
```typescript
// Validate balance
if (wallet.balance < amount) {  ❌ WRONG (business rule in controller)
  throw new InsufficientBalanceError();
}
```

---

## Why It's Wrong

1. **Single Source of Truth Violation**: Business rule exists in 3 places
2. **Maintenance Nightmare**: Updating rule requires changing 3 files
3. **Inconsistency Risk**: Files can drift apart over time
4. **Documentation Drift**: One file gets updated, others don't
5. **Testing Complexity**: Need to test same rule in multiple places

---

## Consequences

### Development Impact
- 🔴 **Difficult Maintenance**: Changes require updates in multiple files
- 🔴 **Confusion**: Developers don't know which source is authoritative
- 🔴 **Merge Conflicts**: Same rule changed in different files

### Quality Impact
- 🔴 **Inconsistency**: Rule wording differs between files
- 🔴 **Documentation Drift**: One file updated, others forgotten
- 🔴 **Broken References**: Links point to outdated definitions

### AI Agent Impact
- 🔴 **Conflicting Context**: AI agents get inconsistent information
- 🔴 **Wrong Implementation**: AI agents duplicate instead of referencing
- 🔴 **Broken Tests**: Tests validate wrong version of rule

---

## How to Fix It

### ✅ CORRECT: Define Once, Reference Everywhere

**File 1: `domains/wallet/business-rules.md`** (ONLY definition)
```markdown
### BR-W-002: Balance Cannot Be Negative
**Rule**: Wallet available balance must never be negative.
**Formal Definition**: availableBalance >= 0
**Enforcement Points**: debit() method
**Violation Handling**: Throw InsufficientBalanceException
**Purpose**: Maintain financial integrity
```

**File 2: `application/use-cases/consume-credit.md`** (Reference only)
```markdown
## Domain References

### Business Rules Enforced
- [Wallet business rules](../domains/wallet/business-rules.md) - All wallet rules including balance validation
```

**File 3: `api/wallet-controller.ts`** (No business logic)
```typescript
// Controller delegates to domain
const result = await consumeCreditUseCase.execute(request);
// Business validation happens in domain layer
```

---

## How to Detect It

### Code Review Checklist
- [ ] Search for business rule name - should appear ONLY in `business-rules.md`
- [ ] Check if use cases define business rules - they should reference instead
- [ ] Verify API docs don't define business rules - reference domain docs
- [ ] Ensure controllers don't implement business validation

### Automated Detection
```bash
# Search for business rule duplicates
grep -r "Balance must be non-negative" .claude/knowledge/billing-service/
# Should return exactly 1 result (in business-rules.md)

# Find business rules in wrong places
grep -r "## Business Rules" .claude/knowledge/billing-service/application/
# Should return 0 results (application layer should reference, not define)
```

### Detection Script
```bash
#!/bin/bash
# detect-duplicated-rules.sh

echo "Checking for duplicated business rules..."

# Find all business rule definitions
RULE_FILES=$(grep -l "BR-.*:" .claude/knowledge/billing-service/domains/*/business-rules.md)

for rule in $(grep -oP "BR-\w+-\d+" .claude/knowledge/billing-service/domains/*/business-rules.md); do
  COUNT=$(grep -r "$rule" .claude/knowledge/billing-service/ | wc -l)
  if [ $COUNT -gt 2 ]; then
    echo "⚠️  Rule $rule appears $COUNT times (expected: 2-3 for references)"
    grep -r "$rule" .claude/knowledge/billing-service/
  fi
done
```

---

## Prevention Strategies

### 1. Reference Pattern Training
Teach developers and AI agents to ALWAYS reference:
```markdown
## Business Rules
- [{Domain} business rules](../domains/{domain}/business-rules.md)
```

### 2. Documentation Guidelines
Update `CONTRIBUTING_AI.md` with:
```markdown
### NEVER Define Business Rules Outside Domain
Business rules belong ONLY in `domains/{domain}/business-rules.md`
All other documents reference via links.
```

### 3. Code Review Template
Add to PR template:
```markdown
## Documentation Changes
- [ ] No business rules duplicated outside `domains/`
- [ ] Use cases reference domain documents
- [ ] API docs reference business rules (not define)
```

### 4. AI Agent Prompting
Include in AI agent prompts:
```markdown
When documenting business rules:
1. Define ONLY in domains/{domain}/business-rules.md
2. Reference from other documents via links
3. Never copy-paste business rule definitions
```

---

## Related Pitfalls

- [Placing Business Rules in Wrong Layer](./business-rules-in-wrong-layer.md)
- [Business Logic in Use Cases](../application/business-logic-in-use-cases.md)
- [Business Rules in Controllers](../api/business-rules-in-controllers.md)

---

## Related Documentation

- [Wallet Business Rules](../../domains/wallet/business-rules.md) - Correct location
- [CONTRIBUTING_AI Guidelines](../../CONTRIBUTING_AI.md#-anti-pattern-1-duplicating-business-rules)
- [Shared DDD Conventions](../../../shared/ddd-conventions.md)
- [Domain-Driven Design Principles](../../architecture/README.md)

---

## Real-World Example

### Scenario: Adding Minimum Balance Rule

**❌ WRONG Approach** (Duplicated):
```markdown
# domains/wallet/business-rules.md
### BR-W-006: Minimum Balance
Balance must be >= 10000

# application/use-cases/consume-credit.md
## Business Rules
- Balance must be >= 10000  ❌ DUPLICATED

# api/index.md
## Business Rules
- Minimum balance: 10000  ❌ DUPLICATED AGAIN
```

**✅ CORRECT Approach** (Single source):
```markdown
# domains/wallet/business-rules.md
### BR-W-006: Minimum Balance
**Rule**: Wallet must maintain minimum balance.
**Formal Definition**: availableBalance >= 10000
...

# application/use-cases/consume-credit.md
## Domain References
- [Wallet business rules](../domains/wallet/business-rules.md) - Including BR-W-006

# api/index.md
## Business Rules
For detailed rules, see [Wallet business rules](../domains/wallet/business-rules.md)
```

---

## Detection Metrics

Track these metrics to ensure compliance:

| Metric | Target | How to Measure |
|--------|--------|----------------|
| **Rule Duplication Rate** | 0% | Count rules appearing in >1 file |
| **Reference Compliance** | 100% | Use cases reference domain docs |
| **Documentation Consistency** | 100% | Same rule definition across files |

---

## Checklist

Before finalizing documentation changes:

- [ ] Business rule defined ONLY in `domains/{domain}/business-rules.md`
- [ ] Use cases reference via links (not copy-paste)
- [ ] API docs reference via links (not define)
- [ ] Controllers don't implement business validation
- [ ] No business rules in database constraints (beyond technical)
- [ ] Search for rule name returns 1 definition + references

---

**Last Updated**: 2026-07-05
**Severity**: HIGH
**Frequency**: Common
**Impact**: Maintenance, Consistency, AI Agent Context
