# Documentation Update Playbook

**Purpose**: Systematic workflow for updating documentation when code changes, ensuring Single Source of Truth is maintained.

**When to Use**: Whenever code changes require documentation updates.

---

## Phase 1: Change Identification

### Step 1.1: Understand the Code Change

**Load these files first**:
1. `CONTRIBUTING_AI.md` - Documentation principles
2. `architecture/change-impact-matrix.md` - Expected impact

**Analyze the code change**:
- Which domain concepts changed?
- Which business rules changed?
- Which aggregates changed?
- Which use cases changed?
- Which APIs changed?

---

### Step 1.2: Determine Documentation Impact

**Use this matrix** to identify affected documentation:

| Code Change | Documentation to Update | Auto-Update? |
|-------------|----------------------|--------------|
| **Entity attribute added** | `domains/{domain}/model.md`, `business-rules.md` (if validation needed) | Use cases, API docs (via refs) |
| **Business rule added** | `domains/{domain}/business-rules.md` | Use cases, API docs (via refs) |
| **Business rule modified** | `domains/{domain}/business-rules.md` | Use cases, API docs (via refs) |
| **Aggregate operation added** | `domains/{domain}/aggregate.md` | Use cases (via refs) |
| **Domain event added** | `domains/{domain}/domain-events.md`, `events/README.md` | Use cases (via refs) |
| **API contract changed** | `api/index.md` | No domain files |
| **Use case flow changed** | `application/use-cases/{use-case}.md` | No domain files |
| **Policy changed** | `policies/{policy}.md` | Affected use cases |

**Expected Outcome**: List of documentation files that need updates.

---

## Phase 2: Documentation Updates

### Phase 2A: Update Domain Documentation

**When**: Domain concepts, business rules, or aggregates changed.

**Step 1**: Update `domains/{domain}/model.md`
- Add new entity attributes
- Add new value objects
- Update entity relationships

**Step 2**: Update `domains/{domain}/business-rules.md` (if needed)
- Add new business rules
- Modify existing business rules
- Ensure formal definitions present
- Document enforcement points

**Step 3**: Update `domains/{domain}/aggregate.md` (if needed)
- Add new operations
- Update aggregate root
- Update entities

**Step 4**: Update `domains/{domain}/domain-events.md` (if needed)
- Add new events
- Update event payloads
- Document event triggers

**Expected Outcome**: Domain documentation updated, SSOT maintained.

---

### Phase 2B: Update Use Case Documentation

**When**: Application workflows changed.

**Step 1**: Update `application/use-cases/{use-case}.md`
- Update main flow
- Add alternative flows
- Add failure flows
- **DO NOT** update domain references (they should already be correct)
- **DO NOT** duplicate business rules

**Step 2**: Verify domain references still correct
- Check that aggregate links still valid
- Check that business rule links still valid
- Add new policy references if cross-aggregate logic added

**Expected Outcome**: Use case documentation updated, no duplication introduced.

---

### Phase 2C: Update API Documentation

**When**: API contracts changed.

**Step 1**: Update `api/index.md`
- Update endpoint path
- Update request/response schemas
- Update error codes
- **DO NOT** duplicate business rules
- **DO NOT** duplicate domain concepts
- Reference domain concepts via links

**Step 2**: Update `api/README.md` catalog (if needed)
- Add new API to table
- Update API descriptions

**Expected Outcome**: API documentation updated, domain concepts referenced (not duplicated).

---

### Phase 2D: Update Event Documentation

**When**: Domain events changed.

**Step 1**: Update `domains/{domain}/domain-events.md`
- Add new event definitions
- Update event payloads
- Document event triggers

**Step 2**: Update `events/README.md` catalog
- Add new event to catalog
- Update event descriptions
- Document event consumers

**Expected Outcome**: Event documentation updated, catalog current.

---

### Phase 2E: Update Policy Documentation

**When**: Cross-aggregate business logic changed.

**Step 1**: Update `policies/{policy}.md`
- Update policy flow
- Update failure handling
- Update participating aggregates

**Step 2**: Update `policies/README.md` catalog (if needed)
- Add policy to catalog
- Update policy descriptions

**Expected Outcome**: Policy documentation updated, catalog current.

---

## Phase 3: Cross-Reference Updates

### Step 3.1: Update Navigation Aids

**If new domain added**:
- Update `domains/README.md` domain list
- Update `README.md` domain model section

**If new policy added**:
- Update `policies/README.md` policy catalog

