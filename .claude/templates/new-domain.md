# New Domain Template

**Purpose**: Template for creating new domain documentation following DDD conventions.

**When to Use**: When implementing a new aggregate/domain concept.

---

## Domain Structure

Every new domain must have these files:

```
domains/{domain}/
├── overview.md           # Domain purpose and scope
├── aggregate.md          # Aggregate root and entities
├── model.md              # Value objects and types
├── business-rules.md     # Business invariants
├── lifecycle.md          # State transitions
├── domain-events.md     # Published events
└── repositories.md      # Repository interfaces (if applicable)
```

---

## Template: overview.md

```markdown
# {Domain Name} Domain

## Purpose
{What this domain handles and why it exists}

## Scope
The {Domain Name} domain encompasses:
- {Key capability 1}
- {Key capability 2}
- {Key capability 3}

## Bounded Context
{Domain Name} operates within the **{Bounded Context}** bounded context, handling {domain responsibilities}.

## Ubiquitous Language
- **{Domain Concept 1}**: {Definition}
- **{Domain Concept 2}**: {Definition}
- **{Domain Concept 3}**: {Definition}

## Related Documents
- [{Domain Name} Aggregate](./aggregate.md) - Complete Aggregate definition
- [{Domain Name} Model](./model.md) - Entities and Value Objects
- [{Domain Name} Business Rules](./business-rules.md) - Business invariants
- [{Domain Name} Lifecycle](./lifecycle.md) - State transitions
- [{Domain Name} Domain Events](./domain-events.md) - Events published
```

---

## Template: aggregate.md

```markdown
# {Domain Name} Aggregate

## Purpose
{What this aggregate manages using DDD principles}

## Aggregate Root
**`{AggregateRootName}`** - The root entity that provides access to the {Domain Name} Aggregate

**Base Classes**: 
- `{BillingAggregateRoot}` (extends `{TMTFullAuditedAggregateRoot<long>}`) - Provides audit trail and soft delete infrastructure

## Entities

### {AggregateRootName} (Aggregate Root)
**Purpose**: {What the root entity represents}

**Responsibilities**:
- {Responsibility 1}
- {Responsibility 2}
- {Responsibility 3}
- {Responsibility 4}

**Key Operations**:

#### Public Methods
- `CreateNew({parameters})` - Static factory method to create new {aggregate}
- `{Operation1}({parameters})` - {What it does}
- `{Operation2}({parameters})` - {What it does}

#### Internal Methods (used by domain logic)
- `{InternalMethod1}({parameters})` - {What it does}
- `{InternalMethod2}({parameters})` - {What it does}

**Domain Events Published**:
- `{Event1Name}` - {When it's published}
- `{Event2Name}` - {When it's published}
- `{Event3Name}` - {When it's published}

---

### {Entity1}
**Purpose**: {What this entity represents}

**Responsibilities**:
- {Responsibility 1}
- {Responsibility 2}

**Internal Operations**:
- `{Operation}({parameters})` - {What it does}

## Value Objects

### {ValueObject1}
**Purpose**: {Value object classification}

**Properties**:
- `{Property1}` - {Description}
- `{Property2}` - {Description}
- `{Property3}` - {Description}

**{ValueObject} Catalog**:
| Code | Name | {Additional Column} | Description |
|-----|-----|---------------------|-------------|
| {CODE1} | {Name1} | {Value} | {Description} |
| {CODE2} | {Name2} | {Value} | {Description} |

## Business Invariants

See [{Domain Name} Business Rules](./business-rules.md) for complete business rules:

### Core Invariants
- `{Invariant1}` - {Description}
- `{Invariant2}` - {Description}
- `{Invariant3}` - {Description}

### Operational Rules
- {Operational rule 1}
- {Operational rule 2}
- {Operational rule 3}

### {Operations} Rules
- **{Operation1}**: {Effect description}
- **{Operation2}**: {Effect description}
- **{Operation3}**: {Effect description}

## Lifecycle

See [{Domain Name} Lifecycle](./lifecycle.md) for complete lifecycle management:

**States**:
- **{State1}**: {Description}
- **{State2}**: {Description}

**Transitions**:
- {Transition 1}
- {Transition 2}

## Domain Events

See [{Domain Name} Domain Events](./domain-events.md) for complete event definitions:

**{Event Category} Events**:
- `{Event1Name}({parameters})` - {When it's published}
- `{Event2Name}({parameters})` - {When it's published}

## Repositories

See [{Domain Name} Repositories](./repositories.md) for repository interfaces:

**I{DomainName}Repository** - Main repository for {AggregateRootName} aggregate
- Inherits `IRepository<{AggregateRootName}>` from ABP Framework
- Provides domain-specific query methods:
  - `{QueryMethod1}({parameters})` - {What it does}
  - `{QueryMethod2}({parameters})` - {What it does}

## Specifications

### Business Rules (Domain Rules)
The {AggregateRootName} aggregate enforces business rules through rule objects:

- `{Rule1Name}` - {What it validates}
- `{Rule2Name}` - {What it validates}
- `{Rule3Name}` - {What it validates}

## Transaction Boundary

**{Transaction scope description}**

All operations on a {AggregateRootName} Aggregate must be performed within a single database transaction to maintain consistency.

### ABP Framework Unit of Work

The ABP Framework automatically manages transactions around application service methods via the `[UnitOfWork]` attribute:

```csharp
[UnitOfWork]
public async Task {Operation}Async({parameters})
{
    var {aggregateVar} = await _{domainRepository}.{QueryMethod}Async({parameters});
    {aggregateVar}.{Operation}({parameters});
    await _{domainRepository}.UpdateAsync({aggregateVar});
    // Transaction automatically commits on successful completion
}
```

### Transaction Rules

**DO**:
- {Best practice 1}
- {Best practice 2}
- {Best practice 3}

**DON'T**:
- {Anti-pattern 1}
- {Anti-pattern 2}
- {Anti-pattern 3}

### Consistency Boundaries

**Strong Consistency** (within {AggregateRootName} aggregate):
- {What's strongly consistent}

**Eventual Consistency** (across aggregates):
- {What's eventually consistent}

## Related Documents
- [{Domain Name} Model](./model.md) - Entities and Value Objects detailed
- [{Domain Name} Business Rules](./business-rules.md) - Business invariants
- [{Domain Name} Lifecycle](./lifecycle.md) - State management
- [{Domain Name} Domain Events](./domain-events.md) - Published events
- [{Domain Name} Repositories](./repositories.md) - Repository interfaces
- [{Domain Name} Overview](./overview.md) - Domain context and scope
```

