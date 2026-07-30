# Credit Transaction Domain

## Purpose
Handle credit consumption, refunds, and adjustments with idempotency support.

## Implementation Status

✅ **IMPLEMENTED** - Fully implemented with comprehensive tests

**⚠️ TDD Violation Note**: This domain was implemented using a code-first approach instead of proper Test-Driven Development. While comprehensive tests exist, they were written after the implementation rather than driving the design. 

**Future domains MUST follow proper TDD workflow**: See [TDD Playbook](../../../templates/tdd-playbook.md) for the correct Outside-In TDD approach.

**See**: [ADR: TDD Violation Fix](../adr/tdd-violation-fix.md) for details about the violation and remediation strategy.

## Related Documents
- [Credit Transaction Aggregate](./aggregate.md)
- [Credit Transaction Model](./model.md)
- [Credit Transaction Business Rules](./business-rules.md)
