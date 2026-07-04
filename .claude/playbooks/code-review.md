# Code Review Playbook

**Purpose**: Comprehensive code review workflow ensuring DDD compliance, documentation quality, and architectural integrity.

**When to Use**: When reviewing pull requests, code changes, or implementation proposals.

---

## Phase 1: Pre-Review Preparation

### Step 1.1: Understand the Change

**Load these files first**:
1. `CONTRIBUTING_AI.md` - Review principles
2. `architecture/change-impact-matrix.md` - Expected impact
3. **Pull request description** - What changed and why

**Key Questions**:
- What is the purpose of this change?
- Which files were modified?
- Which domains/aggregates are affected?
- What is the expected impact?

---

### Step 1.2: Load Affected Documentation

**Based on change type**, load relevant documentation:

**For Domain Changes**:
```bash
# Load modified domain files
domains/{affected-domain}/aggregate.md
domains/{affected-domain}/business-rules.md
domains/{affected-domain}/model.md
```

**For Use Case Changes**:
```bash
# Load modified use case
application/use-cases/{affected-use-case}.md

# Load referenced domains
domains/{referenced-domains}/aggregate.md
domains/{referenced-domains}/business-rules.md
```

**For API Changes**:
```bash
# Load API documentation
api/index.md

# Load referenced domains
domains/{affected-domain}/aggregate.md
```

**Expected Outcome**: Context loaded for review.

---

## Phase 2: Architecture Compliance Review

### Step 2.1: DDD Compliance Check

**Verify**: Business Rules Location

**Check**:
```bash
# Search for business rules - should appear ONLY in business-rules.md
grep -r "BR-" domains/
```

**Expected Result**: Business rules appear ONLY in `domains/*/business-rules.md`

**Red Flags**:
- ❌ Business rules in Use Cases
- ❌ Business rules in API docs
- ❌ Business rules in controllers
- ❌ Business rules duplicated across files

---

**Verify**: Aggregate Boundaries

**Check**:
- Are aggregates consistent with domain documentation?
- Are aggregate boundaries maintained?
- No cross-aggregate transactions?
- Eventual consistency between aggregates?

**Red Flags**:
- ❌ Cross-aggregate transactions
- ❌ Aggregate boundaries violated
- ❌ Direct database access bypassing aggregates

---

**Verify**: Single Source of Truth

**Check**:
- Each concept exists in exactly one place?
- No duplicated definitions?
- References instead of duplication?

**Red Flags**:
- ❌ Same business rule in multiple files
- ❌ Same aggregate definition in multiple places
- ❌ Same event schema documented twice

---

### Step 2.2: Clean Architecture Check

**Verify**: Dependency Direction

**Check**:
- Dependencies point inward?
- Domain has no dependencies on infrastructure?
- Application layer coordinates, doesn't contain logic?

**Red Flags**:
- ❌ Domain depends on infrastructure
- ❌ Domain depends on framework
- ❌ Business logic in application layer

---

**Verify**: Layer Separation

**Check**:
- Domain layer contains only business logic?
- Application layer only coordinates?
- Infrastructure only implements domain interfaces?

**Red Flags**:
- ❌ Business rules in controllers
- ❌ Business rules in repositories
- ❌ Business rules in external services

---

## Phase 3: Documentation Quality Review

### Step 3.1: Documentation Completeness Check

**For Domain Changes**:

**Check**: Domain files have required sections
- [ ] `aggregate.md` - Aggregate root, entities, operations
- [ ] `model.md` - Value objects, types
- [ ] `business-rules.md` - Business invariants with formal definitions
- [ ] `lifecycle.md` - State transitions
- [ ] `domain-events.md` - Published events with payloads

**For Use Case Changes**:

**Check**: Use case has required sections
- [ ] Business goal
- [ ] Actor
- [ ] Trigger
- [ ] Domain References section (critical!)
- [ ] Main flow
- [ ] Alternative flows
- [ ] Failure flows
- [ ] Acceptance criteria
- [ ] Related APIs
- [ ] Related events

**For API Changes**:

**Check**: API documentation has required sections
- [ ] Endpoint
- [ ] Purpose
- [ ] Domain concepts referenced (not duplicated)
- [ ] Request/response examples
- [ ] Error codes
- [ ] Related events

---

### Step 3.2: Reference Quality Check

**Verify**: Domain References

**Check**: Use cases and APIs reference domain concepts
```markdown
## Domain References
### Aggregates Involved
- [Wallet Aggregate](../domains/wallet/aggregate.md)
```

