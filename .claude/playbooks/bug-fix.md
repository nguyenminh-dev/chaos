# Bug Fix Playbook

**Purpose**: Systematic workflow for identifying, analyzing, and fixing bugs while maintaining architectural integrity and documentation quality.

**When to Use**: Whenever fixing defects, errors, or unexpected behavior in the Billing Service.

---

## Phase 1: Bug Analysis

### Step 1.1: Understand the Bug

**Load these files first**:
1. `CONTRIBUTING_AI.md` - Quick reference for structure
2. `README.md` - Service context
3. **Bug report** - Understand the issue

**Key Questions**:
- What is the expected behavior?
- What is the actual behavior?
- What are the steps to reproduce?
- What is the impact/severity?

---

### Step 1.2: Identify Bug Type

**Bug Type Matrix**:

| Bug Type | Location | Documentation Impact |
|----------|----------|---------------------|
| **Business Rule Violation** | Domain business rules | Update business-rules.md |
| **Aggregate Logic Error** | Domain aggregate | Update aggregate.md |
| **Use Case Flow Error** | Application workflow | Update use case .md |
| **API Contract Error** | API implementation | Update api/index.md |
| **Infrastructure Error** | Database/external services | Update infrastructure/ |
| **Integration Error** | External service calls | Update infrastructure/ |
| **Event Handling Error** | Event subscribers | Update use cases or policies |

**Expected Outcome**: Clear understanding of bug location and type.

---

### Step 1.3: Locate Affected Components

**Load relevant documentation** based on bug type:

**For Business Rule Bugs**:
```bash
# Load domain documentation
domains/{affected-domain}/aggregate.md
domains/{affected-domain}/business-rules.md
domains/{affected-domain}/model.md
```

**For Use Case Bugs**:
```bash
# Load use case documentation
application/use-cases/{affected-use-case}.md

# Load referenced domains
domains/{referenced-domains}/aggregate.md
domains/{referenced-domains}/business-rules.md
```

**For API Bugs**:
```bash
# Load API documentation
api/index.md

# Load referenced domain concepts
domains/{affected-domain}/aggregate.md
domains/{affected-domain}/business-rules.md
```

**Expected Outcome**: Complete understanding of affected components.

---

## Phase 2: Root Cause Analysis

### Step 2.1: Trace the Bug

**For Business Rule Violations**:
- Check if business rule exists in `business-rules.md`
- Check if rule has formal definition
- Check if enforcement points are documented
- Verify rule is actually enforced in code

**For Use Case Flow Errors**:
- Check if use case workflow is documented correctly
- Verify domain references are correct
- Check if alternative flows cover the bug scenario
- Verify failure flows are documented

**For API Contract Errors**:
- Check if API documentation matches implementation
- Verify domain concepts are referenced correctly
- Check error codes are documented
- Validate request/response schemas

---

### Step 2.2: Identify Root Cause

**Common Root Causes**:

1. **Missing Business Rule**: Rule not documented or not enforced
2. **Incomplete Business Rule**: Rule doesn't cover edge case
3. **Incorrect Enforcement**: Rule not enforced at right point
4. **Missing Domain Reference**: Use case doesn't reference domain concept
5. **Incomplete Workflow**: Use case doesn't handle edge case
6. **API-Documentation Mismatch**: API doc doesn't match implementation
7. **Event Handling Gap**: Event subscriber not documented
8. **Infrastructure Issue**: External service behavior not documented

---

### Step 2.3: Determine Fix Scope

**Fix Scope Decision Matrix**:

```
If "Business Rule Missing" → Go to Phase 3A: Add Business Rule
If "Business Rule Incomplete" → Go to Phase 3B: Update Business Rule
If "Use Case Flow Error" → Go to Phase 3C: Fix Use Case Flow
If "API Contract Error" → Go to Phase 3D: Fix API Documentation
If "Domain Reference Missing" → Go to Phase 3E: Add Domain Reference
If "Infrastructure Issue" → Go to Phase 3F: Update Infrastructure
```

---

## Phase 3: Implementation

### Phase 3A: Add Missing Business Rule

**When**: Business rule not documented.

**Step 1**: Add to `domains/{domain}/business-rules.md`
```markdown
### BR-{LETTER}-{NUMBER}: {Rule Name}
**Rule**: {Human-readable description}

**Formal Definition**: {Mathematical/logical definition}

**Enforcement Points**: {Where enforced}

**Violation Handling**: {What happens on violation}

**Purpose**: {Why this rule exists}
```

**Step 2**: Update code to enforce rule

**Step 3**: Add tests for rule violation

**Expected Outcome**: Business rule documented and enforced.

---

### Phase 3B: Update Incomplete Business Rule

**When**: Business rule exists but doesn't cover edge case.

**Step 1**: Update existing rule in `business-rules.md`
- Add edge case to rule definition
- Update formal definition
- Update enforcement points if needed

**Step 2**: Update code enforcement

**Step 3**: Add tests for new edge case

**Expected Outcome**: Business rule updated to cover edge case.

---

### Phase 3C: Fix Use Case Flow

**When**: Use case workflow doesn't handle scenario.

**Step 1**: Update `application/use-cases/{use-case}.md`
- Add to Alternative Flow (if edge case)
- Add to Failure Flow (if error scenario)
- Update Main Flow (if missing step)

