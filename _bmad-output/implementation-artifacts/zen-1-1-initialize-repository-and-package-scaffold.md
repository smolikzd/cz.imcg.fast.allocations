# Story 1.1: Initialize repository and package scaffold

Status: review

## Story

As a developer,
I want the `cz.en.orch` repository configured with abapGit structure, abaplint config, and the ZEN_ORCH package definition,
So that all subsequent DDIC objects and classes have a proper, transportable home.

## Acceptance Criteria

1. `src/zen_orch/package.devc.xml` exists defining package `ZEN_ORCH`
2. `.abaplint.json` is present and enforces a 120-character line length limit
3. `.gitignore` excludes standard non-source SAP artifacts
4. `src/zen_orch/.constitution.md` is present (copied from planning repo via sync-constitution.sh)
5. `README.md` describes the repository purpose and package structure

## Tasks / Subtasks

- [x] Create git repository `cz.en.orch` on GitHub (AC: all)
  - [x] Initialize repo locally at `/Users/smolik/DEV/cz.en.orch`
  - [x] Add remote origin to GitHub (smolikzd/cz.en.orch)
  - [x] Create initial commit with README stub
- [x] Configure abapGit structure (AC: 1)
  - [x] Create `src/zen_orch/` directory
  - [x] Create `src/zen_orch/package.devc.xml` for package `ZEN_ORCH`
- [x] Add abaplint configuration (AC: 2)
  - [x] Create `.abaplint.json` with 120-char max line length rule (copy/adapt from planner repo)
- [x] Add `.gitignore` (AC: 3)
  - [x] Exclude standard SAP non-source artifacts (`.acl.xml`, `.apj.xml`, etc.)
- [x] Sync constitution (AC: 4)
  - [x] Add `cz.en.orch` to `repos/registry.md` in planning repo
  - [x] Update `sync-constitution.sh` to include the new repo path
  - [x] Run `./repos/sync-constitution.sh` to copy `.constitution.md` to `src/zen_orch/.constitution.md`
- [x] Write README.md (AC: 5)
  - [x] Document repository purpose (ZEN_ORCH process orchestration engine)
  - [x] Document package structure overview
  - [x] Document how to register a new adapter (pointer to adapter interface)
  - [x] Document Phase 1 / Phase 2 scope boundary

## Dev Notes

### Target Repository

All work in this story is in the **new** `cz.en.orch` repository at `/Users/smolik/DEV/cz.en.orch`.

The planning repository (`cz.imcg.fast.allocations`) receives only two changes:
- `repos/registry.md` — add `cz.en.orch` entry
- `repos/sync-constitution.sh` — add new repo path
- `_bmad-output/implementation-artifacts/sprint-status.yaml` — story status update

### Package Definition (`package.devc.xml`)

abapGit serializes SAP packages as `package.devc.xml`. The file structure for a development package:

```xml
<?xml version="1.0" encoding="utf-8"?>
<abapGit version="v1.0.0" serializer="LCL_OBJECT_DEVC" serializer_version="v1.0.0">
 <asx:abap xmlns:asx="http://www.sap.com/abapxml" version="1.0">
  <asx:values>
   <DEVC>
    <CTEXT>ZEN_ORCH Process Orchestration Engine</CTEXT>
    <DLVUNIT>LOCAL</DLVUNIT>
    <PDEVCLASS>$TMP</PDEVCLASS>
    <KORRFLAG>X</KORRFLAG>
    <PACKTYPE>D</PACKTYPE>
   </DEVC>
  </asx:values>
 </asx:abap>
</abapGit>
```

Adjust `DLVUNIT` to the correct software component once transport layer is known. `PACKTYPE=D` is development package.

### abaplint Configuration (`.abaplint.json`)

The planner repository (`cz.imcg.fast.planner`) uses abaplint with 120-char line limit. Copy its `.abaplint.json` and adjust the `src` path. Minimum required rule:

