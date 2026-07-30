# ADR: TDD Violation - Code-First Implementation

**Status**: Accepted  
**Date**: 2026-07-27  
**Context**: wion-billing service implementation  
**Decision**: Document and prevent TDD violations

---

## Context

During the implementation of Payment, CreditTransaction, Ledger, and Invoice domains, code was implemented **before tests were written**, violating **DDD Rule 3 (Outside-In TDD)**.

### Discovery

The violation was discovered during knowledge base audit when:
- Implementation was marked as "IMPLEMENTED" in domains/README.md
- Tests existed but were written AFTER code
- No TDD workflow documentation existed
- Missing TDD playbook and templates

### What Happened

**Incorrect Flow (What happened)**:
```
1. Domain code implemented (Payment.cs, CreditTransaction.cs, etc.)
2. Domain tests written after implementation
3. Application code implemented
4. Application tests written after implementation
5. API code implemented
```

**Correct TDD Flow (What should have happened)**:
```
1. Write FAILING domain tests (🔴 RED)
2. Implement MINIMUM domain code (🟢 GREEN)
3. Refactor domain code (🔵 BLUE)
4. Write FAILING application tests (🔴 RED)
5. Implement MINIMUM application code (🟢 GREEN)
6. Write FAILING API tests (🔴 RED)
7. Implement MINIMUM API code (🟢 GREEN)
```

---

## Problem

### Violated Rules

1. **DDD Rule 3 (Outside-In TDD)**: "Never implement before tests exist"
2. **Wion Engineering Rule #3**: "Think Before Coding"
3. **Wion Engineering Rule #16**: "Never Stop Halfway"

### Root Causes

1. **No TDD Documentation**: TDD playbook didn't exist
2. **Missing Templates**: No TDD workflow templates
3. **Lack of Enforcement**: No mechanism to prevent code-first approach
4. **Knowledge Gap**: TDD principles not clearly documented

### Impact

- ❌ Tests didn't drive design
- ❌ Business rules not tested first
- ❌ Missing edge case coverage
- ❌ Tests verify implementation, not behavior
- ❌ No failing test phase (Red phase)

---

## Decision

### Immediate Actions

#### 1. Document TDD Violation
- ✅ Create this ADR documenting the violation
- ✅ Update domains/README.md to note TDD violation
- ✅ Add violation warnings to affected domain docs

#### 2. Create TDD Documentation
- ✅ Create TDD playbook (`templates/tdd-playbook.md`)
- ✅ Create missing templates (use-case.md, api.md, policy.md, new-domain.md)
- ✅ Update playbooks to enforce TDD workflow

#### 3. Remediation Strategy
- 🔴 **Critical**: Future work MUST follow TDD
- 🟡 **Important**: Existing code can remain (working code)
- 🟢 **Optional**: Rewrite domains with proper TDD (time-permitting)

### Long-term Prevention

#### 1. Engineering Rules Update
- Add explicit TDD enforcement to CLAUDE.md
- Add TDD checklist to Definition of Done
- Require TDD playbook reference in playbooks

#### 2. Process Changes
- Code reviews MUST verify TDD compliance
- New features MUST start with failing tests
- CI/CD SHOULD verify test-first commits

#### 3. Education
- TDD training for team members
- Pair programming on TDD workflow
- Code review focus on TDD compliance

---

## Implementation

### Phase 1: Documentation Updates ✅ COMPLETED

**Completed Actions**:
1. Created TDD playbook (`templates/tdd-playbook.md`)
2. Created missing templates:
   - `templates/new-domain.md`
   - `templates/use-case.md`
   - `templates/api.md`
   - `templates/policy.md`
