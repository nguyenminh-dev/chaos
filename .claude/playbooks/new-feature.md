# New Feature Implementation Playbook

**Purpose**: Step-by-step workflow for implementing new features in the Billing Service while maintaining DDD principles and documentation quality.

**When to Use**: Whenever adding new functionality, features, or capabilities to the Billing Service.

---

## Phase 1: Analysis & Understanding

### Step 1.1: Load Core Documentation

**Load these files first** (in exact order):
1. `CONTRIBUTING_AI.md` - Contribution guidelines and SSOT principles
2. `README.md` - Service overview and context
3. `architecture/bounded-context.md` - Context ownership and boundaries
4. `architecture/context-map.md` - Service relationships

**Expected Outcome**: Understanding of service boundaries, responsibilities, and integration points.

---

### Step 1.2: Identify Feature Scope

**Ask these questions**:
- Is this a **new domain concept**? → May need new Aggregate
- Is this **cross-aggregate logic**? → Needs Policy
- Is this **application workflow**? → Needs Use Case
- Is this **API endpoint**? → Needs API documentation
- Is this **infrastructure change**? → Needs infrastructure docs

**Decision Matrix**:
```
If "New business concept" → Go to Phase 2A: New Domain
If "Existing domain extension" → Go to Phase 2B: Extend Domain
If "Cross-aggregate coordination" → Go to Phase 2C: New Policy
If "Application workflow" → Go to Phase 2D: New Use Case
If "API only" → Go to Phase 2E: New API
```

**Expected Outcome**: Clear understanding of which parts of the system will change.

---

### Step 1.3: Impact Analysis

**Load relevant domain documentation**:
- Load `domains/{affected-domains}/README.md`
- Load `domains/{affected-domains}/aggregate.md`
- Load `domains/{affected-domains}/business-rules.md`
- Load `architecture/change-impact-matrix.md` for guidance

**Identify**:
- Which Aggregates are affected?
- Which Business Rules may change?
- Which Use Cases may change?
- Which APIs may change?
- Which Policies may be affected?

**Expected Outcome**: Complete list of files that may need updates.

---

## Phase 2: Implementation

### Phase 2A: New Domain (New Aggregate)

**When**: Feature introduces completely new business concept not covered by existing domains.

**Step 1: Create Domain Directory Structure**
```bash
mkdir -p domains/{new-domain}/
```

**Step 2: Create Domain Files** (using templates)
1. `overview.md` - Domain purpose and scope
2. `aggregate.md` - Aggregate root and entities
3. `model.md` - Value objects and types
4. `business-rules.md` - Business invariants
5. `lifecycle.md` - State transitions
6. `domain-events.md` - Published events

**Step 3: Define Business Rules**
- Use `BR-{DOMAIN}-{NUMBER}` naming convention
- Start with BR-{DOMAIN}-001
- Document formal definitions
- Specify enforcement points

**Step 4: Define Domain Events**
- Event name: `{Concept}{StateChange}` (e.g., `WalletCreated`)
- Document trigger, payload, consumers
- Add to `events/README.md` catalog

**Step 5: Update Domain Index**
- Add domain to `domains/README.md`

**Expected Outcome**: Complete new domain with 6-7 documentation files.

---

### Phase 2B: Extend Existing Domain

**When**: Feature adds new capability to existing domain.

**Step 1: Load Existing Domain**
```
domains/{existing-domain}/aggregate.md
domains/{existing-domain}/business-rules.md
domains/{existing-domain}/model.md
```

**Step 2: Add New Business Rule** (if needed)
- Add to `domains/{existing-domain}/business-rules.md`
- Use next sequential number: BR-{LETTER}-{XXX}
- Document formal definition
- Specify enforcement points

**Step 3: Update Aggregate** (if structure changes)
- Update `domains/{existing-domain}/aggregate.md`
- Add new operations if needed
- Update entities if needed

**Step 4: Update Model** (if new types)
- Update `domains/{existing-domain}/model.md`
- Add new value objects if needed

**Step 5: Update Domain Events** (if new events)
- Add to `domains/{existing-domain}/domain-events.md`
- Update `events/README.md` catalog

**Expected Outcome**: Domain extended with new capabilities, all related files updated.

---

### Phase 2C: New Policy (Cross-Aggregate Logic)

**When**: Feature requires coordination across multiple aggregates.

**Step 1: Create Policy File**
```bash
touch policies/{new-policy}.md
```

**Step 2: Document Policy**
- Purpose: What this policy coordinates
- Trigger: Event or condition that triggers policy
- Participating Aggregates: List all involved aggregates
- Domain Events: Consumes and publishes
- Flow: Step-by-step coordination
- Failure Handling: Compensation and recovery

