# Use Case Template

**Purpose**: Template for documenting use cases following DDD conventions.

**When to Use**: When documenting application workflows and use cases.

---

## Use Case Structure

Every use case must follow this structure:

```markdown
# {Use Case Name}

## Business Goal
{What this use case achieves}

## Actor
{Who triggers this use case}

## Trigger
{What starts this use case}

## Preconditions
- {Precondition1}
- {Precondition2}

## Domain References

### Aggregates Involved
- [{Aggregate1}](../domains/{aggregate1}/aggregate.md) - {Role}
- [{Aggregate2}](../domains/{aggregate2}/aggregate.md) - {Role}

### Business Rules Enforced
- [{Domain} Business Rules](../domains/{domain}/business-rules.md) - {Which rules apply}
- Specifically: BR-{LETTER}-{NUMBER}, BR-{LETTER}-{NUMBER}

### Policies Applied
- [{Policy}](../policies/{policy}.md) - {How policy applies}

## Main Flow
1. {Step 1}
2. {Step 2}
3. {Step 3}

## Alternative Flow
### {Scenario}
1. {Alternative step 1}
2. {Alternative step 2}

## Failure Flow
### {Failure Scenario}
1. {Failure handling step 1}
2. {Failure handling step 2}

## Postconditions
- {Postcondition1}
- {Postcondition2}

## Acceptance Criteria
- ✅ {Criteria1}
- ✅ {Criteria2}
- ✅ {Criteria3}

## Related APIs
- [{API1}](../api/index.md#{endpoint}) - {Purpose}
- [{API2}](../api/index.md#{endpoint}) - {Purpose}

## Related Events
- [{Event1}](../domains/{domain}/domain-events.md#{event}) - {When published}
- [{Event2}](../domains/{domain}/domain-events.md#{event}) - {When published}
```

---

## Critical Principles

### 1. Domain References (CRITICAL)

**❌ WRONG**: Duplicating business rules in use case

```markdown
## Business Rules
- Balance must be sufficient ❌ WRONG
- Payment amount must be positive ❌ WRONG
```

**✅ CORRECT**: Referencing domain documentation

```markdown
## Domain References
### Business Rules Enforced
- [Wallet Business Rules](../domains/wallet/business-rules.md)
- [Payment Business Rules](../domains/payment/business-rules.md)
```

### 2. Single Source of Truth

**Principle**: Each piece of knowledge exists in EXACTLY ONE place.

| Concept | Single Location | Use Case References |
|---------|-----------------|---------------------|
| **Aggregate** | `domains/{domain}/aggregate.md` | ✅ Reference only |
| **Business Rules** | `domains/{domain}/business-rules.md` | ✅ Reference only |
| **Domain Events** | `domains/{domain}/domain-events.md` | ✅ Reference only |
| **Policies** | `policies/{policy}.md` | ✅ Reference only |
| **Use Case Flow** | `application/use-cases/{use-case}.md` | ✅ Own this |

### 3. Use Case Responsibilities

**Use Cases OWN**:
- ✅ Application workflow
- ✅ Step ordering
- ✅ Error handling flow
- ✅ Integration coordination

**Use Cases REFERENCE**:
- ❌ NOT aggregate definition
- ❌ NOT business rules
- ❌ NOT domain events
- ❌ NOT policies

---

## Example: Consume Credit Use Case

```markdown
# Consume Credit

## Business Goal
Deduct credits from user wallet for service usage

## Actor
Service (AI Service, POS, SPA, etc.)

## Trigger
Service requests credit consumption for user action

## Preconditions
- User has active wallet
- Wallet has sufficient balance
- Service has valid API credentials

## Domain References

### Aggregates Involved
- [Wallet](../domains/wallet/aggregate.md) - Balance source
- [CreditTransaction](../domains/credit-transaction/aggregate.md) - Transaction tracking
- [Ledger](../domains/ledger/aggregate.md) - Accounting record

### Business Rules Enforced
- [Wallet Business Rules](../domains/wallet/business-rules.md)
  - Specifically: BR-W-001 (Balance non-negative), BR-W-003 (Sufficient balance)
- [CreditTransaction Business Rules](../domains/credit-transaction/business-rules.md)
  - Specifically: BR-C-001 (Positive amount), BR-C-002 (Unique idempotency)

### Policies Applied
- [Consume Credit Policy](../policies/consume-credit.md) - Coordinated multi-aggregate flow

## Main Flow
1. Receive credit consumption request with idempotency key
2. Validate idempotency key - return cached result if exists
3. Load wallet by userId
4. Validate wallet has sufficient balance
5. Reserve credit amount (move from available to reserved)
6. Create CreditTransaction aggregate in PENDING status
7. Confirm credit consumption (deduct from reserved)
8. Mark CreditTransaction as COMPLETED
9. Create LedgerEntry for double-entry accounting
10. Publish CreditConsumedDomainEvent
11. Return success response with transaction details

## Alternative Flow

### Idempotent Request
1. Detect existing CreditTransaction with same idempotency key
2. Return cached transaction result
3. No balance changes made

## Failure Flow

### Insufficient Balance
1. Wallet balance validation fails
2. Create CreditTransaction in FAILED status
3. Publish InsufficientBalanceDomainEvent
4. Return INSUFFICIENT_BALANCE error

### Invalid Idempotency Key
1. Idempotency key validation fails
2. Return IDEMPOTENCY_KEY_INVALID error

## Postconditions
- Wallet balance decreased by consumed amount
- CreditTransaction created and completed
- LedgerEntry created for accounting
- CreditConsumedDomainEvent published

## Acceptance Criteria
- ✅ Valid consumption decreases balance
- ✅ Insufficient balance rejects request
- ✅ Idempotency key prevents double consumption
- ✅ Ledger entry created for audit trail
- ✅ Domain event published for integration

## Related APIs
- [POST /api/v1/credit/consume](../api/index.md#consume-credit) - Main endpoint

## Related Events
- [CreditConsumedDomainEvent](../domains/credit-transaction/domain-events.md#credit-consumed)
- [InsufficientBalanceDomainEvent](../domains/wallet/domain-events.md#insufficient-balance)
```