**Step 2**: Verify domain references are correct
- Check that aggregates are referenced
- Check that business rules are referenced
- Don't duplicate domain knowledge

**Step 3**: Update implementation to match documentation

**Expected Outcome**: Use case flow covers bug scenario.

---

### Phase 3D: Fix API Documentation

**When**: API documentation doesn't match implementation.

**Step 1**: Update `api/index.md`
- Fix endpoint path
- Fix request/response schemas
- Update error codes
- Add missing domain references

**Step 2**: Update `api/README.md` catalog if needed

**Step 3**: Verify implementation matches documentation

**Expected Outcome**: API documentation accurate.

---

### Phase 3E: Add Missing Domain Reference

**When**: Use case or API doesn't reference domain concept.

**Step 1**: Add reference to use case or API doc
```markdown
## Domain References

### Aggregates Involved
- [{Aggregate}](../domains/{domain}/aggregate.md)
```

**Step 2**: Verify link works

**Expected Outcome**: Missing domain reference added.

---

### Phase 3F: Update Infrastructure Documentation

**When**: Infrastructure issue (database, external service, etc.).

**Step 1**: Update appropriate infrastructure file
- `infrastructure/database.md` for database issues
- External service documentation for integration issues

**Step 2**: Document behavior and constraints

**Expected Outcome**: Infrastructure issue documented.

---

## Phase 4: Testing

### Step 4.1: Reproduce Bug

**Create test** that reproduces the bug:
- Test should fail before fix
- Test should pass after fix

---

### Step 4.2: Verify Fix

**Test the fix**:
- Verify bug is resolved
- Verify no regressions introduced
- Test edge cases
- Test related functionality

---

### Step 4.3: Regression Test

**Run existing tests**:
- All existing tests should still pass
- No new bugs introduced

---

## Phase 5: Documentation Updates

### Step 5.1: Update Change Documentation

**If business rule added/modified**:
- Update `architecture/change-impact-matrix.md` if needed

**If new edge case discovered**:
- Consider updating `playbooks/bug-fix.md` with pattern

---

### Step 5.2: Validate Documentation

**Verify**:
- [ ] Bug fix documented
- [ ] No duplicated knowledge introduced
- [ ] Domain references correct
- [ ] Links resolve correctly
- [ ] DDD compliance maintained

---

## Common Bug Patterns

### Pattern 1: Missing Business Rule

**Symptoms**: Behavior undefined or inconsistent

**Solution**: Add business rule to appropriate domain

**Example**:
- Bug: Wallet can go negative in certain conditions
- Fix: Add BR-W-006: "Balance cannot be negative in any state"

---

### Pattern 2: Incomplete Business Rule

**Symptoms**: Rule exists but doesn't cover edge case

**Solution**: Update business rule to cover edge case

**Example**:
- Bug: BR-W-002 doesn't handle frozen wallets
- Fix: Update BR-W-002: "Balance cannot be negative unless wallet is frozen"

---

### Pattern 3: Missing Domain Reference

**Symptoms**: Use case or API implements domain logic but doesn't reference it

**Solution**: Add domain reference

**Example**:
- Bug: Use case checks balance but doesn't reference BR-W-002
- Fix: Add reference to Wallet business rules

---

### Pattern 4: API-Documentation Mismatch

**Symptoms**: API behaves differently than documented

**Solution**: Fix documentation or implementation to match

**Example**:
- Bug: API returns 403 but docs say 404
- Fix: Update API documentation to match implementation

---

### Pattern 5: Event Handling Gap

**Symptoms**: Event published but no handler documented

**Solution**: Document event subscriber in use case or policy

**Example**:
- Bug: PaymentSucceeded event published but invoice not created
- Fix: Document policy that subscribes to PaymentSucceeded

---

## Quick Reference

### "Bug in X, where do I look?"

| Bug Location | Documentation Files | Fix Action |
|--------------|-------------------|------------|
| **Business logic** | `domains/{domain}/business-rules.md` | Add/update rule |
| **Aggregate behavior** | `domains/{domain}/aggregate.md` | Fix operation |
| **Workflow** | `application/use-cases/{use-case}.md` | Fix flow |
| **API behavior** | `api/index.md` | Fix documentation |
| **External service** | `infrastructure/` | Document integration |
| **Event handling** | `events/README.md` + use cases | Document subscriber |

---

## Validation Checklist

### Before Marking Bug Fixed

- [ ] Root cause identified
- [ ] Fix implemented
- [ ] Test reproduces bug (fails before fix)
- [ ] Test verifies fix (passes after fix)
- [ ] Documentation updated
- [ ] No regressions introduced
- [ ] Single source of truth maintained
- [ ] DDD compliance verified

---

## Success Criteria

### Bug Fix Complete When:

- ✅ Bug reproduced and understood
- ✅ Root cause identified
- ✅ Fix implemented
- ✅ Tests passing
- ✅ Documentation updated
- ✅ No regressions
- ✅ Architecture integrity maintained

---

## Related Documents

- [Contribution Guidelines](../CONTRIBUTING_AI.md) - Documentation principles
- [Change Impact Matrix](../architecture/change-impact-matrix.md) - Change propagation
- [Navigation Guide](../architecture/navigation.md) - Which docs to load

---

**Playbook Version**: 1.0  
**Last Updated**: 2026-07-05  
**Maintained By**: AI Technical Lead