**Step 3: Update Policy Catalog**
- Add to `policies/README.md`

**Step 4: Identify Affected Use Cases**
- Find use cases that should reference this policy
- Update their "Domain References" sections

**Expected Outcome**: New policy documenting cross-aggregate coordination.

---

### Phase 2D: New Use Case (Application Workflow)

**When**: Feature introduces new application workflow.

**Step 1: Create Use Case File**
```bash
touch application/use-cases/{new-use-case}.md
```

**Step 2: Document Use Case**
- Business Goal: What this achieves
- Actor: Who triggers this
- Trigger: What starts this
- Preconditions: What must be true
- Domain References: Link to aggregates and business rules
- Main Flow: Step-by-step process
- Alternative Flow: Edge cases
- Failure Flow: Error scenarios
- Postconditions: What changed
- Acceptance Criteria: Verification checklist
- Related APIs: Affected endpoints
- Related Events: Published events

**CRITICAL**: Use Cases must REFERENCE domain knowledge, NEVER duplicate it.

**Step 3: Update Application Index**
- Add to `application/README.md`

**Expected Outcome**: New use case with proper domain references.

---

### Phase 2E: New API Endpoint

**When**: Feature adds new API endpoint.

**Step 1: Document API**
- Add to `api/index.md`
- Document endpoint, purpose, request/response
- Reference domain concepts (not duplicate)
- Reference business rules (not duplicate)
- Reference events (not duplicate)

**Step 2: Update API Catalog**
- Add to `api/README.md` table of contents

**Expected Outcome**: New API documented with proper domain references.

---

## Phase 3: Documentation Updates

### Step 3.1: Update Navigation

**If new domain added**:
- Update `domains/README.md` domain list
- Update `README.md` domain model section

**If new policy added**:
- Update `policies/README.md` policy catalog

**If new use case added**:
- Update `application/README.md` use case list

**If new API added**:
- Update `api/README.md` API catalog

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
# Search for business rule name - should appear ONLY in business-rules.md
grep -r "BR-W-002" .claude/billing-service/domains/

