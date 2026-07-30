# Test-Driven Development (TDD) Playbook

**Purpose**: Strict TDD workflow for implementing features following Outside-In TDD principles.

**When to Use**: FOR EVERY FEATURE implementation. No exceptions.

---

## Core TDD Principle

**🔴 RED → 🟢 GREEN → 🔵 BLUE**

1. **🔴 RED**: Write FAILING test first
2. **🟢 GREEN**: Write MINIMUM code to make test pass
3. **🔵 BLUE**: Refactor to improve quality

**NEVER write code before tests exist.**

---

## Outside-In TDD Workflow

### Phase 1: Understand Requirements

**Step 1.1: Load Documentation**
```
1. Load CLAUDE.md (Wion Engineering Rules)
2. Load service README.md
3. Load relevant domain documentation
4. Load DDD conventions
```

**Step 1.2: Identify Acceptance Criteria**
- What business capability is being added?
- What are the success conditions?
- What are the failure conditions?
- What edge cases exist?

---

### Phase 2: Design API Contract

**Step 2.1: Define Interface**
```csharp
// Define what the interface SHOULD look like
public interface IWalletService
{
    Task<decimal> GetBalanceAsync(string userId);
}
```

**Step 2.2: Document Expected Behavior**
```markdown
## GetBalanceAsync
**Input**: userId (string)
**Output**: available balance (decimal)
**Errors**: WalletNotFoundException, InsufficientBalanceException
```

---

### Phase 3: Write FAILING Tests (🔴 RED)

### Step 3.1: Acceptance Test (Outer Layer)

**Write test that fails because implementation doesn't exist:**

```csharp
[Fact]
public async Task Should_Get_Wallet_Balance()
{
    // Arrange
    var userId = "user-123";
    var expectedBalance = 1000m;
    
    // Act
    var balance = await _walletService.GetBalanceAsync(userId);
    
    // Assert
    balance.ShouldBe(expectedBalance);
}
```

**VERIFY TEST FAILS**: Run test - it MUST fail.

---

### Step 3.2: Application Test (Middle Layer)

**Write failing application test:**

```csharp
[Fact]
public async Task Should_Return_Balance_When_Wallet_Exists()
{
    // Arrange
    var input = new GetBalanceDto { UserId = "user-123" };
    
    // Act
    var result = await _walletAppService.GetAsync(input);
    
    // Assert
    result.ShouldNotBeNull();
    result.Balance.ShouldBeGreaterThanOrEqualTo(0);
}
```

**VERIFY TEST FAILS**: Run test - it MUST fail.

---

### Step 3.3: Domain Test (Inner Layer)

**Write failing domain test:**

```csharp
[Fact]
public void Should_Create_Wallet_With_Zero_Balance()
{
    // Arrange
    var userId = "user-123";
    
    // Act
    var wallet = Wallet.CreateNew(userId);
    
    // Assert
    wallet.AvailableBalance.ShouldBe(0);
    wallet.ReservedBalance.ShouldBe(0);
}
```

**VERIFY TEST FAILS**: Run test - it MUST fail.

---

## Phase 4: Implement Domain (🟢 GREEN)

### Step 4.1: Implement MINIMUM Domain Code

**Write ONLY enough code to make domain test pass:**

```csharp
public class Wallet : BillingAggregateRoot
{
    public decimal AvailableBalance { get; private set; }
    
    public static Wallet CreateNew(string userId)
    {
        return new Wallet
        {
            UserId = userId,
            AvailableBalance = 0  // Minimum to pass test
        };
    }
}
```

**VERIFY TEST PASSES**: Run domain test - it MUST pass.

---

### Step 4.2: Implement Application Layer

**Write ONLY enough application code to make test pass:**

```csharp
public class WalletAppService : ApplicationService
{
    [UnitOfWork]
    public async Task<BalanceDto> GetAsync(GetBalanceDto input)
    {
        var wallet = await _walletRepository.FindByUserIdAsync(input.UserId);
        return new BalanceDto { Balance = wallet.AvailableBalance };
    }
}
```

**VERIFY TEST PASSES**: Run application test - it MUST pass.

---

### Step 4.3: Implement API Layer

**Write ONLY enough API code to make test pass:**

