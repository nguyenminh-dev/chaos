# Documentation Standards

**Purpose**: Universal documentation standards and patterns for all WION microservices.

**Scope**: These standards apply to **ALL services**. Use these patterns consistently across services.

---

## Documentation Principles

### 1. Single Source of Truth
Every concept exists in exactly ONE place.

**Location Matrix**:
| Concept | Location | NEVER In |
|---------|----------|----------|
| Aggregate Definition | `domains/{aggregate}/aggregate.md` | Use Cases, API docs |
| Business Rules | `domains/{aggregate}/business-rules.md` | Use Cases, Controllers |
| Lifecycle | `domains/{aggregate}/lifecycle.md` | Diagrams, API docs |
| Domain Events | `domains/{aggregate}/domain-events.md` | Event index, API docs |
| Entities & VOs | `domains/{aggregate}/model.md` | Separate files |

---

### 2. Reference, Don't Duplicate
**ALWAYS** use markdown links to reference knowledge.

**✅ CORRECT**:
```markdown
## Domain References
- [Wallet business rules](../domains/wallet/business-rules.md)
- [Payment events](../domains/payment/domain-events.md)
```

**❌ WRONG**:
```markdown
## Business Rules
- Balance must be non-negative  ❌ Duplicated from domain
```

---

### 3. Domain is Primary
- Business knowledge belongs in `domains/`
- Application layer coordinates (doesn't duplicate)
- API layer exposes (doesn't implement)
- Infrastructure supports (doesn't define)

---

## Documentation Structure

### Standard Service Layout
```
knowledge/
└── {service}/
    ├── README.md                 # Service overview
    ├── architecture/             # Architecture docs
    │   ├── bounded-context.md
    │   ├── context-map.md
    │   └── navigation.md
    ├── domains/                  # Business knowledge
    │   ├── {domain}/
    │   │   ├── overview.md
    │   │   ├── aggregate.md
    │   │   ├── model.md
    │   │   ├── business-rules.md
    │   │   ├── lifecycle.md
    │   │   ├── domain-events.md
    │   │   └── repositories.md
    ├── application/              # Use cases
    │   └── use-cases/
    ├── flows/                    # Business workflows
    ├── examples/                 # API examples
    ├── policies/                 # Cross-aggregate logic
    ├── decisions/                # ADRs
    ├── pitfalls/                 # Common mistakes
    ├── api/                      # API docs
    ├── events/                   # Event catalog
    ├── infrastructure/           # Technical details
    ├── reference/                # Service glossary
    └── adr/                      # Legacy ADR location
```

---

## Optional Directories

### flows/
**Purpose**: End-to-end business workflows

**Create when**:
- ✅ Process has 3+ distinct steps
- ✅ Multiple aggregates participate
- ✅ External system integration

**Don't create when**:
- ❌ Simple CRUD operation
- ❌ Single use case

**Examples**:
- Payment Topup to Invoice Flow
- Order Fulfillment Flow
- User Onboarding Flow

---

### examples/
**Purpose**: Real API request/response examples

**Create when**:
- ✅ API has non-trivial payloads
- ✅ Multiple error scenarios
- ✅ Integration complexity

**Examples**:
- Consume Credit Success
- Payment Webhook Error
- Invoice Generation

---

### decisions/
**Purpose**: Architecture Decision Records

**Create when**:
- ✅ Major architectural choice
- ✅ Technology selection
- ✅ Integration pattern decision

**Don't create for**:
- ❌ Implementation details
- ❌ Temporary workarounds

---

### pitfalls/
**Purpose**: Common implementation mistakes

**Create when**:
- ✅ Same mistake made 2+ times
- ✅ AI agents repeat error
- ✅ Code review catches frequently

**Examples**:
- Duplicating Business Rules
- Business Logic in Controllers
- Missing Idempotency

---

## Document Templates

### Business Rule Template
Location: `templates/business-rule.md`

**Usage**: Copy template for new business rules

**Structure**:
```markdown
### BR-{DOMAIN}-{NUMBER}: {Rule Name}
**Rule**: {Human-readable description}
**Formal Definition**: {Mathematical definition}
**Enforcement Points**: {Where enforced}
**Violation Handling**: {What happens}
**Purpose**: {Why it exists}
**Related**: {Links}
```

---

### Use Case Template
Location: `templates/use-case.md`

**Structure**:
```markdown
# {Use Case Name}

## Business Goal
## Actor
## Trigger
## Preconditions
## Domain References
### Aggregates Involved
### Business Rules Enforced
### Policies Applied
## Main Flow
## Alternative Flow
## Failure Flow
## Postconditions
## Acceptance Criteria
```

---

### API Documentation Template
Location: `templates/api.md`

**Structure**:
```markdown
## {API Name}

### Endpoint
`METHOD /api/v1/{path}`

### Purpose
{What this API does}

### Request
{Headers, parameters, body}

### Response
{Success and error responses}

### Domain Concepts
This API operates on [{Aggregate}](../domains/{aggregate}/aggregate.md).

### Business Rules
See [{Aggregate} business rules](../domains/{aggregate}/business-rules.md).
```

---

## Documentation Quality Standards

### Completeness Checklist
Every document should:
- [ ] Have clear purpose statement
- [ ] Link to related documents
- [ ] Use consistent terminology
- [ ] Include examples where helpful
- [ ] Be indexed in parent README

---

### Quality Metrics

| Metric | Target | Measurement |
|--------|--------|-------------|
| **Link Validity** | 100% | No broken internal links |
| **Term Consistency** | 100% | Use shared glossary terms |
| **Duplication** | 0% | No concept defined twice |
| **Navigation** | < 3 clicks | Reach any doc from README |

---

## Writing Guidelines

### 1. Use Active Voice
```markdown
✅ CORRECT: "The service validates the request."
❌ WRONG: "The request is validated by the service."
```

### 2. Be Specific
```markdown
✅ CORRECT: "Returns 400 Bad Request with INSUFFICIENT_BALANCE error code."
❌ WRONG: "Returns an error if balance is low."
```

### 3. Use Examples
```markdown
✅ CORRECT:
```json
{
  "tenantId": "tenant-12345",
  "amount": 1000
}
```
❌ WRONG: "Include tenantId and amount in request."
```

### 4. Link, Don't Repeat
```markdown
✅ CORRECT:
"See [Wallet business rules](../domains/wallet/business-rules.md)."

❌ WRONG:
"Wallet business rules include: balance >= 0, one wallet per tenant..."
```

---

## Maintenance Workflow

### Adding New Documentation

1. **Identify location** using structure matrix
2. **Load templates** from relevant template
3. **Reference** existing docs (don't duplicate)
4. **Link** from parent README
5. **Validate** no broken links

---

### Updating Existing Documentation

1. **Load document** to understand context
2. **Identify impact** on related docs
3. **Update** single source of truth
4. **Verify** references still work
5. **Update** navigation if needed

---

### Removing Documentation

1. **Check references** via search
2. **Update or remove** referencing docs
3. **Remove** from navigation
4. **Delete** file
5. **Verify** no broken links

---

## Common Anti-Patterns

### ❌ Anti-Pattern 1: Duplicating Shared Knowledge

**WRONG**:
```markdown
# service/reference/glossary.md

## Aggregate
Definition: A cluster of domain objects...  ❌ WRONG
```

**CORRECT**:
```markdown
# service/reference/glossary.md

## Aggregate
See [shared glossary](../../shared/glossary.md#aggregate).

Service-specific aggregates:
- Wallet Aggregate
- Payment Aggregate
```

---

### ❌ Anti-Pattern 2: Feature-Based Documentation

**WRONG**:
```
knowledge/
└── billing-service/
    ├── features/
    │   ├── wallet-management/
    │   └── payment-processing/
```

**CORRECT**:
```
knowledge/
└── billing-service/
    ├── domains/
    │   ├── wallet/
    │   └── payment/
    ├── application/
    └── flows/
```

---

### ❌ Anti-Pattern 3: Outdated Navigation

**WRONG**: README lists files that don't exist

**CORRECT**: Keep README synchronized with actual files

---

## Tools & Automation

### Link Validation
```bash
# Find broken markdown links
find .claude -name "*.md" -exec grep -l "\]\(" {} \; | xargs -I {} markdown-link-check {}
```

### Duplication Detection
```bash
# Find duplicated section headers
grep -r "^## " .claude/knowledge/ | sort | uniq -d
```

### Orphaned Files
```bash
# Find files not linked from any README
# (Custom script needed)
```

---

## Related Documentation

- [DDD Conventions](./ddd-conventions.md) - Domain modeling standards
- [Event Conventions](./event-conventions.md) - Event documentation patterns
- [API Conventions](./api-conventions.md) - API documentation standards
- [Shared Glossary](./glossary.md) - Universal terminology

---

## Service Documentation Checklist

When creating or updating service documentation:

### Structure
- [ ] Standard directories exist (domains, application, api, etc.)
- [ ] Optional directories created only when needed (flows, examples, decisions, pitfalls)
- [ ] No empty directories
- [ ] No duplicate directory structures

### Content
- [ ] Business knowledge in `domains/`
- [ ] Use cases in `application/use-cases/`
- [ ] Business rules only in `business-rules.md`
- [ ] No duplication of shared definitions
- [ ] All concepts have single source of truth

### Navigation
- [ ] README.md in each directory
- [ ] Service README links to all sections
- [ ] Domain README lists all domains
- [ ] All files reachable from navigation

### Quality
- [ ] No broken internal links
- [ ] Consistent terminology (use shared glossary)
- [ ] Examples provided where helpful
- [ ] Related documents linked

---

**Last Updated**: 2026-07-05
**Maintained By**: Architecture Team
**Version**: 1.0
