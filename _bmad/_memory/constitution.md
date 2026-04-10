# ZFI_PROCESS Framework Constitution

## Core Principles

### I. DDIC-First Architecture

All type definitions MUST use SAP Data Dictionary (DDIC) objects rather than local type definitions. Every table type, structure, data element, and domain SHALL be defined in DDIC and properly transported.

**Rationale**: DDIC objects ensure reusability across programs, proper transportation between systems, eliminate generic table type errors in method signatures, and provide centralized documentation. Local types create maintenance debt and transportation issues.

**Non-Negotiable Rules**:
- NO local TYPE definitions for structures or table types in class definitions or programs
- ALL method signatures MUST use DDIC table types (e.g., ZFI_PROCESS_TT_PROC_STEP)
- ALL fields MUST be based on data elements, which MUST be based on domains
- Naming convention: ZFI_PROCESS prefix for domains/data elements/tables, ZFI_PROCESS_TT_ for table types

### II. SAP Standards Compliance

All code MUST follow SAP naming conventions, ABAP development standards, and proper object structure. Code SHALL be organized following SAP package structure with proper separation of DDIC objects, classes, interfaces, and programs.

**Rationale**: SAP standards ensure code is maintainable by any ABAP developer, passes SAP Code Inspector checks, and integrates properly with SAP transport system and abapGit.

**Non-Negotiable Rules**:
- Naming: ZCL_FI_PROCESS (classes), ZIF_FI_PROCESS (interfaces), ZCX_FI_PROCESS (exceptions), ZFI_PROCESS (programs/tables)
- All public methods MUST have ABAP-Doc comments with @parameter and @raising annotations
- All database tables MUST be client-dependent (include MANDT field)
- All changes MUST be transportable via abapGit XML format
- Factory pattern MUST be used for object instantiation (private constructors)

### III. Consult SAP Documentation (Non-Negotiable)

Before making ANY changes to ABAP code, data dictionary objects, or framework structure, the developer MUST consult the mcp-sap-docs resource. This applies to new features, bug fixes, optimizations, and refactoring.

**Rationale**: SAP systems have complex dependencies, version-specific behaviors, and best practices that are not intuitive. Consulting official documentation prevents introducing subtle bugs, performance issues, or incompatibilities. Trial-and-error with SAP APIs (e.g., BALI, RAP, CDS) wastes time and introduces bugs.

**Non-Negotiable Rules**:
- MUST search mcp-sap-docs BEFORE implementing any SAP standard API (BALI, RAP, CDS, etc.)
- MUST search mcp-sap-docs BEFORE implementing any DDIC change
- MUST verify ABAP syntax compatibility for target SAP version (7.58 for this project)
- MUST check SAP Help Portal for recommended patterns when implementing standard SAP patterns
- MUST validate against SAP Community solutions when troubleshooting errors
- MUST verify correct method names, parameters, and return types from official API documentation
- As specified in AGENTS.md: "all change consult with mcp-sap-docs"

**Example Workflow (BALI Application Log API)**:
1. Before implementing logging: Query mcp-sap-docs for "BALI application log IF_BALI_MESSAGE_GETTER"
2. Read official documentation for class/interface structure
3. Use documented methods (e.g., `GET_ALL_VALUES`, not guessed `get_message_detail()`)
4. Review existing framework code for consistent API usage patterns
5. Only after consulting docs: Implement and test

