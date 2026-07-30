# Navigation Guide for AI Agents

This document guides AI agents on which documents to load for different tasks, ensuring efficient and accurate assistance.

## Quick Reference

| Task Type | Primary Documents | Secondary Documents |
|-----------|------------------|-------------------|
| **Domain Understanding** | `architecture/bounded-context.md`, `architecture/context-map.md` | `domains/`, `policies/` |
| **Business Rules** | `domains/*/{domain}/business-rules.md` | Specific aggregate docs |
| **Use Case Understanding** | `application/use-cases/{use-case}.md` | Referenced domain docs |
| **API Development** | `api/index.md` | Referenced domain docs |
| **Event Handling** | `events/index.md` | `domains/*/{domain}/domain-events.md` |
| **Database Changes** | `infrastructure/database.md` | `domains/*/{domain}/model.md` |
| **Adding Features** | `CONTRIBUTING_AI.md` | This navigation guide |
| **Architecture Decisions** | `architecture/` | Context-specific docs |

---

## Task-Based Navigation

### Task 1: "Help me understand the Billing Service"

**Load in Order**:
1. `README.md` - Overview and quick links
2. `architecture/bounded-context.md` - Context ownership and responsibilities
3. `architecture/context-map.md` - Relationships with other contexts
4. `domains/wallet/overview.md` - Example domain structure

**Do NOT Load** (unless specifically asked):
- All domain files (start with one example)
- All use cases (load only if workflow question)
- Infrastructure details (unless technical question)

**Expected Outcome**: AI agent understands service boundaries, responsibilities, and relationships without loading all 40+ files.

---

### Task 2: "What business rules apply to Wallet?"

**Load in Order**:
1. `domains/wallet/business-rules.md` - Complete business rules

**Reference Links** (follow if needed):
2. `domains/wallet/aggregate.md` - Aggregate context
3. `domains/wallet/model.md` - Entity definitions

**Do NOT Load**:
- Use Cases (they reference, not duplicate)
- API docs (they reference, not duplicate)

**Expected Outcome**: AI agent provides accurate business rules from single source of truth.

---

### Task 3: "How do I consume credits?"

**Load in Order**:
1. `application/use-cases/consume-credit.md` - Complete use case flow

**Do NOT Load**:
- All domain files (use case already references them)
- API docs (unless endpoint details needed)

**Expected Outcome**: AI agent explains complete workflow with domain knowledge embedded via references.

---

### Task 4: "How do I add a new API endpoint?"

**Load in Order**:
1. `CONTRIBUTING_AI.md` - Contribution guidelines
2. `api/index.md` - Existing API patterns

**Do NOT Load**:
- Domain files (unless need to reference domain concepts)

**Expected Outcome**: AI agent guides API development following correct structure.

---

### Task 5: "What happens when a payment succeeds?"

**Load in Order**:
1. `application/use-cases/handle-webhook.md` - Webhook processing
2. `policies/invoice-after-payment.md` - Cross-aggregate coordination