**Red Flags**:
- ❌ Use cases duplicate domain knowledge
- ❌ API docs duplicate business rules
- ❌ Missing references to domain concepts

---

**Verify**: Link Validity

**Check**: All links resolve correctly
- Relative paths correct
- Cross-domain references use correct paths
- No broken links

**Red Flags**:
- ❌ Broken reference links
- ❌ Incorrect relative paths
- ❌ References to non-existent files

---

### Step 3.3: Business Rule Documentation Check

**Verify**: Business Rules Have Formal Definitions

**Check**: Each business rule has:
```markdown
### BR-{LETTER}-{NUMBER}: {Rule Name}
**Rule**: {Human-readable description}
**Formal Definition**: {Mathematical/logical definition}
**Enforcement Points**: {Where enforced}
**Violation Handling**: {What happens on violation}
```

**Red Flags**:
- ❌ Missing formal definition
- ❌ Missing enforcement points
- ❌ Missing violation handling
- ❌ Informal language only

---

## Phase 4: Implementation Quality Review

### Step 4.1: Code-Domain Alignment Check

**Verify**: Implementation matches domain documentation

**Check**:
- Are business rules enforced in code?
- Are aggregate operations implemented?
- Are domain events published?
- Are invariants maintained?

**Red Flags**:
- ❌ Business rule documented but not enforced
- ❌ Aggregate operation documented but not implemented
- ❌ Domain event documented but not published

---

### Step 4.2: Testing Quality Check

**Verify**: Tests cover documentation

**Check**:
- Tests for business rules?
- Tests for aggregate operations?
- Tests for use case workflows?
- Tests for error scenarios?
- Tests for edge cases?

**Red Flags**:
- ❌ Untested business rules
- ❌ Untested error scenarios
- ❌ Missing edge case tests
- ❌ Tests don't match documentation

---

### Step 4.3: Error Handling Check

**Verify**: Error handling documented and implemented

**Check**:
- Error codes documented in API docs?
- Error scenarios documented in use cases?
- Failure flows documented?
- Domain events for errors published?

**Red Flags**:
- ❌ Undocumented error codes
- ❌ Missing error scenarios
- ❌ No failure flow documentation
- ❌ Errors don't publish events

---

## Phase 5: Integration Review

### Step 5.1: External Integration Check

**Verify**: External service integrations documented

**Check**:
- External services documented in `infrastructure/`?
- Integration patterns consistent?
- Error handling documented?
- Retry strategies documented?

**Red Flags**:
- ❌ Undocumented external service calls
- ❌ Inconsistent integration patterns
- ❌ Missing error handling
- ❌ No retry documentation

---

### Step 5.2: Event Integration Check

**Verify**: Event publishing and handling documented

**Check**:
- Events documented in `domain-events.md`?
- Events catalogued in `events/README.md`?
- Event subscribers documented?
- Event schemas defined?

**Red Flags**:
- ❌ Undocumented events
- ❌ Events not in catalog
- ❌ Missing event schemas
- ❌ Undocumented event subscribers

---

## Phase 6: Regression Review

### Step 6.1: Impact Analysis Check

**Verify**: Change impact properly analyzed

**Check**:
- All affected files identified?
- All impacted domains identified?
- All impacted use cases identified?
- Change impact matrix updated?

**Red Flags**:
- ❌ Missing impact analysis
- ❌ Unidentified affected domains
- ❌ Unidentified breaking changes

---

### Step 6.2: Backward Compatibility Check

**Verify**: No breaking changes without documentation

**Check**:
- API changes documented?
- Event changes documented?
- Business rule changes documented?
- Migration guide provided if needed?

**Red Flags**:
- ❌ Undocumented breaking changes
- ❌ API contract changes without version bump
- ❌ Event schema changes without version
- ❌ No migration guide for breaking changes

---

## Phase 7: Approval Decision

### Step 7.1: Review Summary

**Document findings**:
- ✅ Approve: No issues found
- ✅ Approve with suggestions: Minor improvements suggested
- ⚠️ Request changes: Issues that must be fixed
- ❌ Reject: Fundamental architectural violations

---

### Step 7.2: Feedback Generation

**For Approval with Suggestions**:
- List suggested improvements
- Explain why suggestions are made
- Provide examples if helpful

**For Request Changes**:
- List issues that must be fixed
- Explain why each issue is blocking
- Provide guidance on how to fix
- Reference relevant documentation