---

## Template: business-rules.md

```markdown
# {Domain Name} Business Rules

## Purpose
Business invariants and rules that the {AggregateRootName} aggregate must enforce.

## Rule Format

Each business rule follows this format:

### BR-{LETTER}-{NUMBER}: {Rule Name}
**Rule**: {Human-readable description}
**Formal Definition**: {Mathematical/logical definition}
**Enforcement Points**: {Where enforced}
**Violation Handling**: {What happens on violation}
**Purpose**: {Why this rule exists}

---

## Core Business Rules

### BR-{LETTER}-001: {Rule Name}
**Rule**: {Human-readable description}
**Formal Definition**: {Mathematical representation}
**Enforcement Points**: 
- {Where it's enforced}
- {Method that enforces it}

**Violation Handling**: 
- Exception thrown: {Exception type}
- Event published: {Event name (if any)}
- User message: {Error message}

**Purpose**: {Business justification}

---

### BR-{LETTER}-002: {Rule Name}
**Rule**: {Human-readable description}
**Formal Definition**: {Mathematical representation}
**Enforcement Points**: 
- {Where it's enforced}
- {Method that enforces it}

**Violation Handling**: 
- Exception thrown: {Exception type}
- Event published: {Event name (if any)}

**Purpose**: {Business justification}

---

## Operational Rules

### {Operation} Rules

#### BR-{LETTER}-003: {Rule Name}
**Rule**: {Human-readable description}
**Formal Definition**: {Mathematical/logical definition}
**Enforcement Points**: 
- {Operation}({parameters}) method
- {Additional enforcement points}

**Violation Handling**: 
- Exception thrown: {Exception type}

**Purpose**: {Business justification}

---

## State Transition Rules

### {State1} → {State2} Transition

#### BR-{LETTER}-004: {Rule Name}
**Rule**: {Conditions required for transition}
**Formal Definition**: {Logical conditions}
**Enforcement Points**: 
- {Method} that performs transition

**Violation Handling**: 
- Exception thrown: {Exception type}
- State remains: {State1}

**Purpose**: {Business justification}

---

## Cross-Aggregate Rules

**Note**: Rules involving multiple aggregates should be implemented as **Policies**, not as domain business rules.

See [Policies](../policies/README.md) for cross-aggregate coordination.

---

## Rule Enforcement

### Enforcement Locations

Business rules are enforced at:
1. **Domain Layer** - Primary enforcement in aggregates
2. **Application Layer** - Use case validation
3. **Database Layer** - Constraint validation (defense in depth)

### Enforcement Pattern

```csharp
public void {Operation}({parameters})
{
    // Rule 1: {Rule name}
    CheckRule(new {Rule1}({parameters}));
    
    // Rule 2: {Rule name}
    CheckRule(new {Rule2}({parameters}));
    
    // Perform operation
    {OperationLogic};
    
    // Publish domain event
    AddLocalEvent(new {Event}(...));
}
```

---

## Testing Business Rules

### Rule Testing Pattern

```csharp
[Fact]
public void Should_Enforce_{RuleName}_When_{Condition}()
{
    // Arrange
    var {aggregate} = {Aggregate}.CreateNew({parameters});
    
    // Act & Assert
    Should.Throw<BusinessRuleValidationException>(() =>
    {
        {aggregate}.{Operation}({invalid_parameters});
    });
}
```

---

## Related Documents
- [{Domain Name} Aggregate](./aggregate.md) - Aggregate definition
- [{Domain Name} Lifecycle](./lifecycle.md) - State transitions
- [DDD Conventions](../../shared/ddd-conventions.md) - Business rule format
```

