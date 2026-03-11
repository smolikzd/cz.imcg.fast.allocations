# Technical Specifications

This directory contains technical specifications for changes to the ZFI_PROCESS framework.

## Current Specifications

### SPEC-001: Instance Lifecycle State Validation
**Status:** Draft  
**Priority:** HIGH  
**Target Repository:** `cz.imcg.fast.planner`

**Problem:** The `execute()` method has no validation of instance status, allowing execution on RUNNING, COMPLETED, FAILED, or CANCELLED instances. This causes:
- Concurrent execution risks
- Data corruption from re-executing completed processes  
- Incorrect API usage (bypassing restart() method)
- Audit trail corruption (overwriting timestamps)

**Solution:** Add guard clause validation to `execute()` method to only allow execution from NEW status (with special bypass for restart() via iv_start_from_step parameter).

**Files:**
- [SPEC-001-instance-lifecycle-validation.md](./SPEC-001-instance-lifecycle-validation.md) - Full technical specification
- [SPEC-001-state-diagram.md](./SPEC-001-state-diagram.md) - State transition diagrams and validation logic

**Estimated Effort:** 2 hours  
**Constitution Compliance:** Principles II, III, V

---

## Specification Template

New specifications should follow the template in `_bmad/bmm/workflows/bmad-quick-flow/quick-spec/tech-spec-template.md`.

Required sections:
- Problem Statement
- Solution  
- Scope (In/Out)
- Constitution Compliance
- Implementation Plan with Tasks
- Acceptance Criteria
- Testing Strategy