**For Rejection**:
- Explain architectural violations
- Cite DDD principles violated
- Explain impact on system
- Suggest fundamental redesign

---

## Common Review Findings

### Finding 1: Duplicated Business Knowledge

**Severity**: 🔴 **HIGH** - Must fix

**Example**:
```markdown
# application/use-cases/consume-credit.md
## Business Rules
- Balance must be sufficient  ❌ WRONG
```

**Fix**:
```markdown
# application/use-cases/consume-credit.md
## Domain References
### Business Rules Enforced
- [Wallet business rules](../domains/wallet/business-rules.md)
```

---

### Finding 2: Business Rules in Wrong Layer

**Severity**: 🔴 **HIGH** - Must fix

**Example**:
- Business rule in controller
- Business rule in repository
- Business rule in external service

**Fix**: Move business rule to domain layer

---

### Finding 3: Missing Domain References

**Severity**: 🟡 **MEDIUM** - Should fix

**Example**:
```markdown
# api/index.md
## Get Wallet Balance
This API checks if balance is sufficient  ❌ WRONG
```

**Fix**:
```markdown
# api/index.md
## Get Wallet Balance
This API operates on the [Wallet Aggregate](../domains/wallet/aggregate.md) and enforces [Wallet business rules](../domains/wallet/business-rules.md).
```

---

### Finding 4: Incomplete Business Rule Documentation

**Severity**: 🟡 **MEDIUM** - Should fix

**Example**:
```markdown
### BR-W-002: Balance Cannot Be Negative
Balance must be non-negative.  ❌ INCOMPLETE
```

**Fix**:
```markdown
### BR-W-002: Balance Cannot Be Negative
**Rule**: Wallet available balance must never be negative.
**Formal Definition**: availableBalance >= 0
**Enforcement Points**: debit(), reserve() operations
**Violation Handling**: Reject with "Insufficient balance" error
```

---

### Finding 5: Missing Tests

**Severity**: 🟡 **MEDIUM** - Should fix

**Example**:
- Business rule implemented but no tests
- Edge case not tested
- Error scenario not tested

**Fix**: Add comprehensive tests

---

### Finding 6: Inconsistent Naming

**Severity**: 🟢 **LOW** - Nice to fix

**Example**:
- "credit-transaction" vs "creditTransaction"
- "wallet_id" vs "walletId"

**Fix**: Use consistent naming

---

## Review Checklist

### Before Approving

**Architecture Compliance**:
- [ ] DDD principles followed
- [ ] Clean architecture maintained
- [ ] Single source of truth
- [ ] Aggregate boundaries preserved
- [ ] No cross-aggregate transactions

**Documentation Quality**:
- [ ] All required sections present
- [ ] Domain references correct
- [ ] No duplicated knowledge
- [ ] Links resolve correctly
- [ ] Business rules have formal definitions

**Implementation Quality**:
- [ ] Code matches documentation
- [ ] Tests comprehensive
- [ ] Error handling documented
- [ ] Events documented
- [ ] Integrations documented

**Regression Prevention**:
- [ ] Impact analysis complete
- [ ] Backward compatibility checked
- [ ] Breaking changes documented
- [ ] No architectural violations

---

## Quick Reference

### "Issue in X, where do I check?"

| Issue Type | Documentation Files | What to Check |
|-----------|-------------------|---------------|
| **DDD violation** | `CONTRIBUTING_AI.md` | DDD principles |
| **Business rule duplication** | `domains/*/business-rules.md` | Single source of truth |
| **Missing domain reference** | Use cases, API docs | References exist |
| **Broken link** | All files | Link validity |
| **Incomplete documentation** | Domain files, use cases | Required sections |
| **Implementation mismatch** | Code vs documentation | Alignment |
| **Missing tests** | Test files | Coverage |

---

## Success Criteria

### Review Complete When:

- ✅ All architectural violations identified
- ✅ All documentation issues identified
- ✅ All implementation issues identified
- ✅ Feedback provided to author
- ✅ Approval decision made
- ✅ Review documented

---

## Related Documents

- [Contribution Guidelines](../CONTRIBUTING_AI.md) - Documentation principles
- [Change Impact Matrix](../architecture/change-impact-matrix.md) - Change propagation
- [DDD Principles](../architecture/README.md) - Architecture principles

---

**Playbook Version**: 1.0  
**Last Updated**: 2026-07-05  
**Maintained By**: AI Technical Lead
