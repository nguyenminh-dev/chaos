# Change Impact Matrix

This document describes which documents must be updated when different parts of the system change, ensuring Single Source of Truth (SSOT) is maintained.

## Impact Categories

| Change Type | Description | Files to Update | Files That Auto-Update |
|-------------|-------------|----------------|----------------------|
| **Aggregate Change** | Aggregate structure, entities, value objects change | `domains/{aggregate}/aggregate.md`, `model.md` | Use Cases (via references), API docs (via references) |
| **Business Rule Change** | Business invariant added/modified | `domains/{aggregate}/business-rules.md` | Use Cases (via references), API docs (via references) |
| **API Change** | New endpoint, contract modification | `api/index.md` | - |
| **Domain Event Change** | Event added, payload modified | `domains/{aggregate}/domain-events.md` | Use Cases (via references), event-index.md |
| **Policy Change** | Cross-aggregate logic modified | `policies/{policy}.md` | Affected Use Cases |
| **Use Case Change** | Application workflow modified | `application/use-cases/{use-case}.md` | - |

---

## Detailed Change Scenarios

### Scenario 1: Add New Attribute to an Entity

**Example**: Add `creditLimit` attribute to Wallet entity

**Files to UPDATE**:
1. `domains/wallet/model.md` - Add `creditLimit` to Wallet entity
2. `domains/wallet/business-rules.md` - Add BR-W-XXX: Credit limit validation rule

**Files That AUTO-UPDATE** (via references):
- ✅ `application/use-cases/consume-credit.md` - References business-rules.md
- ✅ `api/index.md` - References Wallet model

**Files to NEVER TOUCH**:
- ❌ Other domain files (no change)
- ❌ Other use cases (no change needed)
- ❌ Database schema (separate concern)

**Validation**:
- [ ] Credit limit rule appears ONLY in business-rules.md
- [ ] No duplication of credit limit definition
- [ ] All references still resolve correctly

---

### Scenario 2: Add New Business Rule

**Example**: Add "Wallet balance cannot exceed credit limit" rule

**Files to UPDATE**:
1. `domains/wallet/business-rules.md` - Add BR-W-XXX

**Files That AUTO-UPDATE** (via references):
- ✅ `application/use-cases/consume-credit.md` - References business-rules.md
- ✅ `application/use-cases/process-payment.md` - References business-rules.md
- ✅ `api/index.md` - References business-rules.md

**Files to NEVER TOUCH**:
- ❌ Don't update Use Cases (they already reference the rule file)
- ❌ Don't update API docs (they already reference the rule file)
- ❌ Don't duplicate in database constraints (technical, not business)

**Validation**:
- [ ] Business rule appears ONLY in business-rules.md
- [ ] Use Cases still have correct reference links
- [ ] API docs still have correct reference links

---

### Scenario 3: Modify Aggregate Root

**Example**: Add new operation to Wallet Aggregate

**Files to UPDATE**:
1. `domains/wallet/aggregate.md` - Add operation to Aggregate Root

**Files That AUTO-UPDATE** (via references):
- ✅ `application/use-cases/consume-credit.md` - References aggregate.md

**Files to NEVER TOUCH**:
- ❌ Don't update Use Cases (they reference aggregate.md)
- ❌ Don't update API docs

**Validation**:
- [ ] Aggregate definition appears ONLY in aggregate.md
- [ ] Operation documented with purpose and parameters

---

### Scenario 4: Add New Domain Event

**Example**: Add `WalletFrozen` event

**Files to UPDATE**:
1. `domains/wallet/domain-events.md` - Add event definition and payload
2. `events/index.md` - Add event to catalog

**Files That AUTO-UPDATE** (via references):
- ✅ `application/use-cases/consume-credit.md` - References domain-events.md

**Files to NEVER TOUCH**:
- ❌ Don't update Use Cases (they reference domain-events.md)
- ❌ Don't duplicate event definition in multiple places