```json
{
  "global": {
    "files": "/src/**",
    "exclude": []
  },
  "syntax": {
    "version": "v758"
  },
  "rules": {
    "line_length": {
      "length": 120
    }
  }
}
```

Reference planner repo for the complete rule set — do not omit rules already established there.

### .gitignore

Standard SAP abapGit `.gitignore`. Exclude at minimum:
- `*.bak`
- `.DS_Store`
- abapGit transport request files if generated locally

Reference `.gitignore` from `cz.imcg.fast.planner` as template.

### sync-constitution.sh Update

The script at `repos/sync-constitution.sh` copies `_bmad/_memory/constitution.md` to each code repo. Add entry for `cz.en.orch`:

```bash
TARGET_REPOS=(
  "/Users/smolik/DEV/cz.imcg.fast.planner/src/.constitution.md"
  "/Users/smolik/DEV/cz.imcg.fast.ovysledovka/src/.constitution.md"
  "/Users/smolik/DEV/cz.en.orch/src/zen_orch/.constitution.md"   # NEW
)
```

Run the script after adding the entry to verify the copy succeeds.

### repos/registry.md Entry

Add the following row to the repository registry table:

| Repository | Remote | Purpose | Constitution |
|------------|--------|---------|--------------|
| `cz.en.orch` | https://github.com/smolikzd/cz.en.orch.git | Orchestration Engine (ZEN_ORCH) | Synced to `src/zen_orch/.constitution.md` |

### README.md Content Outline

```markdown
# cz.en.orch — ZEN_ORCH Process Orchestration Engine

## Purpose
ZEN_ORCH is a framework-agnostic ABAP orchestration engine ...

## Package Structure
Package: ZEN_ORCH
Prefix:  ZCL_EN_ORCH_* / ZIF_EN_ORCH_* / ZCX_EN_ORCH_*

## Phase Scope
Phase 1: Foundation + Engine + Adapters (this repository)
Phase 2: BSR + Dashboard (deferred)

## How to Register a New Adapter
1. Implement ZIF_EN_ORCH_ADAPTER in your package
2. Insert a row into ZEN_ORCH_ADAPTER_REG: ADAPTER_TYPE = 'MY_TYPE', IMPL_CLASS = 'ZCL_MY_ADAPTER'
3. The engine instantiates your class via the factory — no orchestrator changes needed

## Running Tests
- ABAP Unit: run ZCL_EN_ORCH_ENGINE test class
- Integration: execute program ZEN_ORCH_TEST (uses MOCK adapter)

## Architecture
See docs/ folder or planning repository architecture.md for full design decisions.
```

### Architecture Key Points for This Story

This story is **infrastructure only** — no DDIC objects, no ABAP classes. The deliverable is:
1. A properly structured abapGit-compatible directory tree
2. abaplint configured to enforce 120-char limit (Constitution Principle II)
3. Constitution file present (Constitution Principle I — sets the rules for all subsequent stories)

**Activation order context:** This story covers layer 0 (repository infrastructure) before the 12-layer DDIC activation sequence begins in Story 1.2.

### Constitution Compliance

| Principle | Application |
|-----------|-------------|
| Principle I — DDIC-First | No DDIC objects in this story; establishes the foundation for them |
| Principle II — SAP Standards | abaplint 120-char rule enforced from day one |
| Principle III — Consult SAP Docs | N/A — no SAP APIs used in this story |
| Principle IV — Factory Pattern | N/A — no classes |
| Principle V — Error Handling | N/A — no classes |

### Project Structure Notes

- **Target repository:** `/Users/smolik/DEV/cz.en.orch` (new repo, separate from planner/ovysledovka)
- **Package name:** `ZEN_ORCH` — must be registered in SAP system with this exact name
- **abapGit path:** `src/zen_orch/` — all objects in this path, matching package name lowercase
- **No compile-time dependency** on `ZFI_PROCESS` or `ZFI_ALLOC_PROCESS` — enforced by repo isolation
- **Constitution location:** `src/zen_orch/.constitution.md` (convention from planner: `.constitution.md` inside src package folder)