---

## Template: domain-events.md

```markdown
# {Domain Name} Domain Events

## Purpose
Events that the {AggregateRootName} aggregate publishes to notify other parts of the system about state changes.

## Event Format

All domain events follow this structure:

```csharp
public class {EventName} : IDomainEvent
{
    public Guid EventId { get; }
    public DateTime OccurredAt { get; }
    
    // Event-specific data
    public {DataType} {PropertyName} { get; }
    
    public {EventName}({parameters})
    {
        EventId = Guid.NewGuid();
        OccurredAt = DateTime.UtcNow;
        {Property} = {value};
    }
}
```

---

## Published Events

### {EventCategory} Events

#### {Event1Name}
**Purpose**: {What this event signifies}

**Published By**: 
- {Operation}({parameters}) method

**Event Data**:
| Property | Type | Description |
|----------|------|-------------|
| {Property1} | {Type} | {Description} |
| {Property2} | {Type} | {Description} |
| {Property3} | {Type} | {Description} |

**Consumers**:
- {Consumer1} - {Why it consumes this event}
- {Consumer2} - {Why it consumes this event}

**Example**:
```csharp
// When published
var wallet = Wallet.CreateNew("user-123");
// Event published: WalletCreatedDomainEvent

// Event data
{
    "EventId": "guid-here",
    "OccurredAt": "2026-07-27T10:00:00Z",
    "WalletId": 123,
    "UserId": "user-123"
}
```

---

#### {Event2Name}
**Purpose**: {What this event signifies}

**Published By**: 
- {Operation}({parameters}) method

**Event Data**:
| Property | Type | Description |
|----------|------|-------------|
| {Property1} | {Type} | {Description} |
| {Property2} | {Type} | {Description} |

**Consumers**:
- {Consumer1} - {Why it consumes this event}
- {Consumer2} - {Why it consumes this event}

---

## Event Publishing Pattern

### Standard Pattern

```csharp
public void {Operation}({parameters})
{
    // Enforce business rules
    CheckRule(new {BusinessRule}({parameters}));
    
    // Perform operation
    {OperationLogic};
    
    // Publish domain event
    AddLocalEvent(new {Event}(
        {EventData}
    ));
}
```

---

## Event Handling

### Local Event Handlers

**Local events** are handled within the same bounded context:

```csharp
public class {Event}Handler : ILocalEventHandler<{Event}>
{
    public async Task HandleEventAsync({Event} eventData)
    {
        // Handle event
    }
}
```

### Distributed Event Handlers

**Distributed events** are handled across bounded contexts:

```csharp
public class {Event}Handler : IDistributedEventHandler<{Event}Eto>
{
    public async Task HandleEventAsync({Event}Eto eventData)
    {
        // Handle event from another service
    }
}
```

---

## Event Catalog

All events are cataloged in [Events Index](../events/README.md).

---

## Testing Domain Events

### Event Testing Pattern

```csharp
[Fact]
public void Should_Publish_{EventName}_When_{Condition}()
{
    // Arrange
    var {aggregate} = {Aggregate}.CreateNew({parameters});
    
    // Act
    {aggregate}.{Operation}({parameters});
    
    // Assert
    var domainEvents = {aggregate}.GetLocalEvents();
    var {event} = domainEvents.FirstOrDefault(e => e.EventData is {EventName});
    {event}.ShouldNotBeNull();
    
    // Verify event data
    var eventData = ({EventName}){event}.EventData;
    eventData.{Property}.ShouldBe({expectedValue});
}
```

---

## Related Documents
- [{Domain Name} Aggregate](./aggregate.md) - Aggregate operations
- [{Domain Name} Lifecycle](./lifecycle.md) - State transitions
- [Event Conventions](../../shared/event-conventions.md) - Event standards
- [Events Catalog](../events/README.md) - All service events
```

---

## Template: lifecycle.md