```csharp
[HttpPost("balance")]
public async Task<BalanceDto> GetBalance([FromBody] GetBalanceDto input)
{
    return await _walletAppService.GetAsync(input);
}
```

**VERIFY TEST PASSES**: Run acceptance test - it MUST pass.

---

## Phase 5: Refactor (🔵 BLUE)

### Step 5.1: Improve Domain Code

**Add business rules, value objects, etc.:**

```csharp
public class Wallet : BillingAggregateRoot
{
    public decimal AvailableBalance { get; private set; }
    
    public static Wallet CreateNew(string userId)
    {
        var wallet = new Wallet { UserId = userId };
        wallet.AddLocalEvent(new WalletCreatedDomainEvent(wallet.Id, userId));
        return wallet;
    }
    
    public void Credit(decimal amount)
    {
        CheckRule(new CreditAmountMustBePositiveRule(amount));
        AvailableBalance += amount;
        AddLocalEvent(new BalanceChangedDomainEvent(...));
    }
}
```

**VERIFY ALL TESTS STILL PASS**: No tests should break during refactoring.

---

### Step 5.2: Improve Application Code

**Add error handling, validation, etc.:**

```csharp
public class WalletAppService : ApplicationService
{
    [UnitOfWork]
    public async Task<BalanceDto> GetAsync(GetBalanceDto input)
    {
        var wallet = await _walletRepository.FindByUserIdAsync(input.UserId);
        
        if (wallet == null)
            throw new BusinessException(WalletNotFound);
            
        return new BalanceDto { Balance = wallet.AvailableBalance };
    }
}
```

**VERIFY ALL TESTS STILL PASS**: All tests must pass.

---

## TDD Rules (Mandatory)

### Rule 1: Never Skip Red Phase
❌ **WRONG**: Write implementation code, then write tests
✅ **CORRECT**: Write failing tests, then write implementation

### Rule 2: Write Minimum Code
❌ **WRONG**: Implement full feature in Green phase
✅ **CORRECT**: Write ONLY enough code to make test pass

### Rule 3: Refactor Only When Green
❌ **WRONG**: Refactor while tests are failing
✅ **CORRECT**: Only refactor when all tests pass

### Rule 4: One Test at a Time
❌ **WRONG**: Write 10 tests, then implement all
✅ **CORRECT**: Write one test, implement, repeat

### Rule 5: Test Behavior, Not Implementation
❌ **WRONG**: Test private methods, implementation details
✅ **CORRECT**: Test public behavior, observable outcomes

---

## Test Order (Outside-In)

### 1. Acceptance Tests (First)
**Purpose**: Define what the feature should do
**Focus**: User-visible behavior
**Example**: API endpoint returns correct response

### 2. Application Tests (Second)
**Purpose**: Define use case orchestration
**Focus**: Application service behavior
**Example**: AppService calls correct domain methods

### 3. Domain Tests (Third)
**Purpose**: Define business rules
**Focus**: Aggregate behavior
**Example**: Wallet enforces balance invariants

---

## Common TDD Mistakes to Avoid

### ❌ Mistake 1: Implementation First
**Wrong**: Write domain code, then write tests
**Correct**: Write failing test, then write domain code

### ❌ Mistake 2: Testing Implementation
**Wrong**: Test that private method was called
**Correct**: Test that observable behavior occurred

### ❌ Mistake 3: Skipping Refactoring
**Wrong**: Leave messy code after Green phase
**Correct**: Always refactor while tests protect you

### ❌ Mistake 4: Writing Too Many Tests at Once
**Wrong**: Write 10 tests, implement all, then run
**Correct**: Write one test, implement, verify, repeat

### ❌ Mistake 5: Ignoring Failing Tests
**Wrong**: Leave failing tests for later
**Correct**: Never commit failing tests

---

## TDD Checklist

### Before Starting Implementation
- [ ] Requirements understood
- [ ] Acceptance criteria defined
- [ ] API contract designed
- [ ] Test environment setup

### During Red Phase
- [ ] Test written FIRST
- [ ] Test RUNS and FAILS
- [ ] Failure reason understood

### During Green Phase
- [ ] MINIMUM code written
- [ ] Test PASSES
- [ ] No extra features added

### During Blue Phase
- [ ] Code refactored
- [ ] ALL tests still pass
- [ ] No behavior changes

