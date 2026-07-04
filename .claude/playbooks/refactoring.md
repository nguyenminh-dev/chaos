# Refactoring Playbook

**Purpose**: Systematic workflow for refactoring code while maintaining DDD principles, architectural integrity, and documentation quality.

**When to Use**: When improving code structure, optimizing performance, or enhancing maintainability.

---

## Phase 1: Refactoring Analysis

### Step 1.1: Understand Current State

**Load these files first**:
1. `CONTRIBUTING_AI.md` - Architecture principles
2. `README.md` - Service context
3. `architecture/bounded-context.md` - Context boundaries
4. Domain documentation for affected areas

**Analyze current implementation**:
- What is the current structure?
- What are the pain points?
- What violates DDD principles?
- What creates maintenance burden?

---

### Step 1.2: Identify Refactoring Scope

**Refactoring Types**:

| Refactoring Type | Scope | Documentation Impact |
|-----------------|-------|---------------------|
| **Internal Domain Refactor** | Single domain internals | Domain files only |
| **Cross-Domain Refactor** | Multiple domains | Domain + policies |
| **Application Refactor** | Use case workflows | Application files only |
| **Infrastructure Refactor** | External services | Infrastructure files only |
| **API Refactor** | API contracts | API files only |
| **Aggregate Split** | Divide aggregate | Domain + use cases |
| **Aggregate Merge** | Combine aggregates | Domain + use cases |

**Expected Outcome**: Clear understanding of refactoring scope and impact.

---

### Step 1.3: Risk Assessment

**Assess refactoring risks**:

| Risk Factor | Assessment | Mitigation |
|-------------|------------|------------|
| **Breaking changes** | High/Medium/Low | Versioning, migration guide |
| **Domain boundary changes** | High/Medium/Low | Careful analysis, phased rollout |
| **Business rule changes** | High/Medium/Low | Preserve rules, update enforcement |
| **API contract changes** | High/Medium/Low | Versioning, backward compatibility |
| **Event schema changes** | High/Medium/Low | Versioning, event evolution |
| **Data migration needed** | High/Medium/Low | Migration scripts, rollback plan |

**Expected Outcome**: Risk profile and mitigation strategy.

---

## Phase 2: Refactoring Design

### Phase 2A: Internal Domain Refactoring

**When**: Refactoring within a single domain boundary.

**Step 1**: Analyze current domain structure
- Load `domains/{domain}/aggregate.md`
- Load `domains/{domain}/model.md`
- Load `domains/{domain}/business-rules.md`
- Identify refactoring opportunities

**Common refactorings**:
- Extract value object
- Combine related entities
- Split large entity
- Rename concepts for clarity
- Reorganize business rules

**Step 2**: Design new structure
- Preserved business rules (no changes)
- Preserved domain events (no changes)
- Improved internal organization
- Better encapsulation

**Step 3**: Verify impact
- No impact on other domains
- No impact on use cases (references stay valid)
- No impact on APIs (domain concepts unchanged)
- No impact on policies (domain boundaries unchanged)

**Expected Outcome**: Improved internal structure, zero external impact.

---

### Phase 2B: Aggregate Boundary Refactoring

**When**: Splitting or merging aggregates.

**Step 1**: Analyze current aggregates
- Load `domains/{domain}/aggregate.md` for affected aggregates
- Identify aggregate boundaries
- Analyze consistency boundaries
- Identify cross-aggregate transactions

**Step 2**: Design new aggregates
- Define new aggregate roots
- Define new entities
- Define new business rules
- Define new domain events

**Step 3**: Verify consistency
- Each aggregate has single consistency boundary
- No cross-aggregate transactions
- Eventual consistency between aggregates
- Business rules preserved

**Expected Outcome**: Improved aggregate boundaries, better consistency.

---

### Phase 2C: Cross-Domain Refactoring

**When**: Refactoring spans multiple domains.

**Step 1**: Analyze all affected domains
- Load all affected domain documentation
- Identify shared concepts
- Identify duplication opportunities
- Identify policy opportunities

**Step 2**: Design refactoring
- Extract common concepts to shared kernel if needed
- Move concepts to appropriate domains
- Create policies for cross-aggregate coordination
- Update domain boundaries

**Step 3**: Verify DDD compliance
- No business rules duplicated
- Single source of truth maintained
- Aggregate boundaries preserved
- Policies used for coordination

**Expected Outcome**: Eliminated duplication, clearer boundaries.

---

### Phase 2D: Application Layer Refactoring

**When**: Refactoring use case workflows.

**Step 1**: Analyze current use cases
- Load `application/use-cases/{use-case}.md`
- Identify workflow improvements
- Identify orchestration opportunities