---

## Testing Template

### Use Case Test Structure

```csharp
public class {UseCase}Tests : {ApplicationTestBase}
{
    [Fact]
    public async Task Should_{HappyPath}()
    {
        // Arrange
        var input = new {InputDto} { {Property} = {value} };
        
        // Act
        var result = await _{appService}.{UseCase}Async(input);
        
        // Assert
        result.ShouldNotBeNull();
        result.{ExpectedProperty}.ShouldBe({expectedValue});
    }
    
    [Fact]
    public async Task Should_Fail_When_{FailureCondition}()
    {
        // Arrange
        var input = new {InputDto} { {Property} = {invalidValue} };
        
        // Act & Assert
        await Should.ThrowAsync<BusinessException>(
            async () => await _{appService}.{UseCase}Async(input)
        );
    }
    
    [Fact]
    public async Task Should_Handle_{AlternativeFlow}()
    {
        // Arrange
        var input = new {InputDto} { {Property} = {alternativeValue} };
        
        // Act
        var result = await _{appService}.{UseCase}Async(input);
        
        // Assert
        result.{ExpectedProperty}.ShouldBe({expectedForAlternative});
    }
}
```

---

## Common Pitfalls to Avoid

### ❌ Pitfall 1: Duplicating Business Rules

**Wrong**: Adding business rules to Use Case
```markdown
## Business Rules
- Balance must be sufficient  ❌ WRONG
```

**Correct**: Reference business rules
```markdown
## Domain References
### Business Rules Enforced
- [Wallet Business Rules](../domains/wallet/business-rules.md)
```

### ❌ Pitfall 2: Defining Aggregates in Use Case

**Wrong**: Explaining aggregate structure
```markdown
## Wallet Aggregate
The Wallet has AvailableBalance and ReservedBalance... ❌ WRONG
```

**Correct**: Reference aggregate documentation
```markdown
## Domain References
### Aggregates Involved
- [Wallet](../domains/wallet/aggregate.md) - Balance source
```

### ❌ Pitfall 3: Missing Domain References

**Wrong**: No references to domain documentation
```markdown
## Main Flow
1. Check balance is sufficient
2. Deduct amount
```

**Correct**: Include domain references
```markdown
## Domain References
### Business Rules Enforced
- [Wallet Business Rules](../domains/wallet/business-rules.md)
  - Specifically: BR-W-003 (Sufficient balance)

## Main Flow
1. Check balance is sufficient (enforced by BR-W-003)
2. Deduct amount (via Wallet.Reserve() operation)
```

---

## Quick Reference

### "What goes in a Use Case?"

| Section | Content | Source |
|---------|---------|--------|
| **Business Goal** | What this achieves | Use Case (own this) |
| **Actor** | Who triggers this | Use Case (own this) |
| **Domain References** | Links to domain docs | Domain docs (reference) |
| **Main Flow** | Step-by-step process | Use Case (own this) |
| **Acceptance Criteria** | Success checklist | Use Case (own this) |
| **Related APIs** | API endpoints | API docs (reference) |
| **Related Events** | Domain events | Domain docs (reference) |

---

## Validation Checklist

Before marking a Use Case as complete:

- [ ] Business goal clearly defined
- [ ] Actor and trigger identified
- [ ] Domain References section included
- [ ] Aggregates referenced (not defined)
- [ ] Business Rules referenced (not duplicated)
- [ ] Main flow documented
- [ ] Alternative flows documented
- [ ] Failure flows documented
- [ ] Acceptance criteria defined
- [ ] Related APIs linked
- [ ] Related Events linked
- [ ] No business rules duplicated
- [ ] No aggregate definitions duplicated
- [ ] All links resolve correctly

---

**Template Version**: 1.0  
**Last Updated**: 2026-07-27  
**Based On**: DDD Conventions and Wion Engineering Rules