```markdown
# {Domain Name} Lifecycle

## Purpose
State transitions and lifecycle management for the {AggregateRootName} aggregate.

## States

### {State1}
**Description**: {What this state represents}

**Entry Conditions**:
- {Condition1}
- {Condition2}

**Allowed Operations**:
- {Operation1}
- {Operation2}

**Exit Conditions**:
- {Condition1}

**Transitions To**:
- {State2} (via {TransitionMethod})
- {State3} (via {TransitionMethod})

---

### {State2}
**Description**: {What this state represents}

**Entry Conditions**:
- {Condition1}
- {Condition2}

**Allowed Operations**:
- {Operation1}
- {Operation2}

**Exit Conditions**:
- {Condition1}

**Transitions To**:
- {State3} (via {TransitionMethod})

---

## State Transitions

### {State1} → {State2}

**Trigger**: {What causes this transition}

**Method**: `{TransitionMethod}({parameters})`

**Preconditions**:
- {Precondition1}
- {Precondition2}

**Postconditions**:
- {Postcondition1}
- {Postcondition2}

**Business Rules**:
- {Rule1}
- {Rule2}

**Example**:
```csharp
// Before
var {aggregate} = {Aggregate}.CreateNew({params});
// State: {State1}

// Transition
{aggregate}.{TransitionMethod}({params});
// State: {State2}
```

---

### {State2} → {State3}

**Trigger**: {What causes this transition}

**Method**: `{TransitionMethod}({parameters})`

**Preconditions**:
- {Precondition1}

**Postconditions**:
- {Postcondition1}

**Business Rules**:
- {Rule1}

---

## State Machine Diagram

```mermaid
stateDiagram-v2
    [*] --> {State1}
    {State1} --> {State2}: {Trigger1}
    {State2} --> {State3}: {Trigger2}
    {State3} --> {State1}: {Trigger3}
    {State3} --> [*]
```

---

## Lifecycle Rules

### Creation
**Initial State**: {InitialState}

**Creation Method**: `{FactoryMethod}({parameters})`

**Creation Rules**:
- {Rule1}
- {Rule2}

**Creation Events**:
- {Event1}

---

### Deletion
**Final State**: {FinalState}

**Deletion Method**: `{DeletionMethod}()`

**Deletion Rules**:
- {Rule1}
- {Rule2}

**Deletion Events**:
- {Event1}

---

## State Validation

### State Checking Pattern

```csharp
public class {State}Specification
{
    public bool IsSatisfiedBy({Aggregate} aggregate)
    {
        return aggregate.{StateProperty} == {ExpectedState};
    }
}
```

### State Transition Validation

```csharp
public void {TransitionMethod}({parameters})
{
    // Validate current state
    CheckRule(new {CurrentStateRule}(this.{StateProperty}));
    
    // Validate transition conditions
    CheckRule(new {TransitionConditionRule}({parameters}));
    
    // Perform transition
    {StateProperty} = {NewState};
    
    // Publish event
    AddLocalEvent(new {StateChangedEvent}(...));
}
```

---

## Testing Lifecycle

### State Transition Testing Pattern

```csharp
[Fact]
public void Should_Transition_From_{OldState}_To_{NewState}_When_{Condition}()
{
    // Arrange
    var {aggregate} = {Aggregate}.CreateNew({params});
    // Verify initial state
    {aggregate}.{StateProperty}.ShouldBe({OldState});
    
    // Act
    {aggregate}.{TransitionMethod}({params});
    
    // Assert
    {aggregate}.{StateProperty}.ShouldBe({NewState});
}
```

### Invalid State Transition Testing

```csharp
[Fact]
public void Should_Fail_When_Transitioning_From_{InvalidState}()
{
    // Arrange
    var {aggregate} = {Aggregate}.CreateNew({params});
    {aggregate}.{SetStateMethod}({InvalidState});
    
    // Act & Assert
    Should.Throw<BusinessRuleValidationException>(() =>
    {
        {aggregate}.{TransitionMethod}({params});
    });
}
```

---

## Related Documents
- [{Domain Name} Aggregate](./aggregate.md) - Aggregate operations
- [{Domain Name} Business Rules](./business-rules.md) - Business invariants
- [{Domain Name} Domain Events](./domain-events.md) - Lifecycle events
```

---

## Usage Instructions

### Creating a New Domain

1. **Copy this template** to `domains/{new-domain}/`
2. **Replace placeholders** with domain-specific content
3. **Follow DDD conventions** from shared documentation
4. **Update domains/README.md** with new domain
5. **Create implementation** following TDD workflow

### Placeholder Replacement

Replace these placeholders with actual content:
- `{Domain Name}` → Actual domain name
- `{AggregateRootName}` → Root entity name
- `{LETTER}` → Domain letter (W, P, C, L, I, etc.)
- `{parameters}` → Actual parameters
- `{Operation}` → Actual operation name

---

**Template Version**: 1.0  
**Last Updated**: 2026-07-27  
**Based On**: DDD Conventions and Wion Engineering Rules