**Lesson Learned**: Story 4.4 health check implementation required multiple iterations due to guessing BALI API methods instead of consulting documentation first. Official SAP Help Portal documentation (https://help.sap.com/docs/SAP_S4HANA_CLOUD/...) is the single source of truth.

### IV. Factory Pattern & Encapsulation

All complex classes MUST use factory methods for instantiation. Constructors SHALL be private or protected. Direct object instantiation with NEW is prohibited for framework classes.

**Rationale**: Factory pattern enables proper initialization, validation, and future extensibility. It prevents incomplete object construction and provides a single point of control for object creation.

**Non-Negotiable Rules**:
- Framework classes (ZCL_FI_PROCESS_MANAGER, ZCL_FI_PROCESS_INSTANCE, ZCL_FI_PROCESS_DEFINITION) MUST have private constructors
- Public factory methods (e.g., GET_INSTANCE, CREATE) MUST handle initialization and validation
- Singleton pattern MUST be used for manager classes
- Step implementation classes MAY use public constructors (as they implement ZIF_FI_PROCESS_STEP)

### V. Error Handling & Observability

All business logic errors MUST raise the custom exception class ZCX_FI_PROCESS_ERROR with appropriate message IDs and context. All process execution MUST log status changes, timestamps, and user actions for audit trail.

**Rationale**: Centralized exception handling ensures consistent error messages, proper error propagation, and integration with SAP message system. Audit trail is required for compliance and debugging.

**Non-Negotiable Rules**:
- MUST use TRY-CATCH blocks around all step execution
- MUST raise ZCX_FI_PROCESS_ERROR (not CX_ROOT or other generic exceptions)
- MUST populate message class ZFI_PROCESS with descriptive error messages
- PREFFER crash loud principle, unless silent exception is part of algorithm
- MUST log: created_by, changed_by, started_by, timestamps for all process and step records
- Error messages MUST include context (step number, process type, instance ID)

## Development Standards

### Code Formatting

**Line Length** (Non-Negotiable):
- Maximum line length: **120 characters** for ABAP coding with .abap extension (enforced by abaplint.json)
- Absolute maximum: **255 characters** (ABAP parser limit - causes "field unknown" errors)
- **MUST** wrap long lines at logical boundaries (parameters, field assignments, conditions)

**Rationale**: ABAP parser fails with cryptic errors ("Field X is unknown") when lines exceed 255 characters. Lines over 120 characters reduce readability and violate SAP Code Inspector recommendations.

**Mandatory Wrapping Patterns**:
```abap
" ✅ CORRECT: Wrap VALUE constructor
DATA(ls_ctx) = VALUE ty_structure(
  field1 = value1
  field2 = value2
  field3 = value3
).

" ✅ CORRECT: Wrap method calls
update_status(
  iv_param1 = value1
  iv_param2 = value2
  iv_param3 = value3
).

" ✅ CORRECT: Wrap RAISE EXCEPTION
RAISE EXCEPTION TYPE zcx_error
  EXPORTING
    textid = zcx_error=>error_id
    value  = |Long message here|.

" ✅ CORRECT: Wrap FOR loops with WHERE
lt_result = VALUE #(
  FOR item IN lt_source
  WHERE ( condition1 = value1
      AND condition2 = value2 )
  ( item )
).

" ❌ WRONG: Single line over 120 chars
DATA(ls_ctx) = VALUE ty_structure( field1 = value1 field2 = value2 field3 = value3 field4 = value4 field5 = value5 ).
```

### Code Documentation

- Public methods MUST include ABAP-Doc comments with:
  - One-line description starting with "!
  - @parameter annotations for all parameters
  - @raising annotations for all exceptions
- Complex business logic MUST include inline comments explaining WHY, not WHAT
- Each class MUST have a purpose statement at the top

**ABAP Doc comment placement (Non-Negotiable)**:
- `"!` comments are ONLY allowed directly before an **individual declaration** (`METHODS`, `CLASS-METHODS`, `DATA`, `CLASS-DATA`) in the DEFINITION section
- `"!` is **NOT valid** before compound block declarations such as `CONSTANTS: BEGIN OF ... END OF` — use regular `"` there
- `"!` inside a method IMPLEMENTATION body is a **syntax error**
- Use regular `"` for all comments inside IMPLEMENTATION (method bodies, SQL blocks, inline) and before `CONSTANTS` blocks
- Violation produces: `ABAP Doc comment is in the wrong position.` or `"#" is not allowed here. "." is expected.`

**Inter-method comments (Non-Negotiable)**:
- Comments MUST NOT appear between `ENDMETHOD.` and the next `METHOD` at class IMPLEMENTATION level
- The only valid content between two methods is blank lines
- Any `"` or `"!` comment placed at class level (outside a method body) produces: `The class contains unknown comments which can't be stored.`
- If a comment describes the next method, move it **inside** that method as its first line
- This applies to ALL files: never leave section headers, phase labels, or group separators between methods

### Database Conventions

- All tables MUST include: MANDT, key fields, created_by, created_at, changed_by, changed_at
- Status fields MUST use domains with fixed values (not free text)
- Foreign key relationships MUST be defined in DDIC
- Indexes MUST be defined for query performance on large tables

### Constants & Configuration

- Status values MUST be defined as constants in relevant classes (gc_status structure)
- Magic numbers and strings are prohibited (use named constants)
- Process configuration MUST be stored in customizing tables (ZFI_PROC_TYPE, ZFI_PROC_DEF)

## Quality Assurance

### Testing Requirements

- Demo/test programs MUST be provided (e.g., ZFI_PROCESS_FRAMEWORK)
- Setup programs for test data MUST be available (e.g., ZFI_SETUP_DEMO_DATA)
- Critical changes MUST be tested with demo data before transport

### Error Scenarios

All error scenarios MUST be handled:
- Process type not found
- Step class instantiation failure
- Step execution failure with/without retry
- Invalid status transitions
- Restart from failed step
- Rollback requirements

### Code Review Gates

Before considering code complete:
- [ ] DDIC objects defined (no local types)
- [ ] ABAP-Doc present on all public methods
- [ ] Factory pattern used appropriately
- [ ] ZCX_FI_PROCESS_ERROR raised with proper message IDs
- [ ] Audit fields populated (created_by, timestamps)
- [ ] Naming conventions followed
- [ ] **Line length** ≤ 120 characters (CRITICAL: ≤ 255 absolute limit to avoid parser errors)
- [ ] Long statements wrapped at logical boundaries (parameters, fields, conditions)
- [ ] Test program updated
- [ ] mcp-sap-docs consulted for any new patterns

## Governance

### Amendment Procedure

Changes to this constitution require:
1. Documented justification explaining why the change is necessary
2. Review of impact on existing code and templates
3. Update of all dependent templates (plan-template.md, spec-template.md, tasks-template.md)
4. Communication to all developers
5. Version increment following semantic versioning

### Versioning Policy

- **MAJOR** (X.0.0): Principle removed or fundamentally redefined (breaking change)
- **MINOR** (0.X.0): New principle added or section materially expanded
- **PATCH** (0.0.X): Clarifications, wording improvements, non-semantic fixes

### Compliance Review

- All feature specifications MUST include "Constitution Check" section (per plan-template.md)
- All task lists MUST reference constitution principles when applicable
- Code reviews MUST verify compliance with all Core Principles
- Complexity that violates principles MUST be justified in plan documentation

### Exception Process

If a principle MUST be violated for valid technical reasons:
1. Document in plan.md "Complexity Tracking" section
2. Explain why the principle cannot be followed
3. Document what simpler alternatives were considered and rejected
4. Require explicit approval from technical lead
5. Add compensating controls or documentation

### Guidance Files

Runtime development guidance is maintained in:
- `learnings/PROJECT_CONVENTIONS.md` - Detailed coding patterns and examples
- `learnings/TPOOL_FORMAT.md` - Text pool XML formatting
- `learnings/QUEUE_NAMING.md` - Substep mode configuration
- `AGENTS.md` - AI agent guidance (consult mcp-sap-docs requirement)

These guidance files MUST stay aligned with constitution principles.

**Version**: 1.0.2 | **Ratified**: 2025-11-10 | **Last Amended**: 2026-04-10