**Validation**:
- [ ] Event defined ONLY in domain-events.md
- [ ] Event catalog references domain-events.md
- [ ] Use Cases reference domain-events.md

---

### Scenario 5: Add New API Endpoint

**Example**: Add `POST /api/v1/wallets/{id}/freeze`

**Files to UPDATE**:
1. `api/index.md` - Add API documentation

**Files to REFERENCE** (not update):
- ℹ️ `domains/wallet/aggregate.md` - Reference aggregate concepts
- ℹ️ `domains/wallet/business-rules.md` - Reference business rules

**Files to NEVER TOUCH**:
- ❌ Don't update domain files (no business logic change)
- ❌ Don't update Use Cases (no application logic change)

**Validation**:
- [ ] API docs reference domain concepts via links
- [ ] No business rules defined in API docs
- [ ] No Aggregate definitions in API docs

---

### Scenario 6: Add New Use Case

**Example**: Add "Transfer Credits" use case

**Files to CREATE**:
1. `application/use-cases/transfer-credits.md` - New use case document

**Files to REFERENCE** (not update):
- ℹ️ `domains/wallet/aggregate.md` - Reference Wallet operations
- ℹ️ `domains/credit-transaction/aggregate.md` - Reference transaction tracking

**Files to NEVER TOUCH**:
- ❌ Don't update domain files (use case is application logic)
- ❌ Don't duplicate domain knowledge

**Validation**:
- [ ] Use Case references Aggregates via links
- [ ] Use Case references Business Rules via links
- [ ] No business logic implemented in Use Case

---

### Scenario 7: Modify Use Case Flow

**Example**: "Consume Credit" use case adds retry logic

**Files to UPDATE**:
1. `application/use-cases/consume-credit.md` - Update flow

**Files to NEVER TOUCH**:
- ❌ Don't update domain files (no business logic change)

**Validation**:
- [ ] Domain references still correct
- [ ] No business rules duplicated

---

### Scenario 8: Add New Policy

**Example**: Add "Multi-Wallet Transfer" policy

**Files to CREATE**:
1. `policies/multi-wallet-transfer.md` - New policy document

**Files to REFERENCE** (not update):
- ℹ️ `domains/wallet/aggregate.md` - Reference Wallet operations
- ℹ️ `domains/credit-transaction/aggregate.md` - Reference transaction tracking

**Files to UPDATE**:
- ℹ️ Use Cases that use this policy (add policy reference)

**Files to NEVER TOUCH**:
- ❌ Don't update domain files (policy is coordination layer)

**Validation**:
- [ ] Policy references multiple Aggregates
- [ ] Policy documents Trigger, Flow, Failure Handling
- [ ] Use Cases reference policy when applicable

---

### Scenario 9: Modify Policy

**Example**: "Invoice After Payment" policy changes retry strategy

**Files to UPDATE**:
1. `policies/invoice-after-payment.md` - Update retry logic

**Files That AUTO-UPDATE** (via references):
- ✅ `application/use-cases/create-invoice.md` - References policy
- ✅ `application/use-cases/handle-webhook.md` - References policy

**Files to NEVER TOUCH**:
- ❌ Don't update domain files (policy coordinates, doesn't contain logic)

**Validation**:
- [ ] Policy changes documented only in policy file
- [ ] Use Cases still reference policy correctly

---

### Scenario 10: Modify Lifecycle

**Example**: Wallet lifecycle adds "Frozen" state

**Files to UPDATE**:
1. `domains/wallet/lifecycle.md` - Update lifecycle states

**Files That AUTO-UPDATE** (via references):
- ✅ `domains/wallet/aggregate.md` - References lifecycle.md
- ✅ `domains/wallet/business-rules.md` - May reference lifecycle states

**Files to NEVER TOUCH**:
- ❌ Don't update Use Cases (they reference lifecycle.md)
- ❌ Don't update API docs

