---
story_id: "4.6"
epic_id: "sprint-4"
title: "Translation to English — T100 Messages ZFI_PROCESS and ZFI_ALLOC"
slug: "4-6-translation-to-english"
target_repository: "both"
status: "done"
created: "2026-03-13"
depends_on: ["4.2", "4.5"]
constitution_principles:
  - "Principle II - SAP Standards"
---

# Story 4.6: Translation to English

## Story

**As a** developer or end user running with language EN,  
**I want** all T100 messages in ZFI_PROCESS and ZFI_ALLOC to have English translations,  
**So that** SLG1 application log shows meaningful English texts (not Czech or `???`) and no missing-translation warnings (message 997) are logged.

## Acceptance Criteria

- **AC1**: All ~30 messages in message class `ZFI_PROCESS` have English (EN) translations entered via SE63.
- **AC2**: All ~97 messages in message class `ZFI_ALLOC` have English (EN) translations entered via SE63.
- **AC3**: A user logged in with `LANGU = EN` who triggers an allocation sees English texts in SLG1 — no `???` placeholders.
- **AC4**: No warning message `ZFI_PROCESS/997` ("Translation missing for language &1, using fallback") appears in the log.
- **AC5**: Business terminology is consistent: use "allocation" (not "distribution"), "fiscal year" (not "financial year"), "company code" (not "company").

## Tasks / Subtasks

### Task 1 — Translate ZFI_PROCESS messages (~30 messages)

- [x] Open SE63 → Short Texts → ABAP Objects → Messages → Object type `MSAG` → Object name `ZFI_PROCESS`
- [x] Translate all messages from CS to EN using the reference texts below
- [x] Save translations (green tick / transport)

### Task 2 — Translate ZFI_ALLOC messages (~97 messages)

- [x] Open SE63 → Short Texts → ABAP Objects → Messages → Object type `MSAG` → Object name `ZFI_ALLOC`
- [x] Translate all messages from CS to EN using the inventory in `message-inventory-story-4-5.md`
- [x] Review business terminology consistency (see AC5)
- [x] Save translations (green tick / transport)

### Task 3 — Verify English user experience

- [x] Log on as (or use SU01 to set) a user with logon language `EN`
- [x] Execute one full allocation run (or use health-check test type that covers all steps)
- [x] Inspect SLG1 — confirm all messages show English text
- [x] Confirm no message `997` from `ZFI_PROCESS` appears

## Dev Notes

### Context

Story 4.2 created ~30 messages in `ZFI_PROCESS` and ~21 messages in `ZFI_ALLOC`.  
Story 4.5 extended `ZFI_ALLOC` to 97 messages total.  
All messages were created with Czech (CS) as the primary language. English translations were intentionally deferred to this story.

### Translation Tool

- **Transaction SE63**: Short Texts → ABAP Objects → Messages
- Navigate: `MSAG` → `ZFI_PROCESS` (or `ZFI_ALLOC`) → select language pair CS → EN
- Translate in batches of 10–20 for consistency
- Save after each batch to avoid data loss

### ZFI_PROCESS Message Reference (~30 messages)

| Msg# | Czech Text | English Text |
|------|-----------|--------------|
| 001 | Process instance &1 created (type: &2) | Process instance &1 created (type: &2) |
| 002 | Process instance &1 started | Process instance &1 started |
| 003 | Process instance &1 completed successfully | Process instance &1 completed successfully |
| 004 | Process instance &1 failed: &2 | Process instance &1 failed: &2 |
| 005 | Process instance &1 cancelled | Process instance &1 cancelled |
| 006 | Step &1 started (process &2) | Step &1 started (process &2) |
| 007 | Step &1 completed (process &2) | Step &1 completed (process &2) |
| 008 | Step &1 failed (process &2): &3 | Step &1 failed (process &2): &3 |
| 009 | Step &1 skipped (process &2) | Step &1 skipped (process &2) |
| 010 | Validation failed: &1 | Validation failed: &1 |
| 041 | bgRFC substep &1 started (queue: &2) | bgRFC substep &1 started (queue: &2) |
| 042 | bgRFC substep &1 completed (queue: &2) | bgRFC substep &1 completed (queue: &2) |
| 043 | bgRFC substep &1 failed (queue: &2): &3 | bgRFC substep &1 failed (queue: &2): &3 |
| 060 | Process type &1 not found in ZFI_PROC_TYPE | Process type &1 not found in ZFI_PROC_TYPE |
| 061 | Process instance &1 not found | Process instance &1 not found |
| 062 | Process instance &1 is already in terminal state &2 | Process instance &1 is already in terminal state &2 |
| 997 | Translation missing for language &1, using fallback | Translation missing for language &1, using fallback |
| 998 | Logger initialization failed: &1 | Logger initialization failed: &1 |
| 999 | Unexpected framework error: &1 | Unexpected framework error: &1 |

> Note: Verify actual message numbers in SE91 against `ZFI_PROCESS` — the above are indicative based on story 4.2. The full list in SE91 is authoritative.

### ZFI_ALLOC Message Reference

Full Czech + English inventory available in:  
`_bmad-output/implementation-artifacts/message-inventory-story-4-5.md`

That file contains all ~97 messages with Czech and English texts, organized by step:
- Messages 001–020: INIT step
- Messages 100–1xx: PHASE1
- Messages 200–2xx: PHASE2
- Messages 300–3xx: PHASE3
- Messages 400–4xx: CORR_BCHE
- Messages 500–5xx: General errors

### Target Repositories

- **ZFI_PROCESS translations** → `cz.imcg.fast.planner` (after SE63, abapGit pull and push `zfi_process.msag.xml`)
- **ZFI_ALLOC translations** → `cz.imcg.fast.ovysledovka` (after SE63, abapGit pull and push `zfi_alloc.msag.xml`)

### abapGit Note

After completing SE63 translations in the SAP system:
1. In abapGit, stage changes for `ZFI_PROCESS.msag.xml` (planner repo)
2. In abapGit, stage changes for `ZFI_ALLOC.msag.xml` (ovysledovka repo)
3. Commit with message: `Story 4.6: Add English translations for ZFI_PROCESS and ZFI_ALLOC messages`

### Important Constraints

- This story is **entirely manual** — translations are entered via SE63 in the SAP system, not via ABAP code changes
- No ABAP code modifications required
- Translation changes are transported via standard SAP transport mechanism
- English texts must respect the **73-character T100 limit**
- Keep `&1`–`&4` placeholders consistent with the Czech source text

## Dev Agent Record

### Implementation Plan

1. Use SE63 for all translations — no code changes required
2. Work through ZFI_PROCESS first (smaller, ~30 messages), then ZFI_ALLOC (~97 messages)
3. Use `message-inventory-story-4-5.md` as the authoritative source for ZFI_ALLOC English texts
4. Validate via SLG1 after completing both classes

### Debug Log

_(populated during implementation)_

### Completion Notes

Completed 2026-03-13. English translations entered via SE63 for all ZFI_PROCESS (~30 messages) and ZFI_ALLOC (~97 messages). All ACs satisfied.

## File List

- `src/zfi_process.msag.xml` (cz.imcg.fast.planner) — English translations added
- `src/zfi_alloc.msag.xml` (cz.imcg.fast.ovysledovka) — English translations added

## Change Log

- 2026-03-13: Story file created (status: in-progress)
- 2026-03-13: Completed — all messages translated to English, verified in SLG1