# Search for Aggregate name - should appear ONLY in aggregate.md (except references)
grep -r "Wallet Aggregate" .claude/billing-service/domains/
```

**Expected Result**: Each concept appears in EXACTLY ONE location (plus references).

---

### Step 4.2: Link Validation

**Test all links**:
- All relative links resolve correctly
- No broken image links
- Cross-domain references use correct relative paths
- All referenced documents exist

---

### Step 4.3: DDD Compliance Validation

**Verify**:
- [ ] Business rules ONLY in Domain layer
- [ ] Use Cases reference Aggregates (not duplicate)
- [ ] Use Cases reference Business Rules (not duplicate)
- [ ] API docs reference domain concepts (not duplicate)
- [ ] No cross-aggregate logic inside any Aggregate
- [ ] Cross-aggregate logic in Policies

---

### Step 4.4: Documentation Completeness Validation

**Verify**:
- [ ] All referenced documents exist
- [ ] All domain files have required sections
- [ ] All use cases have "Domain References" section
- [ ] All policies document Trigger, Flow, Failure Handling
- [ ] All business rules have formal definitions

---

## Phase 5: Testing

### Step 5.1: Acceptance Test

**Create test** for the new feature:
- Test happy path
- Test validation errors
- Test edge cases
- Test business rule violations

---

### Step 5.2: Integration Test

**Create test** for integrations:
- Test external service calls (if any)
- Test event publishing
- Test event handling
- Test database operations

---

### Step 5.3: Regression Test

**Run existing tests**:
- Verify no existing functionality broken
- Verify existing business rules still enforced
- Verify existing use cases still work

---

## Common Pitfalls to Avoid

### ❌ Pitfall 1: Duplicating Business Rules

**Wrong**: Adding business rules to Use Cases or API docs
```markdown
# application/use-cases/consume-credit.md
## Business Rules
- Balance must be sufficient  ❌ WRONG
```

**Correct**: Reference business rules from Use Cases
```markdown
# application/use-cases/consume-credit.md
## Domain References
### Business Rules Enforced
- [Wallet business rules](../domains/wallet/business-rules.md)
```

---

### ❌ Pitfall 2: Defining Aggregates in Multiple Places

**Wrong**: Defining Aggregate in both domain docs and feature docs

**Correct**: Define Aggregate ONCE in `domains/{domain}/aggregate.md`

---

### ❌ Pitfall 3: Cross-Aggregate Logic Inside Aggregate

**Wrong**: Adding cross-aggregate logic to domain business rules
```markdown
# domains/payment/business-rules.md
### BR-P-006: Invoice After Payment
Invoice must be created after payment succeeds  ❌ WRONG
```

**Correct**: Create policy for cross-aggregate coordination
```markdown
# policies/invoice-after-payment.md
## Trigger
PaymentSucceeded event from Payment Aggregate
```

---

### ❌ Pitfall 4: Skipping Documentation Updates

**Wrong**: Implementing feature without updating documentation

**Correct**: Update documentation AS PART OF implementation

---

### ❌ Pitfall 5: Not Testing Business Rules

**Wrong**: Only testing happy path

**Correct**: Test business rule violations, edge cases, and failures

---

## Quick Reference

### "I want to add X, where does it go?"

| What to Add | Location | Template |
|-------------|----------|----------|
| **New Aggregate** | `domains/{aggregate}/` (6-7 files) | `templates/new-domain/` |
| **New Business Rule** | `domains/{aggregate}/business-rules.md` | `templates/business-rule.md` |
| **New Entity/Value Object** | `domains/{aggregate}/model.md` | N/A (add to existing) |
| **New Domain Event** | `domains/{aggregate}/domain-events.md` | `templates/domain-event.md` |
| **New Use Case** | `application/use-cases/{use-case}.md` | `templates/use-case.md` |
| **New API** | `api/index.md` | `templates/api.md` |
| **New Cross-Aggregate Logic** | `policies/{policy}.md` | `templates/policy.md` |

---

### "I changed X, what else needs updating?"

| Changed | What to Update | Auto-Update? |
|---------|---------------|--------------|
| **Entity attribute** | business-rules.md (may need validation) | Use Cases, API docs (via refs) |
| **Business rule** | domains/{domain}/business-rules.md | Use Cases, API docs (via refs) |
| **Aggregate operation** | domains/{domain}/aggregate.md | Use Cases (via refs) |
| **Domain event** | domains/{domain}/domain-events.md, events/README.md | Use Cases (via refs) |
| **API contract** | api/index.md | No domain files (API owns contract) |
| **Use Case flow** | application/use-cases/{use-case}.md | No domain files (Use Case owns flow) |
| **Policy logic** | policies/{policy}.md | Use Cases that reference policy |

---

## Examples

### Example 1: Adding "Wallet Freeze" Feature

**Analysis**:
- Existing domain: Wallet
- New business rule: BR-W-006 (Frozen wallet validation)
- New lifecycle state: FROZEN
- New API: POST /api/v1/wallets/{id}/freeze

**Implementation Sequence**:
1. Update `domains/wallet/model.md` - Add isFrozen attribute
2. Update `domains/wallet/lifecycle.md` - Add FROZEN state
3. Update `domains/wallet/business-rules.md` - Add BR-W-006
4. Update `domains/wallet/aggregate.md` - Add freeze/thaw operations
5. Add to `api/index.md` - Document freeze API

**Files Updated**: 4 domain files + 1 API file

**Validation**:
- [ ] BR-W-006 appears ONLY in business-rules.md
- [ ] FROZEN state appears ONLY in lifecycle.md
- [ ] Use Cases auto-update (no changes needed)

---

### Example 2: Adding "Multi-Wallet Transfer" Feature

**Analysis**:
- Cross-aggregate coordination
- Multiple Wallet aggregates involved
- CreditTransaction tracking needed
- Ledger accounting needed

**Implementation Sequence**:
1. Create `policies/multi-wallet-transfer.md`
2. Document policy: Trigger, Flow, Failure Handling
3. Update `policies/README.md` - Add to catalog
4. Create `application/use-cases/transfer-wallet.md` (if new workflow)

**Files Created**: 1 policy file + 1 use case file

**Validation**:
- [ ] Policy references multiple Aggregates
- [ ] Policy documents coordination (not business logic)
- [ ] Use Case references policy

---

## Success Criteria

### Implementation Complete When:

- ✅ All documentation updated
- ✅ Single source of truth maintained
- ✅ No duplicated knowledge
- ✅ All links validated
- ✅ DDD compliance verified
- ✅ Tests passing
- ✅ Business rules enforced
- ✅ Acceptance criteria met

---

## Related Documents

- [Contribution Guidelines](../CONTRIBUTING_AI.md) - How to contribute
- [Change Impact Matrix](../architecture/change-impact-matrix.md) - Change propagation
- [Navigation Guide](../architecture/navigation.md) - Which docs to load
- [Architecture Principles](../architecture/README.md) - Design principles

---

**Playbook Version**: 1.0  
**Last Updated**: 2026-07-05  
**Maintained By**: AI Technical Lead