**Validation**:
- [ ] Lifecycle documented ONLY in lifecycle.md
- [ ] Other files reference lifecycle.md via links

---

## Cross-Reference Validation

### After Any Change

1. **Search for business rule name** - Should appear ONLY in `business-rules.md`
   ```bash
   grep -r "BR-W-002" billing-service/domains/
   # Should find 1 result: domains/wallet/business-rules.md
   ```

2. **Search for Aggregate name** - Should appear ONLY in `aggregate.md` (except references)
   ```bash
   grep -r "Wallet Aggregate" billing-service/domains/
   # Should find 1 result: domains/wallet/aggregate.md
   ```

3. **Check for broken links** - All references should resolve
   - Test by clicking links in markdown preview
   - Verify relative paths are correct

4. **Verify Use Cases** - Should reference, not duplicate
   - Check that Use Case has "Domain References" section
   - Check that business rules are referenced, not duplicated

---

## Dependency Graph

### File Dependency Matrix

```
[README.md]
  ↓
├─→ [architecture/bounded-context.md]
├─→ [architecture/context-map.md]
├─→ [CONTRIBUTING_AI.md]
└─→ [domains/]

[api/index.md]
  ↓
├─→ [domains/*/aggregate.md] (references concepts)
└─→ [domains/*/business-rules.md] (references rules)

[application/use-cases/*.md]
  ↓
├─→ [domains/*/aggregate.md] (references Aggregates)
├─→ [domains/*/business-rules.md] (references rules)
└─→ [policies/*.md] (references policies)

[policies/*.md]
  ↓
└─→ [domains/*/aggregate.md] (references multiple Aggregates)
```

### Change Propagation

When a file changes:
- **Files it references** → Auto-update (no changes needed, references handle it)
- **Files that reference it** → May need updates if structure changes significantly

**Example**:
```
domains/wallet/business-rules.md CHANGES
    ↓ (references)
application/use-cases/consume-credit.md → Auto-updated (no change needed)
api/index.md → Auto-updated (no change needed)
```

---

## Change Impact Summary Table

| Change | Impact Scope | Files Affected | Auto-Update? |
|--------|--------------|---------------|--------------|
| **Add Entity Attribute** | Low | 1 domain file | ✅ Yes (via refs) |
| **Add Business Rule** | Low | 1 domain file | ✅ Yes (via refs) |
| **Modify Aggregate** | Low | 1-2 domain files | ✅ Yes (via refs) |
| **Add Domain Event** | Medium | 1 domain file + event-index | ✅ Yes (via refs) |
| **Add API** | Low | 1 API file | ❌ No |
| **Add Use Case** | Low | 1 use case file | ❌ No |
| **Modify Use Case** | Low | 1 use case file | ❌ No |
| **Add Policy** | Medium | 1 policy file | ❌ No |
| **Modify Policy** | Medium | 1 policy file | ✅ Yes (via refs) |
| **Remove Entity Attribute** | Low | 1 domain file | ✅ Yes (via refs) |
| **Remove Business Rule** | Low | 1 domain file | ✅ Yes (via refs) |
| **Remove API** | Low | 1 API file | ❌ No |
| **Remove Use Case** | Low | 1 use case file | ❌ No |

**Legend**:
- Low Impact: 1-2 files, straightforward change
- Medium Impact: 2-3 files, some coordination needed
- High Impact: 4+ files, requires careful validation

---

## Validation Checklist

### After Making Changes

Verify the following:

#### SSOT Validation
- [ ] Business rule appears in ONLY `business-rules.md`
- [ ] Aggregate appears in ONLY `aggregate.md` (except references)
- [ ] Domain event appears in ONLY `domain-events.md` (except catalog)
- [ ] No business knowledge duplicated across documents

#### Link Validation
- [ ] All relative links resolve correctly
- [ ] No broken image links
- [ ] No circular references (A→B→A)
- [ ] Cross-domain references use correct relative paths