**Do NOT Load**:
- All domain files (policies already aggregate what's needed)

**Expected Outcome**: AI agent explains complete flow with multiple aggregates coordinated.

---

### Task 6: "I need to modify the Wallet Aggregate"

**Load in Order**:
1. `CONTRIBUTING_AI.md` - Change guidelines
2. `domains/wallet/aggregate.md` - Current Aggregate definition
3. `domains/wallet/model.md` - Current entities
4. `domains/wallet/business-rules.md` - Current business rules

**Do NOT Load**:
- Use Cases (they reference aggregate, no need to update)
- API docs (they reference aggregate, no need to update)

**Expected Outcome**: AI agent understands current structure before making changes.

---

### Task 7: "Where should I document a new business rule?"

**Load in Order**:
1. `CONTRIBUTING_AI.md` - Contribution guidelines
2. `domains/{aggregate}/business-rules.md` - Example format

**Expected Outcome**: AI agent knows exactly where and how to document.

---

### Task 8: "How do I integrate a new external service?"

**Load in Order**:
1. `architecture/context-map.md` - Existing integration patterns
2. `architecture/bounded-context.md` - Context boundaries

**Do NOT Load**:
- Domain files (unless understanding domain requirements)

**Expected Outcome**: AI agent follows established integration patterns.

---

### Task 9: "What events does Billing Service publish?"

**Load in Order**:
1. `events/index.md` - Complete event catalog
2. `domains/{domain}/domain-events.md` - Domain-specific event details (if deeper understanding needed)

**Expected Outcome**: AI agent provides complete event overview with domain context.

---

### Task 10: "I need to add a cross-aggregate feature"

**Load in Order**:
1. `CONTRIBUTING_AI.md` - Contribution guidelines
2. `policies/invoice-after-payment.md` - Example policy structure

**Do NOT Load**:
- Domain files (policies coordinate across domains)

**Expected Outcome**: AI agent creates policy in correct location.

---

## Document Loading Priority

### Tier 1: Always Load First
These documents provide foundational understanding:
1. `README.md`
2. `CONTRIBUTING_AI.md`
3. `architecture/bounded-context.md`
4. `architecture/context-map.md`

### Tier 2: Load by Task Type
- **Domain questions**: Load specific `domains/{domain}/` files
- **Use Case questions**: Load specific `application/use-cases/` files
- **API questions**: Load `api/index.md`
- **Event questions**: Load `events/index.md`

### Tier 3: Load on Demand
- Deep domain details: Load specific domain files
- Infrastructure details: Load `infrastructure/` files
- Architecture decisions: Load `architecture/` files

---

## Agent Behavior Guidelines

### For Exploratory Questions
When agent asks exploratory questions ("What does X do?", "How does Y work?"):
1. Load `README.md` first
2. Load relevant context document (`bounded-context.md` or `context-map.md`)
3. Load specific domain or use case file
4. Synthesize answer with references

### For Implementation Questions
When agent helps with implementation ("Add feature X", "Modify API Y"):
1. Load `CONTRIBUTING_AI.md` first
2. Load relevant templates/examples
3. Follow contribution guidelines
4. Verify structure compliance

### For Verification Questions
When agent verifies structure ("Is this correct?", "Did I miss anything?"):
1. Load related files
2. Check SSOT compliance (no duplication)
3. Verify DDD principles
4. Provide feedback

---

## Common Navigation Mistakes

### ❌ Mistake 1: Loading All Files
**Wrong**: Loading all 47 files for every question

**Correct**: Load only relevant files based on task

### ❌ Mistake 2: Loading Domain Files for Use Case Questions
**Wrong**: Loading domain files when use case already has the info

**Correct**: Load use case file, follow references if needed

### ❌ Mistake 3: Loading Infrastructure for Domain Questions
**Wrong**: Loading database files for business rule questions

**Correct**: Load domain business-rules.md

### ❌ Mistake 4: Not Loading Context First
**Wrong**: Jumping into implementation without understanding context

**Correct**: Load bounded-context.md and context-map.md first

---

## Navigation Templates

### Template: "Implement New Feature"
```
Load sequence:
1. CONTRIBUTING_AI.md
2. architecture/context-map.md (for relationships)
3. Relevant domain files
4. Similar use cases (for patterns)

Check:
- Is this a new Aggregate? → Create in domains/
- Is this cross-aggregate? → Create in policies/
- Is this application flow? → Create in application/use-cases/
```

### Template: "Modify Business Rule"
```
Load sequence:
1. CONTRIBUTING_AI.md
2. domains/{aggregate}/business-rules.md
3. Related aggregate.md (for context)

Check:
- Does this affect other Aggregates? → May need policy
- Does this change Use Cases? → No, use cases reference
- Does this change APIs? → No, APIs reference
```

### Template: "Add New API"
```
Load sequence:
1. CONTRIBUTING_AI.md
2. api/index.md (for patterns)
3. Relevant domain/aggregate.md (for concepts)

Check:
- Domain concept exists? → Reference it
- Business rule exists? → Reference it
- Event published? → Reference it
```

---

## Document Relationships

### Reference Graph
```
README.md
  ↓
├─→ architecture/bounded-context.md
├─→ architecture/context-map.md
├─→ CONTRIBUTING_AI.md
└─→ domains/ (via links)
    └─→ {domain}/overview.md
        └─→ {domain}/aggregate.md
            └─→ {domain}/business-rules.md
```

### Loading Strategy
```
For ANY task:
1. Load README.md (overview + links)
2. Load CONTRIBUTING_AI.md (guidelines)
3. Load task-specific document
4. Follow reference links as needed

STOP when you have sufficient information
```

---

## Related Documents
- [Bounded Context](./architecture/bounded-context.md)
- [Context Map](./architecture/context-map.md)
- [Contribution Guidelines](./CONTRIBUTING_AI.md)
- [README](./README.md)