**If new use case added**:
- Update `application/README.md` use case list

**If new API added**:
- Update `api/README.md` API catalog

**If new event added**:
- Update `events/README.md` event catalog

---

### Step 3.2: Update Change Impact Matrix

**If structural changes made**:
- Review `architecture/change-impact-matrix.md`
- Add new scenarios if applicable
- Update dependency graph if needed

---

### Step 3.3: Update Glossary (if new terms)

**If new domain terms introduced**:
- Add to `reference/glossary.md`
- Document definition and usage

---

## Phase 4: Validation

### Step 4.1: Single Source of Truth Validation

**Verify NO duplication**:
```bash
# Search for business rule - should appear ONLY in business-rules.md
grep -r "BR-W-002" .claude/billing-service/domains/

# Search for aggregate name - should appear ONLY in aggregate.md (except references)
grep -r "Wallet Aggregate" .claude/billing-service/domains/
```

**Expected Result**: Each concept appears in EXACTLY ONE location (plus references).

**If duplication found**:
- Remove duplicated content
- Replace with reference to authoritative source

---

### Step 4.2: Link Validation

**Test all links**:
- All relative links resolve correctly
- No broken image links
- Cross-domain references use correct relative paths
- All referenced documents exist

**If broken links found**:
- Fix link paths
- Remove references to deleted files
- Update moved file references

---

### Step 4.3: DDD Compliance Validation

**Verify**:
- [ ] Business rules ONLY in Domain layer
- [ ] Use Cases reference Aggregates (not duplicate)
- [ ] Use Cases reference Business Rules (not duplicate)
- [ ] API docs reference domain concepts (not duplicate)
- [ ] No cross-aggregate logic inside any Aggregate
- [ ] Cross-aggregate logic in Policies

**If violations found**:
- Remove duplicated business knowledge
- Add proper references to domain documents
- Move cross-aggregate logic to policies

---

### Step 4.4: Documentation Completeness Validation

**Verify**:
- [ ] All referenced documents exist
- [ ] All domain files have required sections
- [ ] All use cases have "Domain References" section
- [ ] All policies document Trigger, Flow, Failure Handling
- [ ] All business rules have formal definitions
- [ ] All APIs reference domain concepts

**If incompleteness found**:
- Add missing sections
- Add missing references
- Complete business rule definitions

---

## Phase 5: Consistency Check

### Step 5.1: Code-Documentation Alignment

**Verify**: Documentation matches implementation

**Check**:
- Business rules documented are actually enforced?
- Aggregate operations documented are actually implemented?
- Domain events documented are actually published?
- API behavior matches documentation?

**If misalignment found**:
- Update documentation to match implementation, OR
- Update implementation to match documentation (depending on intent)

---

### Step 5.2: Terminology Consistency

**Verify**: Consistent terminology across documents

**Check**:
- Domain concepts use consistent names?
- Business rules use consistent format?
- Events use consistent naming?
- APIs use consistent naming?

**If inconsistency found**:
- Standardize terminology
- Update all occurrences
- Add to glossary if new term

---

## Common Documentation Update Scenarios

### Scenario 1: Added New Entity Attribute

**Documentation Updates**:
1. Update `domains/{domain}/model.md` - Add attribute to entity
2. Update `domains/{domain}/business-rules.md` - Add validation rule if needed
3. **Do NOT update** use cases (they auto-update via references)
4. **Do NOT update** API docs (they auto-update via references)

**Validation**:
- Attribute documented ONLY in model.md
- Validation rule ONLY in business-rules.md (if needed)
- Use cases still reference domain correctly
- API docs still reference domain correctly

---

### Scenario 2: Added New Business Rule

**Documentation Updates**:
1. Update `domains/{domain}/business-rules.md` - Add new business rule
2. **Do NOT update** use cases (they auto-update via references)
3. **Do NOT update** API docs (they auto-update via references)

**Validation**:
- Business rule documented ONLY in business-rules.md
- No duplication in use cases
- No duplication in API docs
- Business rule has formal definition

---

### Scenario 3: Modified Use Case Flow

**Documentation Updates**:
1. Update `application/use-cases/{use-case}.md` - Update flow
2. **Do NOT update** domain files (no business logic change)
3. **Do NOT update** API docs (no API change)

**Validation**:
- Domain references still correct
- No business rules duplicated
- Use case still references domain concepts

---

### Scenario 4: Modified API Contract

