# Business Rule Template

**Purpose**: Template for documenting business rules (invariants) in domain business-rules.md files.

**Usage**: Copy this template for each new business rule. Replace placeholders with actual content.

---

## Rule Template

### BR-{DOMAIN}-{NUMBER}: {Rule Name}

**Rule**: {Human-readable description of what the rule enforces}

**Formal Definition**: {Mathematical or logical definition}

**Examples**:
- ✅ **Valid**: `availableBalance >= 0`
- ✅ **Valid**: `count(transactions) <= 1000`
- ❌ **Invalid**: "Balance should be positive" (too informal)

---

**Enforcement Points**: {Where in the code this rule is enforced}

**Examples**:
- ✅ **Valid**: `debit(amount)` operation validates: `availableBalance - amount >= 0`
- ✅ **Valid**: Database constraint on `available_balance` column
- ✅ **Valid**: Application service validation before calling domain

**List specific operations, methods, or constraints**:
1. {Enforcement point 1}
2. {Enforcement point 2}
3. {Enforcement point 3}

---

**Violation Handling**: {What happens when this rule is violated}

**Examples**:
- ✅ **Valid**: Reject operation with error: "Insufficient balance"
- ✅ **Valid**: Publish `InsufficientBalance` domain event
- ✅ **Valid**: Throw `BusinessRuleViolationException`
- ❌ **Invalid**: "It fails" (too vague)

**Specify**:
- Error message (if applicable)
- Exception type (if applicable)
- Domain event published (if applicable)
- Fallback behavior (if applicable)

---

**Purpose**: {Why this rule exists}

**Examples**:
- ✅ **Valid**: "Maintain financial integrity - balance cannot go negative"
- ✅ **Valid**: "Prevent wallet overflow - balance has maximum limit"
- ✅ **Valid**: "Ensure audit trail - all transactions must be logged"

**Explain the business reason**:
- Business risk being mitigated
- Business requirement being satisfied
- Regulatory requirement being met
- Technical constraint being enforced

---

**Related**: {References to related concepts}

**Examples**:
- ✅ **Valid**: Related: [Wallet Model](./model.md) - Balance operations
- ✅ **Valid**: Related: BR-W-003 - Reserved balance validation
- ✅ **Valid**: Related: [BalanceChanged](./domain-events.md) event

**Link to**:
- Related business rules
- Related model entities
- Related domain events
- Related aggregate operations

---

## Complete Example

### BR-W-002: Balance Cannot Be Negative

**Rule**: Wallet available balance must never be negative.

**Formal Definition**: `availableBalance >= 0`

**Enforcement Points**:
- `debit(amount)` operation validates: `availableBalance - amount >= 0`
- `reserve(amount)` operation validates: `availableBalance - amount >= 0`
- Database check constraint on `available_balance` column

**Violation Handling**:
- Reject operation with error: "Insufficient balance"
- Publish `InsufficientBalance` domain event
- Return HTTP 400 for API calls

**Purpose**: Maintain financial integrity - a negative balance would represent debt that hasn't been properly documented or approved.

**Related**: 
- [Wallet Model](./model.md) - Balance operations
- [InsufficientBalance](./domain-events.md) event

---

## Best Practices

### ✅ DO

**Use formal definitions**:
```markdown
**Formal Definition**: availableBalance >= 0
```

**Specify exact enforcement points**:
```markdown
**Enforcement Points**:
- `Wallet.debit(amount)` method
- Database constraint: `CHECK (available_balance >= 0)`
```

**Document violation handling clearly**:
```markdown
**Violation Handling**:
- Throw `InsufficientBalanceException`
- Publish `InsufficientBalance` event
```

**Explain business purpose**:
```markdown
**Purpose**: Maintain financial integrity and prevent unauthorized debt.
```

---

### ❌ DON'T

**Don't use informal language**:
```markdown
❌ **Rule**: Balance should be positive
✅ **Rule**: Balance must be non-negative
✅ **Formal Definition**: availableBalance >= 0
```

**Don't omit enforcement points**:
```markdown
❌ **Enforcement Points**: It's enforced somewhere
✅ **Enforcement Points**:
- `Wallet.debit()` method
- Database constraint
```

**Don't leave violation handling vague**:
```markdown
❌ **Violation Handling**: It fails
✅ **Violation Handling**:
- Throw `InsufficientBalanceException`
- Return HTTP 400
```

**Don't forget the purpose**:
```markdown
❌ **Purpose**: None
✅ **Purpose**: Maintain financial integrity
```

---

## Naming Convention

### Rule ID Format

**Format**: `BR-{DOMAIN}-{NUMBER}`

**Examples**:
- `BR-W-001` - Wallet rule #1
- `BR-P-003` - Payment rule #3
- `BR-C-007` - Credit Transaction rule #7

**Domain Letters**:
- `W` - Wallet
- `P` - Payment
- `C` - Credit Transaction
- `L` - Ledger
- `I` - Invoice

**Numbering**:
- Start at 001 for each domain
- Use sequential numbers
- No gaps in numbering

---

## Common Business Rule Patterns

### Pattern 1: Validation Rule

**Example**:
```markdown
### BR-W-002: Balance Cannot Be Negative
**Rule**: Balance must be non-negative
**Formal Definition**: availableBalance >= 0
**Enforcement Points**: debit() operation
**Violation Handling**: Reject with error
```

---

### Pattern 2: Uniqueness Rule

