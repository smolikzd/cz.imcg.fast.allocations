# Architecture Documentation

This directory contains architecture documentation for the ZFI_PROCESS framework and Fast Allocations implementation.

## Available Documents

### Process Instance Lifecycle

**File**: [`process-instance-lifecycle.md`](./process-instance-lifecycle.md)  
**Version**: 1.0.0  
**Date**: 2026-03-11  
**Status**: Final

Complete documentation of the process instance lifecycle in the ZFI_PROCESS framework, including:

- All 5 instance status values (NEW, RUNNING, COMPLETED, FAILED, CANCELLED)
- Complete state transition diagram
- All lifecycle methods with signatures and behavior
  - Factory methods: `create()`, `load()`
  - Execution methods: `execute()`, `restart()`, `cancel()`
  - Query methods: `get_status()`
  - Configuration methods: `remove_step()`
- Status validation rules for each method
- Code examples for all common scenarios
- Integration with Process Manager
- Error handling patterns

**Target Audience**: Developers, architects, technical leads

---

## Document Categories

### Lifecycle & State Management
- [`process-instance-lifecycle.md`](./process-instance-lifecycle.md) - Process instance lifecycle documentation

### Architecture Decision Records (ADRs)
*No ADRs yet - will be added as architectural decisions are made*

### Design Patterns
*To be added - will document framework design patterns (factory, template method, etc.)*

### Integration Guides
*To be added - will document integration with SAP modules (FI, CO, PS, etc.)*

---

## Related Documentation

- **Technical Specifications**: [`../specs/`](../specs/)
- **Framework Constitution**: [`../../_bmad/_memory/constitution.md`](../../_bmad/_memory/constitution.md)
- **Project Status**: [`../PROJECT-STATUS.md`](../PROJECT-STATUS.md)
- **Cross-Repository Workflow**: [`../CROSS-REPO-WORKFLOW.md`](../CROSS-REPO-WORKFLOW.md)

---

**Maintained By**: ZFI_PROCESS Framework Team  
**Review Cycle**: Quarterly