**Step 2**: Design new workflows
- Simplify complex flows
- Extract common patterns
- Improve error handling
- Enhance validation

**Step 3**: Verify domain independence
- No business logic in use cases
- Domain knowledge referenced, not duplicated
- Aggregate boundaries maintained

**Expected Outcome**: Improved application workflows, cleaner separation.

---

### Phase 2E: Infrastructure Refactoring

**When**: Refactoring external service integration, database, etc.

**Step 1**: Analyze current infrastructure
- Load `infrastructure/README.md`
- Load relevant infrastructure files
- Identify improvement opportunities

**Step 2**: Design refactoring
- Improve external service abstraction
- Enhance caching strategy
- Optimize database queries
- Improve error handling

**Step 3**: Verify domain independence
- Domain layer unchanged
- Infrastructure implements domain interfaces
- No domain logic in infrastructure

**Expected Outcome**: Improved infrastructure, domain protected.

---

## Phase 3: Documentation Updates

### Step 3.1: Update Domain Documentation

**For internal domain refactoring**:
- Update `domains/{domain}/model.md` - New structure
- Update `domains/{domain}/aggregate.md` - New operations
- **Do NOT update** business rules (no changes)
- **Do NOT update** domain events (no changes)

**For aggregate boundary refactoring**:
- Create new domain directories if splitting
- Update all affected domain files
- Update business rules if moving
- Update domain events if moving
- Update `domains/README.md` domain list

**For cross-domain refactoring**:
- Update all affected domain files
- Ensure no duplicated knowledge
- Update `domains/README.md` if needed

---

### Step 3.2: Update Application Documentation

**For use case refactoring**:
- Update `application/use-cases/{use-case}.md` - New workflow
- **Do NOT update** domain files (no business logic change)
- Verify domain references still correct

**For aggregate refactoring**:
- Update affected use cases
- Update domain references
- Add new policy references if needed

---

### Step 3.3: Update API Documentation

**For API refactoring**:
- Update `api/index.md` - New contracts
- Update `api/README.md` catalog if needed
- **Do NOT duplicate** domain concepts
- Reference domain concepts via links

**For aggregate refactoring**:
- Update affected APIs if aggregate concepts changed
- Maintain backward compatibility if possible

---

### Step 3.4: Update Policy Documentation

**For cross-domain refactoring**:
- Update `policies/{policy}.md` if policies affected
- Create new policies if new cross-aggregate coordination needed
- Update `policies/README.md` catalog

---

## Phase 4: Implementation

### Step 4.1: Implement Refactoring

**Follow this sequence**:
1. Update tests to match new structure
2. Implement new structure (green field)
3. Migrate existing data/code
4. Update documentation
5. Verify all tests pass

**Key Principles**:
- Preserve business rules
- Preserve domain events (no breaking changes)
- Maintain aggregate consistency
- Keep use cases working

---

### Step 4.2: Data Migration (if needed)

**If data structure changed**:
- Design migration script
- Test migration on sample data
- Plan rollback strategy
- Execute migration during maintenance window
- Verify migration success

**Expected Outcome**: Data migrated successfully, no data loss.

---

## Phase 5: Validation

### Step 5.1: Business Rule Preservation Check

**Verify**: All business rules still enforced

**Check**:
- All business rules from before still present?
- All business rules still enforced?
- No business rule violations?
- Business rule tests passing?

**Expected Result**: Zero business rule regressions.

---

### Step 5.2: Aggregate Consistency Check

**Verify**: Aggregate boundaries maintained

**Check**:
- Each aggregate is consistent boundary?
- No cross-aggregate transactions?
- Eventual consistency working?
- Aggregate operations atomic?

**Expected Result**: Aggregate consistency maintained.

---

### Step 5.3: Documentation Accuracy Check

**Verify**: Documentation matches implementation

**Check**:
- Domain documentation updated?
- Use case documentation updated?
- API documentation updated?
- No duplicated knowledge?
- All links valid?

**Expected Result**: Documentation accurate and complete.

---

### Step 5.4: Backward Compatibility Check

**Verify**: No breaking changes without versioning

**Check**:
- API contracts maintain compatibility?
- Event schemas maintain compatibility?
- Database changes backward compatible?
- Migration guide provided if breaking?

**Expected Result**: Compatibility maintained or properly versioned.

---

## Common Refactoring Patterns

### Pattern 1: Extract Value Object

**When**: Primitive value appears with business meaning

**Example**:
```typescript
// Before
class Wallet {
  balance: number;
  currency: string;
}

// After
class Wallet {
  balance: Balance;  // Value object
}

class Balance {
  amount: number;
  currency: Currency;
}
```