### Before Committing
- [ ] All tests pass
- [ ] No test skipped
- [ ] Code reviewed
- [ ] Documentation updated

---

## TDD for Different Layers

### Domain Layer TDD

```csharp
// 1. Write failing test
[Fact]
public void Should_Credit_Wallet_Balance()
{
    var wallet = Wallet.CreateNew("user-123");
    wallet.Credit(1000);
    wallet.AvailableBalance.ShouldBe(1000);  // Will fail - Credit() doesn't exist
}

// 2. Write minimum implementation
public void Credit(decimal amount)
{
    AvailableBalance += amount;  // Minimum code
}

// 3. Refactor with business rules
public void Credit(decimal amount)
{
    CheckRule(new CreditAmountMustBePositiveRule(amount));
    CheckRule(new WalletNotDeletedRule(this));
    AvailableBalance += amount;
    AddLocalEvent(new BalanceChangedDomainEvent(...));
}
```

### Application Layer TDD

```csharp
// 1. Write failing test
[Fact]
public async Task Should_Credit_Wallet()
{
    var input = new CreditDto { UserId = "user-123", Amount = 1000 };
    await _walletAppService.CreditAsync(input);
    // Will fail - CreditAsync doesn't exist
}

// 2. Write minimum implementation
[UnitOfWork]
public async Task CreditAsync(CreditDto input)
{
    var wallet = await _walletRepository.FindByUserIdAsync(input.UserId);
    wallet.Credit(input.Amount);
    await _walletRepository.UpdateAsync(wallet);
}

// 3. Refactor with error handling
[UnitOfWork]
public async Task CreditAsync(CreditDto input)
{
    var wallet = await _walletRepository.FindByUserIdAsync(input.UserId);
    
    if (wallet == null)
        throw new BusinessException(WalletNotFound);
    
    wallet.Credit(input.Amount);
    await _walletRepository.UpdateAsync(wallet);
}
```

### API Layer TDD

```csharp
// 1. Write failing test
[Fact]
public async Task Should_Call_Credit_Endpoint()
{
    var response = await _client.PostAsync("/api/v1/wallets/credit", 
        new StringContent(JsonConvert.SerializeObject(dto)));
    response.StatusCode.ShouldBe(HttpStatusCode.OK);  // Will fail - endpoint doesn't exist
}

// 2. Write minimum implementation
[HttpPost("credit")]
public async Task<IActionResult> Credit([FromBody] CreditDto input)
{
    await _walletAppService.CreditAsync(input);
    return Ok();
}

// 3. Refactor with proper response
[HttpPost("credit")]
[ProducesResponseType(StatusCodes.Status200OK)]
[ProducesResponseType(StatusCodes.Status400BadRequest)]
public async Task<IActionResult> Credit([FromBody] CreditDto input)
{
    try
    {
        await _walletAppService.CreditAsync(input);
        return Ok(new { success = true });
    }
    catch (BusinessException ex)
    {
        return BadRequest(new { error = ex.Message });
    }
}
```

---

## Quick Reference

### "I'm implementing X, where do I start?"

| What | First Step | Test Type |
|------|-----------|-----------|
| **New Aggregate** | Write failing domain test | Domain test |
| **New Use Case** | Write failing application test | Application test |
| **New API** | Write failing acceptance test | Acceptance test |
| **New Business Rule** | Write failing rule test | Domain test |

---

### "My test is failing, what do I do?"

| Failure Type | Solution |
|-------------|----------|
| **Test doesn't compile** | Write minimum interface/stub |
| **Test throws exception** | Write minimum implementation |
| **Test assertion fails** | Fix implementation logic |
| **Test passes unexpectedly** | Verify test is correct, add assertion |

---

## Success Criteria

### TDD Implementation Complete When:
- ✅ Every test was written BEFORE implementation
- ✅ Every test failed initially (Red phase)
- ✅ Every test now passes (Green phase)
- ✅ Code has been refactored (Blue phase)
- ✅ All tests still pass after refactoring
- ✅ No tests are skipped or ignored
- ✅ Documentation updated

---

**Playbook Version**: 1.0  
**Last Updated**: 2026-07-27  
**Maintained By**: Architecture Team  
**Enforced By**: Wion Engineering Rules (DDD Rule 3)