3. Updated domain status from "PLANNED" to "IMPLEMENTED"
4. Fixed technology stack documentation (Node.js → C#)
5. Updated API status documentation

**Files Modified**:
- `.claude/templates/tdd-playbook.md` (created)
- `.claude/templates/new-domain.md` (created)
- `.claude/templates/use-case.md` (created)
- `.claude/templates/api.md` (created)
- `.claude/templates/policy.md` (created)
- `.claude/knowledge/wion-billing/README.md` (fixed tech stack)
- `.claude/knowledge/wion-billing/domains/README.md` (updated status)
- `.claude/knowledge/wion-billing/api/README.md` (updated status)

### Phase 2: TDD Violation Markers 🔴 IN PROGRESS

**Add violation notices to affected domains**:

```markdown
## Implementation Status
✅ **IMPLEMENTED** - Fully implemented with comprehensive tests

**⚠️ TDD Violation Note**: This domain was implemented using code-first approach 
instead of TDD. Future domains MUST follow proper TDD workflow. 
See [TDD Playbook](../../templates/tdd-playbook.md) for correct workflow.
```

**Affected Domains**:
- Payment Domain
- CreditTransaction Domain
- Ledger Domain
- Invoice Domain

### Phase 3: Future Enforcement 🟢 PLANNED

**Planned Actions**:
1. Update CLAUDE.md with explicit TDD requirements
2. Add TDD checklist to playbooks
3. Create TDD compliance check in code reviews
4. Add TDD validation to CI/CD pipeline

---

## Consequences

### Positive Outcomes
1. **Better Documentation**: TDD workflow now clearly documented
2. **Complete Templates**: All required templates created
3. **Awareness**: Team educated on proper TDD approach
4. **Prevention**: Mechanisms in place to prevent future violations

### Negative Outcomes
1. **Technical Debt**: Code-first implementation remains
2. **Test Quality**: Tests may not cover edge cases adequately
3. **Design**: Test-driven design benefits not realized

### Trade-offs
- **Keep existing code**: Working functionality, less time investment
- **Rewrite with TDD**: Proper TDD compliance, time-intensive
- **Decision**: Keep existing code, enforce TDD for new work

---

## Status

### Completed ✅
- [x] TDD violation documented
- [x] Root causes identified
- [x] TDD playbook created
- [x] Missing templates created
- [x] Technology stack fixed
- [x] Domain status updated

### In Progress 🔴
- [ ] Add TDD violation markers to affected domains
- [ ] Update playbooks with TDD enforcement
- [ ] Create TDD compliance checklist

### Planned 🟢
- [ ] Rewrite domains with proper TDD (optional)
- [ ] Add CI/CD TDD validation
- [ ] Team TDD training

---

## Alternatives Considered

### Alternative 1: Rewrite All Violations
**Proposal**: Rewrite Payment, CreditTransaction, Ledger, and Invoice domains with proper TDD

**Pros**:
- Proper TDD compliance
- Better test coverage
- Test-driven design benefits

**Cons**:
- Time-intensive (estimated 2-3 weeks)
- Risk of introducing bugs
- Delays new feature work

**Decision**: Rejected in favor of fixing current issues and enforcing TDD for future work

### Alternative 2: Ignore Violation
**Proposal**: Accept code-first implementation as valid approach

**Pros**:
- No additional work
- Existing code works

**Cons**:
- Violates Wion Engineering Rules
- Sets bad precedent
- Misses TDD benefits

**Decision**: Rejected - violates core engineering principles

### Alternative 3: Hybrid Approach (ACCEPTED)
**Proposal**: Document violation, fix current issues, enforce TDD for future work

**Pros**:
- Acknowledges violation
- Fixes documentation gaps
- Prevents future violations
- Minimal time investment

**Cons**:
- Leaves some technical debt
- Existing code not TDD-compliant

**Decision**: Accepted - best balance of corrective action and future prevention

---

## Lessons Learned

### What Went Wrong
1. **No TDD Documentation**: Team lacked clear TDD guidance
2. **Missing Templates**: No workflow templates to follow
3. **Process Gap**: No enforcement mechanism for TDD
4. **Knowledge Gap**: TDD importance not emphasized

### What Went Right
1. **Discovery**: Audit caught the violation
2. **Documentation**: Comprehensive documentation created
3. **Transparency**: Violation acknowledged and documented
4. **Prevention**: Mechanisms in place for future

### Prevention Strategies
1. **Always Start with Tests**: Never write code without failing test
2. **Use TDD Playbook**: Reference `templates/tdd-playbook.md` for every feature
3. **Code Review Focus**: Reviewers verify TDD compliance
4. **Documentation First**: Understand requirements before coding

---

## Related Documents

### Wion Engineering Rules
- [CLAUDE.md - DDD Rule 3](../../CLAUDE.md) - Outside-In TDD requirement
- [CLAUDE.md - Rule #3](../../CLAUDE.md) - Think Before Coding
- [CLAUDE.md - Rule #16](../../CLAUDE.md) - Never Stop Halfway

### TDD Documentation
- [TDD Playbook](../../templates/tdd-playbook.md) - Complete TDD workflow
- [Testing Conventions](../../shared/testing-conventions.md) - Test patterns
- [DDD Conventions](../../shared/ddd-conventions.md) - Domain testing

### Affected Domains
- [Payment Domain](../domains/payment/overview.md) - Implemented with TDD violation
- [CreditTransaction Domain](../domains/credit-transaction/overview.md) - Implemented with TDD violation
- [Ledger Domain](../domains/ledger/overview.md) - Implemented with TDD violation
- [Invoice Domain](../domains/invoice/overview.md) - Implemented with TDD violation

---

## Next Steps

### Immediate (This Week)
1. **Add TDD violation markers** to affected domain documentation
2. **Update new-feature playbook** with TDD enforcement
3. **Create TDD checklist** for code reviews

### Short-term (This Month)
1. **Team training** on proper TDD workflow
2. **Code review focus** on TDD compliance
3. **CI/CD integration** for TDD validation

### Long-term (Next Quarter)
1. **Consider rewriting** TDD-violation domains (if time permits)
2. **Establish TDD metrics** and reporting
3. **Continuous improvement** of TDD process

---

**ADR Version**: 1.0  
**Last Updated**: 2026-07-27  
**Maintained By**: Architecture Team  
**Status**: Accepted - Implementation in progress