**Documentation Updates**:
1. Update `api/index.md` - Update API contract
2. Update `api/README.md` catalog if needed
3. **Do NOT update** domain files (no business logic change)
4. **Do NOT update** use cases (no workflow change)

**Validation**:
- API docs reference domain concepts (not duplicate)
- No business rules in API docs
- Domain concepts referenced via links

---

### Scenario 5: Added New Domain Event

**Documentation Updates**:
1. Update `domains/{domain}/domain-events.md` - Add event definition
2. Update `events/README.md` catalog - Add event to catalog
3. **Do NOT update** use cases (they auto-update via references)

**Validation**:
- Event documented ONLY in domain-events.md
- Event catalog references domain-events.md
- No duplication across documents

---

## Quick Reference

### "Code changed X, which docs to update?"

| Code Change | Documentation Updates | Auto-Update? |
|-------------|---------------------|--------------|
| **Entity attribute** | `model.md`, `business-rules.md` (if validation) | Use cases, API docs (via refs) |
| **Business rule** | `business-rules.md` | Use cases, API docs (via refs) |
| **Aggregate operation** | `aggregate.md` | Use cases (via refs) |
| **Domain event** | `domain-events.md`, `events/README.md` | Use cases (via refs) |
| **Use case flow** | `application/use-cases/{use-case}.md` | No domain files |
| **API contract** | `api/index.md`, `api/README.md` | No domain files |
| **Policy logic** | `policies/{policy}.md` | Affected use cases |

---

### "How to validate documentation update?"

| Validation Type | Command/Check | Expected Result |
|----------------|----------------|-----------------|
| **No duplication** | `grep -r "BR-W-002" domains/` | 1 result only |
| **Links work** | Click all links in markdown preview | All resolve |
| **DDD compliance** | Manual review of files | Business rules only in domain |
| **Completeness** | Check required sections | All present |

---

## Documentation Update Checklist

### Before Marking Update Complete

**Domain Updates**:
- [ ] Entity attributes added to model.md
- [ ] Business rules added/updated in business-rules.md
- [ ] Aggregate operations added to aggregate.md
- [ ] Domain events added to domain-events.md
- [ ] Event catalog updated if needed

**Application Updates**:
- [ ] Use case flows updated
- [ ] Domain references still correct
- [ ] No business rules duplicated
- [ ] No domain concepts duplicated

**API Updates**:
- [ ] API contracts updated in api/index.md
- [ ] API catalog updated if needed
- [ ] Domain concepts referenced (not duplicated)
- [ ] Business rules referenced (not duplicated)

**Cross-References**:
- [ ] Navigation aids updated
- [ ] Change impact matrix updated if needed
- [ ] Glossary updated if new terms

**Validation**:
- [ ] No duplicated knowledge
- [ ] All links resolve
- [ ] DDD compliance verified
- [ ] Documentation complete
- [ ] Code-documentation aligned

---

## Common Pitfalls to Avoid

### ❌ Pitfall 1: Updating Files That Auto-Update

**Wrong**: Updating use cases when business rule changes

**Correct**: Use cases auto-update via references, no changes needed

---

### ❌ Pitfall 2: Duplicating Instead of Referencing

**Wrong**: Adding business rules to use cases or API docs

**Correct**: Reference domain business rules via links

---

### ❌ Pitfall 3: Forgetting to Update Catalogs

**Wrong**: Adding domain event but forgetting events/README.md catalog

**Correct**: Update both domain-events.md and events/README.md

---

### ❌ Pitfall 4: Breaking Links

**Wrong**: Moving files without updating references

**Correct**: Update all references when moving files

---

### ❌ Pitfall 5: Incomplete Updates

**Wrong**: Adding entity attribute but not adding validation rule

**Correct**: Add both attribute and any validation rules

---

## Success Criteria

### Documentation Update Complete When:

- ✅ All affected documentation files updated
- ✅ No duplicated knowledge introduced
- ✅ All references work correctly
- ✅ DDD compliance maintained
- ✅ Documentation complete
- ✅ Code-documentation aligned
- ✅ All validations pass

---

## Related Documents

- [Contribution Guidelines](../CONTRIBUTING_AI.md) - Documentation principles
- [Change Impact Matrix](../architecture/change-impact-matrix.md) - Change propagation
- [Navigation Guide](../architecture/navigation.md) - Which docs to load

---

**Playbook Version**: 1.0  
**Last Updated**: 2026-07-05  
**Maintained By**: AI Technical Lead