**Example**:
```markdown
### BR-W-004: One Wallet Per Tenant
**Rule**: Each tenant has exactly one wallet
**Formal Definition**: UNIQUE(tenantId) WHERE deleted_at IS NULL
**Enforcement Points**: Wallet creation, database unique constraint
**Violation Handling**: Reject with "Wallet already exists"
```

---

### Pattern 3: Cardinality Rule

**Example**:
```markdown
### BR-W-005: One Asset Per Type Per Wallet
**Rule**: Each wallet has one asset of each type
**Formal Definition**: UNIQUE(tenantId, assetType)
**Enforcement Points**: Asset creation, database unique constraint
**Violation Handling**: Reject with "Asset already exists"
```

---

### Pattern 4: State Transition Rule

**Example**:
```markdown
### BR-P-003: Payment Status Transition
**Rule**: Payments must transition through valid states
**Formal Definition**: 
  PENDING → PROCESSING → COMPLETED
  PENDING → PROCESSING → FAILED
  PENDING → EXPIRED
**Enforcement Points**: Payment status update method
**Violation Handling**: Reject with "Invalid status transition"
```

---

### Pattern 5: Relationship Rule

**Example**:
```markdown
### BR-L-002: Double-Entry Accounting
**Rule**: Every transaction has equal debit and credit entries
**Formal Definition**: SUM(debitAmounts) = SUM(creditAmounts)
**Enforcement Points**: Transaction creation method
**Violation Handling**: Reject with "Unbalanced transaction"
```

---

## Validation Checklist

### Before Finalizing Business Rule

- [ ] **Rule ID** follows naming convention
- [ ] **Rule name** is clear and descriptive
- [ ] **Human-readable description** is unambiguous
- [ ] **Formal definition** is precise and mathematical
- [ ] **Enforcement points** are specific and complete
- [ ] **Violation handling** is clearly specified
- [ ] **Purpose** explains business reason
- [ ] **Related concepts** are linked
- [ ] **No duplication** with other rules
- [ ] **Consistent terminology** with rest of domain

---

## Integration with Other Documentation

### Where Business Rules Appear

**✅ Correct Location**:
- `domains/{domain}/business-rules.md` - **ONLY HERE**

**❌ Incorrect Locations**:
- Use cases - Reference, don't duplicate
- API docs - Reference, don't duplicate
- Controllers - Implement, don't document
- Database - Technical constraints, not business rules

### Reference Pattern

**In Use Cases**:
```markdown
## Domain References
### Business Rules Enforced
- [Wallet business rules](../domains/wallet/business-rules.md)
```

**In API Documentation**:
```markdown
## Business Rules
For detailed rules, see [Wallet business rules](../domains/wallet/business-rules.md).
```

---

## Common Mistakes to Avoid

### ❌ Mistake 1: Informal Language

**Wrong**:
```markdown
### BR-W-002: Balance Rule
Balance should probably be positive or zero, I think.
```

**Correct**:
```markdown
### BR-W-002: Balance Cannot Be Negative
**Rule**: Wallet available balance must never be negative.
**Formal Definition**: availableBalance >= 0
```

---

### ❌ Mistake 2: Missing Formal Definition

**Wrong**:
```markdown
### BR-W-002: Balance Cannot Be Negative
**Rule**: Balance must be non-negative.
```

**Correct**:
```markdown
### BR-W-002: Balance Cannot Be Negative
**Rule**: Wallet available balance must never be negative.
**Formal Definition**: availableBalance >= 0
```

---

### ❌ Mistake 3: Vague Enforcement

**Wrong**:
```markdown
**Enforcement Points**: It's enforced in the code somewhere.
```

**Correct**:
```markdown
**Enforcement Points**:
- `Wallet.debit(amount)` method
- Database constraint: `CHECK (available_balance >= 0)`
```

---

### ❌ Mistake 4: Missing Violation Handling

**Wrong**:
```markdown
**Violation Handling**: It fails.
```

**Correct**:
```markdown
**Violation Handling**:
- Throw `InsufficientBalanceException`
- Publish `InsufficientBalance` domain event
- Return HTTP 400 for API calls
```

---

### ❌ Mistake 5: No Purpose

**Wrong**:
```markdown
**Purpose**: None.
```

**Correct**:
```markdown
**Purpose**: Maintain financial integrity - a negative balance would represent unauthorized debt.
```

---

## Quick Reference

### Business Rule Structure

1. **Rule ID**: `BR-{DOMAIN}-{NUMBER}`
2. **Rule Name**: Clear, descriptive name
3. **Human Description**: What the rule enforces
4. **Formal Definition**: Mathematical/logical definition
5. **Enforcement Points**: Where enforced (specific methods, constraints)
6. **Violation Handling**: What happens on violation
7. **Purpose**: Why the rule exists
8. **Related**: Links to related concepts

---

### When to Create Business Rule

**Create when**:
- New business invariant discovered
- Existing rule needs clarification
- Rule enforcement point changes
- Rule violation handling changes

**Don't create when**:
- Documenting technical validation (not business rule)
- Documenting UI validation (not business rule)
- Documenting database constraints (technical, not business)

---

## Related Templates

- [Domain Event Template](./domain-event.md) - Events published on rule violation
- [Use Case Template](./use-case.md) - Workflows that enforce rules
- [API Template](./api.md) - APIs that enforce rules

---

**Template Version**: 1.0  
**Last Updated**: 2026-07-05  
**Maintained By**: AI Technical Lead