**Documentation Updates**:
- Update `domains/{domain}/model.md` - Add value object
- No business rule changes
- No use case changes

---

### Pattern 2: Split Large Aggregate

**When**: Aggregate has too many responsibilities

**Example**:
```typescript
// Before
Wallet (handles balance, assets, limits, freezes, ...)
// After
Wallet (balance, assets)
WalletLimits (limits, freezes)
```

**Documentation Updates**:
- Create new domain directory for split aggregate
- Update both aggregate files
- Update use cases to reference both aggregates
- Create policy if cross-aggregate coordination needed

---

### Pattern 3: Consolidate Business Rules

**When**: Related rules scattered

**Example**:
```markdown
# Before
### BR-W-002: Balance Check
Balance must be sufficient.

### BR-W-005: Reserve Check
Reserved balance must be sufficient.

### BR-W-008: Available Balance Check
Available balance must be non-negative.

# After
### BR-W-002: Balance Validation
All balance components must be non-negative:
- availableBalance >= 0
- reservedBalance >= 0
- totalBalance >= 0
```

**Documentation Updates**:
- Consolidate in `business-rules.md`
- No use case changes (they auto-update via references)

---

### Pattern 4: Extract Policy

**When**: Cross-aggregate logic in use case

**Example**:
```markdown
# Before
# application/use-cases/process-payment.md
## Main Flow
1. Process payment
2. Create invoice  ❌ Cross-aggregate logic
3. Credit wallet  ❌ Cross-aggregate logic

# After
# policies/payment-completion-policy.md
## Trigger
PaymentSucceeded event

## Flow
1. Create invoice
2. Credit wallet

# application/use-cases/process-payment.md
## Main Flow
1. Process payment
2. Publish PaymentSucceeded event
```

**Documentation Updates**:
- Create `policies/payment-completion-policy.md`
- Update use case to reference policy
- Update `policies/README.md` catalog

---

## Refactoring Checklist

### Before Starting Refactoring

**Analysis**:
- [ ] Current state understood
- [ ] Refactoring scope identified
- [ ] Risk assessment completed
- [ ] Mitigation strategy defined

**Design**:
- [ ] New structure designed
- [ ] Business rules preserved
- [ ] Aggregate boundaries defined
- [ ] DDD compliance verified

**Planning**:
- [ ] Implementation sequence planned
- [ ] Test strategy defined
- [ ] Rollback strategy planned
- [ ] Data migration planned (if needed)

---

### During Refactoring

**Implementation**:
- [ ] Tests updated first
- [ ] New structure implemented
- [ ] Existing code migrated
- [ ] Documentation updated
- [ ] Tests passing

**Validation**:
- [ ] Business rules preserved
- [ ] Aggregate consistency maintained
- [ ] No breaking changes (or properly versioned)
- [ ] Documentation accurate
- [ ] Tests comprehensive

---

### After Refactoring

**Verification**:
- [ ] All tests passing
- [ ] No regressions
- [ ] Performance maintained or improved
- [ ] Documentation complete
- [ ] Team notified of changes

---

## Quick Reference

### "Refactoring X, what's the impact?"

| Refactoring Type | Domain Docs | Use Cases | API Docs | Policies |
|-----------------|------------|-----------|----------|----------|
| **Extract value object** | Update `model.md` | No change | No change | No change |
| **Split aggregate** | Create new domain, update both | Update references | Update if needed | Create if coordination |
| **Merge aggregates** | Merge domains, update | Update references | Update if needed | Remove if no longer needed |
| **Consolidate rules** | Update `business-rules.md` | No change | No change | No change |
| **Extract policy** | No change | Update to reference | No change | Create new policy |
| **Refactor use case** | No change | Update use case | No change | No change |
| **Refactor infrastructure** | No change | No change | No change | No change |

---

## Success Criteria

### Refactoring Complete When:

- ✅ Business rules preserved and enforced
- ✅ Aggregate boundaries maintained
- ✅ Code structure improved
- ✅ Documentation updated and accurate
- ✅ Tests passing
- ✅ No regressions
- ✅ Performance maintained
- ✅ Team aligned on changes

---

## Related Documents

- [Contribution Guidelines](../CONTRIBUTING_AI.md) - Documentation principles
- [Change Impact Matrix](../architecture/change-impact-matrix.md) - Change propagation
- [DDD Principles](../architecture/README.md) - Architecture principles

---

**Playbook Version**: 1.0  
**Last Updated**: 2026-07-05  
**Maintained By**: AI Technical Lead