### Cross-Story Dependencies

- This story has **no dependencies** — it is the foundation for all subsequent stories
- Stories 1.2–1.4 depend on this story (package must exist before DDIC objects can be created)
- Epic 2–6 all transitively depend on this story

### References

- Architecture: `_bmad-output/planning-artifacts/architecture.md` § "Project Structure & Boundaries" — complete directory listing
- Architecture: `_bmad-output/planning-artifacts/architecture.md` § "Activation Order" — 12-layer build sequence
- Architecture: `_bmad-output/planning-artifacts/architecture.md` § D1 (Repository decision), D2 (zero dependency decision)
- Epics: `_bmad-output/planning-artifacts/epics.md` § Story 1.1 — acceptance criteria
- Planning repo sync script: `repos/sync-constitution.sh`
- Planning repo registry: `repos/registry.md`
- Planner repo reference: `/Users/smolik/DEV/cz.imcg.fast.planner` — reference for `.abaplint.json` and `.gitignore` templates

## Dev Agent Record

### Agent Model Used

github-copilot/claude-sonnet-4.6

### Debug Log References

No issues encountered. The `cz.en.orch` repository already existed locally with the GitHub remote
(`smolikzd/cz.en.orch`) configured but had no commits. Implementation proceeded from that clean state.

The `sync-constitution.sh` script was refactored to accept a custom relative target path as a third
argument (previously hardcoded to `src/.constitution.md`), enabling the `src/zen_orch/.constitution.md`
target path for the new repo.

### Completion Notes List

- Repository `cz.en.orch` initialized with 2 commits (2334a37, 8af2217)
- `src/zen_orch/package.devc.xml` defines `ZEN_ORCH` development package (PACKTYPE=D, DLVUNIT=LOCAL)
- `.abaplint.json` copies full rule set from `cz.imcg.fast.planner`, with `line_length` promoted from
  `Off` to `Error` (length 120) per Constitution Principle II
- `.gitignore` mirrors planner repo pattern plus `.acl.xml` / `.apj.xml` SAP exclusions
- `sync-constitution.sh` updated to accept optional target path parameter; `cz.en.orch` added as third
  target with path `src/zen_orch/.constitution.md`; sync verified successful (10932 bytes written)
- `repos/registry.md` updated with `cz.en.orch` entry including purpose, convention, and constitution path
- `README.md` documents: purpose, package naming conventions, phase scope boundary, adapter registration
  steps, test execution, and constitution reference — all AC5 requirements satisfied
- No ABAP code generated in this story (infrastructure only, layer 0)
- All ACs validated: package.devc.xml ✅, .abaplint.json 120-char ✅, .gitignore ✅, .constitution.md ✅, README.md ✅

## File List

### cz.en.orch (new repository)

- `README.md` — repository overview, purpose, package structure, adapter registration, phase scope
- `.abaplint.json` — abaplint configuration with 120-char line length enforced (Error severity)
- `.gitignore` — macOS, editor, SAP artifact exclusions
- `src/zen_orch/package.devc.xml` — ZEN_ORCH DEVC package definition (abapGit format)
- `src/zen_orch/.constitution.md` — constitution synced from planning repo

### cz.imcg.fast.allocations (planning repository)

- `repos/registry.md` — added `cz.en.orch` entry
- `repos/sync-constitution.sh` — added `cz.en.orch` target, refactored to accept custom target path

## Change Log

- 2026-04-04: Story implemented — `cz.en.orch` repository scaffold created with abapGit structure,
  abaplint config (120-char line limit enforced), .gitignore, constitution synced, README written.
  Planning repo: `registry.md` and `sync-constitution.sh` updated to include new repo.
  Commits: 2334a37 (README stub), 8af2217 (scaffold files) in `cz.en.orch`.