#### DDD Compliance
- [ ] Business rules ONLY in Domain layer
- [ ] Use Cases reference Aggregates (not duplicate)
- [ ] Use Cases reference Business Rules (not duplicate)
- [ ] Use Cases reference Policies (for cross-aggregate logic)
- [ ) API docs reference domain concepts (not duplicate)
- [ ] No cross-aggregate logic inside any Aggregate
- [ ) Policies used for cross-aggregate coordination

#### Documentation Completeness
- [ ] All referenced documents exist
- [ ] All domain files have required sections
- [ ] All use cases have "Domain References" section
- [ ] All policies document Trigger, Flow, Failure Handling

---

## Quick Reference

### "I changed X, what else needs updating?"

| Changed | What to Check |
|---------|---------------|
| **Entity attribute** | business-rules.md (may need validation), Use Cases (auto-update), API docs (auto-update) |
| **Business rule** | Use Cases (auto-update), API docs (auto-update) |
| **Aggregate operation** | Use Cases (auto-update) |
| **Domain event** | Use Cases (auto-update), event-index.md |
| **API contract** | No domain files (API owns contract) |
| **Use Case flow** | No domain files (Use Case owns flow) |
| **Policy logic** | Use Cases that reference the policy |

### "I want to add X, where does it go?"

| What to Add | Location |
|-------------|----------|
| **New Aggregate** | `domains/{aggregate}/` (7 files) |
| **New Business Rule** | `domains/{aggregate}/business-rules.md` |
| **New Entity/Value Object** | `domains/{aggregate}/model.md` |
| **New Domain Event** | `domains/{aggregate}/domain-events.md` |
| **New Use Case** | `application/use-cases/{use-case}.md` |
| **New API** | `api/index.md` |
| **New Cross-Aggregate Logic** | `policies/{policy}.md` |

---

## Examples

### Example 1: Adding "Wallet Freeze" Feature

**Step 1**: Identify scope
- Is this a new entity attribute? → Add to model.md
- Is this a new business rule? → Add to business-rules.md
- Is this a new lifecycle state? → Add to lifecycle.md

**Step 2**: Update domain files
```markdown
# domains/wallet/model.md
Add isFrozen: boolean to Wallet entity

# domains/wallet/lifecycle.md
Add [Frozen] state between Active and Soft Deleted

# domains/wallet/business-rules.md
Add BR-W-008: Frozen wallet cannot process transactions
```

**Step 3**: Verify impact
- Use Cases auto-update (no changes needed)
- API docs auto-update (no changes needed)

**Files Updated**: 3 domain files

---

### Example 2: Modifying "Consume Credit" Use Case

**Step 1**: Identify scope
- Is business rule changing? → No
- Is aggregate operation changing? → No
- Is workflow changing? → Yes

**Step 2**: Update use case
```markdown
# application/use-cases/consume-credit.md
Add retry logic to Main Flow
```

**Step 3**: Verify impact
- No domain files changed (use case owns the workflow)
- Business rules still valid (no rule change)

**Files Updated**: 1 use case file

---

### Example 3: Adding "Credit Limit" API

**Step 1**: Identify scope
- New API → Update api/index.md
- New domain concept → Update domain files
- New business rule → Update business-rules.md

**Step 2**: Update files
```markdown
# api/index.md
Add GET /api/v1/wallets/{id}/credit-limit

References:
- [Wallet Aggregate](../domains/wallet/aggregate.md)
- [Wallet Business Rules](../domains/wallet/business-rules.md)
```

**Step 3**: Verify impact
- Domain files referenced (not duplicated)
- Business rules referenced (not duplicated)

**Files Updated**: 1 API file

---

## Related Documents
- [Contribution Guidelines](./CONTRIBUTING_AI.md) - How to contribute
- [Navigation Guide](./architecture/navigation.md) - Which docs to load
- [Bounded Context](./architecture/bounded-context.md) - Context ownership
- [Context Map](./architecture/context-map.md) - Context relationships
