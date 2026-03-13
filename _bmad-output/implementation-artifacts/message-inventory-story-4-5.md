# Message Inventory for Story 4-5: Migrate Step Code Messages to T100

**Epic**: 4 - Message Class Migration  
**Story**: 4-5 - Migrate Step Code  
**Target Repository**: cz.imcg.fast.ovysledovka  
**Message Class**: ZFI_ALLOC  
**Date**: 2026-03-12  

---

## Overview

This document catalogs all hard-coded strings in the 5 allocation step classes that need to be converted to T100 messages. The existing message class ZFI_ALLOC already has messages 001-012 and 500-507.

**Message Number Ranges**:
- **001-099**: INIT step messages (001-012 already exist, can reuse where appropriate)
- **100-199**: PHASE1 step messages
- **200-299**: PHASE2 step messages
- **300-399**: PHASE3 step messages
- **400-499**: CORR_BCHE step messages

---

## 1. ZCL_FI_ALLOC_STEP_INIT

**File**: `src/zfi_alloc_process/zcl_fi_alloc_step_init.clas.abap`

| Line | Current Code | Msg# | English Text | Czech Text | Severity | Variables |
|------|--------------|------|--------------|------------|----------|-----------|
| 63 | `rs_result-message = \|Failed to initialize state row (force restart).\|` | 013 | Failed to initialize state row (force restart) | Selhala inicializace stavového řádku (vynucený restart) | E | - |
| 77 | `rs_result-message = \|Failed to insert state row.\|` | 014 | Failed to insert state row | Selhalo vložení stavového řádku | E | - |
| 85 | `rs_result-message = \|INIT: state row already exists — skipping.\|` | 015 | INIT: state row already exists — skipping | INIT: stavový řádek již existuje — přeskakuji | I | - |
| 91 | `rs_result-message = \|Process initialized\|` | 016 | Process initialized | Proces inicializován | S | - |
| 105-107 | `rs_result-message = \|Allocation ID { mv_allocation_id } already initialized for company code { mv_company_code }, fiscal year { mv_fiscal_year }, fiscal period { mv_fiscal_period }\|` | 017 | Allocation ID &1 already initialized for company &2, year &3, period &4 | Alokační ID &1 již inicializováno pro společnost &2, rok &3, období &4 | E | &1=allocation_id, &2=company_code, &3=fiscal_year, &4=fiscal_period |
| 113 | `rs_result-message = \|Allocation ID { mv_allocation_id } is ready\|` | 018 | Allocation ID &1 is ready | Alokační ID &1 je připraveno | S | &1=allocation_id |
| 128 | `value = 'Rollback failed: instance variables not initialized'` | 019 | Rollback failed: instance variables not initialized | Rollback selhal: instanční proměnné nejsou inicializovány | E | - |
| 155 | `value = 'INIT: serial step, execute_substep not supported'` | 020 | INIT: serial step, execute_substep not supported | INIT: sériový krok, execute_substep není podporováno | E | - |

---

## 2. ZCL_FI_ALLOC_STEP_PHASE1

**File**: `src/zfi_alloc_process/zcl_fi_alloc_step_phase1.clas.abap`

| Line | Current Code | Msg# | English Text | Czech Text | Severity | Variables |
|------|--------------|------|--------------|------------|----------|-----------|
| 65-66 | `value = \|ID alokace { mv_allocation_id } pro účetní období { mv_company_code }/{ mv_fiscal_year }/{ mv_fiscal_period } neexistuje\|` | 100 | Allocation ID &1 for period &2/&3/&4 does not exist | ID alokace &1 pro účetní období &2/&3/&4 neexistuje | E | &1=allocation_id, &2=company_code, &3=fiscal_year, &4=fiscal_period |
| 73-74 | `value = \|ID alokace { mv_allocation_id } pro účetní období { mv_company_code }/{ mv_fiscal_year }/{ mv_fiscal_period } je uzamčeno\|` | 101 | Allocation ID &1 for period &2/&3/&4 is locked | ID alokace &1 pro účetní období &2/&3/&4 je uzamčeno | E | &1=allocation_id, &2=company_code, &3=fiscal_year, &4=fiscal_period |
| 84-85 | `value = \|Stav ID alokace { mv_allocation_id } pro účetní období { mv_company_code }/{ mv_fiscal_year }/{ mv_fiscal_period } a fázi I není iniciální\|` | 102 | Status of allocation ID &1 for period &2/&3/&4 and phase I is not initial | Stav ID alokace &1 pro účetní období &2/&3/&4 a fázi I není iniciální | E | &1=allocation_id, &2=company_code, &3=fiscal_year, &4=fiscal_period |
| 116 | `MESSAGE s000(zfi_alloc) WITH \|Zpracování alokací - Phase I\|` | 103 | Processing allocations - Phase I | Zpracování alokací - Phase I | I | - |
| 119 | `MESSAGE s000(zfi_alloc) WITH \|Účetní okruh: { mv_company_code }\|` | 104 | Company code: &1 | Účetní okruh: &1 | I | &1=company_code |
| 122 | `MESSAGE s000(zfi_alloc) WITH \|Fiskální rok: { mv_fiscal_year }\|` | 105 | Fiscal year: &1 | Fiskální rok: &1 | I | &1=fiscal_year |
| 125 | `MESSAGE s000(zfi_alloc) WITH \|Fiskální období: { mv_fiscal_period }\|` | 106 | Fiscal period: &1 | Fiskální období: &1 | I | &1=fiscal_period |
| 128 | `MESSAGE s000(zfi_alloc) WITH \|ID alokace: { mv_allocation_id }\|` | 107 | Allocation ID: &1 | ID alokace: &1 | I | &1=allocation_id |
| 131 | `MESSAGE s000(zfi_alloc) WITH \|Značka: { '' }\|` | 108 | Brand: &1 | Značka: &1 | I | &1=brand |
| 134 | `MESSAGE s000(zfi_alloc) WITH \|Hier1: { '' }\|` | 109 | Hier1: &1 | Hier1: &1 | I | &1=hier1 |
| 138 | `MESSAGE s000(zfi_alloc) WITH \|Režim přepsání dat: ANO\|` | 110 | Data overwrite mode: YES | Režim přepsání dat: ANO | I | - |
| 142 | `MESSAGE s000(zfi_alloc) WITH \|---\|` | 111 | ----------------------------------- | ----------------------------------- | I | - |
| 145 | `MESSAGE s000(zfi_alloc) WITH \|Výmaz haviček a položek základní tabulky\|` | 112 | Deleting headers and items from base table | Výmaz hlaviček a položek základní tabulky | I | - |
| 148 | `MESSAGE s000(zfi_alloc) WITH \|Získání unikátních klíčů z účetních dokladů\|` | 113 | Fetching unique keys from accounting documents | Získání unikátních klíčů z účetních dokladů | I | - |
| 162 | `MESSAGE s000(zfi_alloc) WITH \|Počet unikátních klíčů: { lv_base_keys_lines }\|` | 114 | Number of unique keys: &1 | Počet unikátních klíčů: &1 | I | &1=count |
| 165 | `MESSAGE s000(zfi_alloc) WITH \|Ukládání klíčů základní tabulky do cache\|` | 115 | Saving base table keys to cache | Ukládání klíčů základní tabulky do cache | I | - |
| 209 | `MESSAGE s000(zfi_alloc) WITH \|Vytváření cache klíčů základní tabulky dokončeno\|` | 116 | Base table key cache creation completed | Vytváření cache klíčů základní tabulky dokončeno | S | - |
| 214 | `rs_result-message = \|Phase 1 completed\|` | 117 | Phase 1 completed | Phase 1 dokončena | S | - |
| 225 | `rs_result-message = 'PHASE1: ALLOCATION_ID is missing'` | 118 | PHASE1: ALLOCATION_ID is missing | PHASE1: ALLOCATION_ID chybí | E | - |
| 232 | `rs_result-message = 'PHASE1: COMPANY_CODE is missing'` | 119 | PHASE1: COMPANY_CODE is missing | PHASE1: COMPANY_CODE chybí | E | - |
| 239 | `rs_result-message = 'PHASE1: FISCAL_YEAR is missing'` | 120 | PHASE1: FISCAL_YEAR is missing | PHASE1: FISCAL_YEAR chybí | E | - |
| 246 | `rs_result-message = 'PHASE1: FISCAL_PERIOD is missing'` | 121 | PHASE1: FISCAL_PERIOD is missing | PHASE1: FISCAL_PERIOD chybí | E | - |
| 250 | `rs_result-message = 'PHASE1 validated successfully'` | 122 | PHASE1 validated successfully | PHASE1 validována úspěšně | S | - |
| 264 | `value = 'Rollback failed: instance variables not initialized'` | 123 | Rollback failed: instance variables not initialized | Rollback selhal: instanční proměnné nejsou inicializovány | E | - |
| 306 | `value = 'PHASE1: serial step, execute_substep not supported'` | 124 | PHASE1: serial step, execute_substep not supported | PHASE1: sériový krok, execute_substep není podporováno | E | - |

---

## 3. ZCL_FI_ALLOC_STEP_PHASE2

**File**: `src/zfi_alloc_process/zcl_fi_alloc_step_phase2.clas.abap`

| Line | Current Code | Msg# | English Text | Czech Text | Severity | Variables |
|------|--------------|------|--------------|------------|----------|-----------|
| 75-76 | `value = \|ID alokace { mv_allocation_id } pro účetní období { mv_company_code }/{ mv_fiscal_year }/{ mv_fiscal_period } neexistuje\|` | 200 | Allocation ID &1 for period &2/&3/&4 does not exist | ID alokace &1 pro účetní období &2/&3/&4 neexistuje | E | &1=allocation_id, &2=company_code, &3=fiscal_year, &4=fiscal_period |
| 83-84 | `value = \|ID alokace { mv_allocation_id } pro účetní období { mv_company_code }/{ mv_fiscal_year }/{ mv_fiscal_period } je uzamčeno\|` | 201 | Allocation ID &1 for period &2/&3/&4 is locked | ID alokace &1 pro účetní období &2/&3/&4 je uzamčeno | E | &1=allocation_id, &2=company_code, &3=fiscal_year, &4=fiscal_period |
| 94-95 | `value = \|Stav ID alokace { mv_allocation_id } pro účetní období { mv_company_code }/{ mv_fiscal_year }/{ mv_fiscal_period } a fázi II není iniciální\|` | 202 | Status of allocation ID &1 for period &2/&3/&4 and phase II is not initial | Stav ID alokace &1 pro účetní období &2/&3/&4 a fázi II není iniciální | E | &1=allocation_id, &2=company_code, &3=fiscal_year, &4=fiscal_period |
| 111 | `MESSAGE s000(zfi_alloc) WITH \|Zpracování alokací - Phase II\|` | 203 | Processing allocations - Phase II | Zpracování alokací - Phase II | I | - |
| 114 | `MESSAGE s000(zfi_alloc) WITH \|Účetní okruh: { mv_company_code }\|` | 204 | Company code: &1 | Účetní okruh: &1 | I | &1=company_code |
| 117 | `MESSAGE s000(zfi_alloc) WITH \|Fiskální rok: { mv_fiscal_year }\|` | 205 | Fiscal year: &1 | Fiskální rok: &1 | I | &1=fiscal_year |
| 120 | `MESSAGE s000(zfi_alloc) WITH \|Fiskální období: { mv_fiscal_period }\|` | 206 | Fiscal period: &1 | Fiskální období: &1 | I | &1=fiscal_period |
| 123 | `MESSAGE s000(zfi_alloc) WITH \|ID alokace: { mv_allocation_id }\|` | 207 | Allocation ID: &1 | ID alokace: &1 | I | &1=allocation_id |
| 127 | `MESSAGE s000(zfi_alloc) WITH \|Režim přepsání dat: ANO\|` | 208 | Data overwrite mode: YES | Režim přepsání dat: ANO | I | - |
| 131 | `MESSAGE s000(zfi_alloc) WITH \|-----------------------------------\|` | 209 | ----------------------------------- | ----------------------------------- | I | - |
| 133 | `MESSAGE s000(zfi_alloc) WITH \|Výmaz položek základní tabulky\|` | 210 | Deleting items from base table | Výmaz položek základní tabulky | I | - |
| 136 | `MESSAGE s000(zfi_alloc) WITH \|Získání unikátních klíčů z cache\|` | 211 | Fetching unique keys from cache | Získání unikátních klíčů z cache | I | - |
| 145 | `MESSAGE s000(zfi_alloc) WITH \|Počet unikátních klíčů: { lv_base_header_lines }\|` | 212 | Number of unique keys: &1 | Počet unikátních klíčů: &1 | I | &1=count |
| 147 | `MESSAGE s000(zfi_alloc) WITH \|Ukládání položek základní tabulky do cache\|` | 213 | Saving base table items to cache | Ukládání položek základní tabulky do cache | I | - |
| 213 | `rs_result-message = \|Phase 2 completed\|` | 214 | Phase 2 completed | Phase 2 dokončena | S | - |
| 224 | `rs_result-message = 'PHASE2: ALLOCATION_ID is missing'` | 215 | PHASE2: ALLOCATION_ID is missing | PHASE2: ALLOCATION_ID chybí | E | - |
| 231 | `rs_result-message = 'PHASE2: COMPANY_CODE is missing'` | 216 | PHASE2: COMPANY_CODE is missing | PHASE2: COMPANY_CODE chybí | E | - |
| 238 | `rs_result-message = 'PHASE2: FISCAL_YEAR is missing'` | 217 | PHASE2: FISCAL_YEAR is missing | PHASE2: FISCAL_YEAR chybí | E | - |
| 245 | `rs_result-message = 'PHASE2: FISCAL_PERIOD is missing'` | 218 | PHASE2: FISCAL_PERIOD is missing | PHASE2: FISCAL_PERIOD chybí | E | - |
| 249 | `rs_result-message = 'PHASE2 validated successfully'` | 219 | PHASE2 validated successfully | PHASE2 validována úspěšně | S | - |
| 268 | `value = 'Rollback failed: instance variables not initialized'` | 220 | Rollback failed: instance variables not initialized | Rollback selhal: instanční proměnné nejsou inicializovány | E | - |
| 286 | `value = 'Rollback failed: cannot deserialize substep context'` | 221 | Rollback failed: cannot deserialize substep context | Rollback selhal: nelze deserializovat kontext substep | E | - |
| 356 | `value = 'on_error failed: instance variables not initialized'` | 222 | on_error failed: instance variables not initialized | on_error selhal: instanční proměnné nejsou inicializovány | E | - |
| 373 | `value = 'Failed to update ZFI_ALLOC_STATE in on_error'` | 223 | Failed to update ZFI_ALLOC_STATE in on_error | Selhala aktualizace ZFI_ALLOC_STATE v on_error | E | - |
| 395 | `value = 'on_success failed: instance variables not initialized'` | 224 | on_success failed: instance variables not initialized | on_success selhal: instanční proměnné nejsou inicializovány | E | - |
| 412 | `value = 'Failed to update ZFI_ALLOC_STATE in on_success'` | 225 | Failed to update ZFI_ALLOC_STATE in on_success | Selhala aktualizace ZFI_ALLOC_STATE v on_success | E | - |
| 633 | `rs_result-message = \|PHASE2 substep completed successfully for key_id { ls_phase2_context-key_id }\|` | 226 | PHASE2 substep completed successfully for key_id &1 | PHASE2 substep dokončen úspěšně pro key_id &1 | S | &1=key_id |

---

## 4. ZCL_FI_ALLOC_STEP_PHASE3

**File**: `src/zfi_alloc_process/zcl_fi_alloc_step_phase3.clas.abap`

| Line | Current Code | Msg# | English Text | Czech Text | Severity | Variables |
|------|--------------|------|--------------|------------|----------|-----------|
| 69-70 | `value = \|ID alokace { mv_allocation_id } pro účetní období { mv_company_code }/{ mv_fiscal_year }/{ mv_fiscal_period } neexistuje\|` | 300 | Allocation ID &1 for period &2/&3/&4 does not exist | ID alokace &1 pro účetní období &2/&3/&4 neexistuje | E | &1=allocation_id, &2=company_code, &3=fiscal_year, &4=fiscal_period |
| 77-78 | `value = \|ID alokace { mv_allocation_id } pro účetní období { mv_company_code }/{ mv_fiscal_year }/{ mv_fiscal_period } je uzamčeno\|` | 301 | Allocation ID &1 for period &2/&3/&4 is locked | ID alokace &1 pro účetní období &2/&3/&4 je uzamčeno | E | &1=allocation_id, &2=company_code, &3=fiscal_year, &4=fiscal_period |
| 85-86 | `value = \|Stav ID alokace { mv_allocation_id } pro účetní období { mv_company_code }/{ mv_fiscal_year }/{ mv_fiscal_period } a fázi III není iniciální\|` | 302 | Status of allocation ID &1 for period &2/&3/&4 and phase III is not initial | Stav ID alokace &1 pro účetní období &2/&3/&4 a fázi III není iniciální | E | &1=allocation_id, &2=company_code, &3=fiscal_year, &4=fiscal_period |
| 101 | `MESSAGE s000(zfi_alloc) WITH \|Zpracování alokací - Phase III\|` | 303 | Processing allocations - Phase III | Zpracování alokací - Phase III | I | - |
| 104 | `MESSAGE s000(zfi_alloc) WITH \|Účetní okruh: { mv_company_code }\|` | 304 | Company code: &1 | Účetní okruh: &1 | I | &1=company_code |
| 107 | `MESSAGE s000(zfi_alloc) WITH \|Fiskální rok: { mv_fiscal_year }\|` | 305 | Fiscal year: &1 | Fiskální rok: &1 | I | &1=fiscal_year |
| 110 | `MESSAGE s000(zfi_alloc) WITH \|Fiskální období: { mv_fiscal_period }\|` | 306 | Fiscal period: &1 | Fiskální období: &1 | I | &1=fiscal_period |
| 113 | `MESSAGE s000(zfi_alloc) WITH \|ID alokace: { mv_allocation_id }\|` | 307 | Allocation ID: &1 | ID alokace: &1 | I | &1=allocation_id |
| 117 | `MESSAGE s000(zfi_alloc) WITH \|Režim přepsání dat: ANO\|` | 308 | Data overwrite mode: YES | Režim přepsání dat: ANO | I | - |
| 121 | `MESSAGE s000(zfi_alloc) WITH \|-----------------------------------\|` | 309 | ----------------------------------- | ----------------------------------- | I | - |
| 124 | `MESSAGE s000(zfi_alloc) WITH \|Výmaz alokačních položek\|` | 310 | Deleting allocation items | Výmaz alokačních položek | I | - |
| 127 | `MESSAGE s000(zfi_alloc) WITH \|Získání nepřiřazených položek (neúplný klíč)\|` | 311 | Fetching unassigned items (incomplete key) | Získání nepřiřazených položek (neúplný klíč) | I | - |
| 138 | `MESSAGE s000(zfi_alloc) WITH \|Počet nepřiřazených položek: { lv_unassigned_lines }\|` | 312 | Number of unassigned items: &1 | Počet nepřiřazených položek: &1 | I | &1=count |
| 142 | `MESSAGE s000(zfi_alloc) WITH \|Vytváření alokačních položek ...\|` | 313 | Creating allocation items ... | Vytváření alokačních položek ... | I | - |
| 286 | `MESSAGE s000(zfi_alloc) WITH \|Počet nenalezených záznamů v základní tabulce: { lv_bhnf }\|` | 314 | Number of records not found in base table: &1 | Počet nenalezených záznamů v základní tabulce: &1 | I | &1=count |
| 289 | `MESSAGE s000(zfi_alloc) WITH \|Počet duplicit v základní tabulce: { lv_bhmf }\|` | 315 | Number of duplicates in base table: &1 | Počet duplicit v základní tabulce: &1 | I | &1=count |
| 292 | `MESSAGE s000(zfi_alloc) WITH \|Počet chyb aktualizace tabulek: { lv_ewait }\|` | 316 | Number of table update errors: &1 | Počet chyb aktualizace tabulek: &1 | I | &1=count |
| 306 | `rs_result-message = \|PHASE3 completed. { lv_items_created } allocation items created.\|` | 317 | PHASE3 completed. &1 allocation items created | PHASE3 dokončena. &1 alokačních položek vytvořeno | S | &1=count |
| 317 | `rs_result-message = 'PHASE3: ALLOCATION_ID is missing'` | 318 | PHASE3: ALLOCATION_ID is missing | PHASE3: ALLOCATION_ID chybí | E | - |
| 324 | `rs_result-message = 'PHASE3: COMPANY_CODE is missing'` | 319 | PHASE3: COMPANY_CODE is missing | PHASE3: COMPANY_CODE chybí | E | - |
| 331 | `rs_result-message = 'PHASE3: FISCAL_YEAR is missing'` | 320 | PHASE3: FISCAL_YEAR is missing | PHASE3: FISCAL_YEAR chybí | E | - |
| 338 | `rs_result-message = 'PHASE3: FISCAL_PERIOD is missing'` | 321 | PHASE3: FISCAL_PERIOD is missing | PHASE3: FISCAL_PERIOD chybí | E | - |
| 342 | `rs_result-message = 'PHASE3 validated successfully'` | 322 | PHASE3 validated successfully | PHASE3 validována úspěšně | S | - |
| 356 | `value = 'Rollback failed: instance variables not initialized'` | 323 | Rollback failed: instance variables not initialized | Rollback selhal: instanční proměnné nejsou inicializovány | E | - |
| 406 | `value = 'PHASE3: serial step, execute_substep not supported'` | 324 | PHASE3: serial step, execute_substep not supported | PHASE3: sériový krok, execute_substep není podporováno | E | - |

---

## 5. ZCL_FI_ALLOC_STEP_CORR_BCHE

**File**: `src/zfi_alloc_process/zcl_fi_alloc_step_corr_bche.clas.abap`

| Line | Current Code | Msg# | English Text | Czech Text | Severity | Variables |
|------|--------------|------|--------------|------------|----------|-----------|
| 90 | `mo_log->log_text( iv_msgty = 'E' iv_text = \|Failed to update header { ls_header-key_id } with no_items='Y'\| )` | 400 | Failed to update header &1 with no_items='Y' | Selhala aktualizace hlavičky &1 s no_items='Y' | E | &1=key_id |
| 95 | `mo_log->log_text( iv_msgty = 'E' iv_text = \|Failed to update header { ls_header-key_id } with no_items='N'\| )` | 401 | Failed to update header &1 with no_items='N' | Selhala aktualizace hlavičky &1 s no_items='N' | E | &1=key_id |
| 104 | `mo_log->log_text( iv_msgty = 'S' iv_text = \|CORR_BCHE completed: { lv_headers_count } headers processed, { lv_no_items_count } without items\| )` | 402 | CORR_BCHE completed: &1 headers processed, &2 without items | CORR_BCHE dokončena: &1 hlaviček zpracováno, &2 bez položek | S | &1=headers_count, &2=no_items_count |
| 108 | `rs_result-message = \|CORR_BCHE completed. { lv_headers_count } headers processed.\|` | 403 | CORR_BCHE completed. &1 headers processed | CORR_BCHE dokončena. &1 hlaviček zpracováno | S | &1=headers_count |
| 119 | `rs_result-message = 'CORR_BCHE: ALLOCATION_ID is missing'` | 404 | CORR_BCHE: ALLOCATION_ID is missing | CORR_BCHE: ALLOCATION_ID chybí | E | - |
| 126 | `rs_result-message = 'CORR_BCHE: COMPANY_CODE is missing'` | 405 | CORR_BCHE: COMPANY_CODE is missing | CORR_BCHE: COMPANY_CODE chybí | E | - |
| 133 | `rs_result-message = 'CORR_BCHE: FISCAL_YEAR is missing'` | 406 | CORR_BCHE: FISCAL_YEAR is missing | CORR_BCHE: FISCAL_YEAR chybí | E | - |
| 140 | `rs_result-message = 'CORR_BCHE: FISCAL_PERIOD is missing'` | 407 | CORR_BCHE: FISCAL_PERIOD is missing | CORR_BCHE: FISCAL_PERIOD chybí | E | - |
| 144 | `rs_result-message = 'CORR_BCHE validated successfully'` | 408 | CORR_BCHE validated successfully | CORR_BCHE validována úspěšně | S | - |
| 161 | `value = 'Rollback failed: instance variables not initialized'` | 409 | Rollback failed: instance variables not initialized | Rollback selhal: instanční proměnné nejsou inicializovány | E | - |
| 165-167 | `lv_message = \|CORR_BCHE rollback: Changes to ZFI_ALLOC_BCHE cannot be automatically undone. Manual correction may be required for company={ mv_company_code }, year={ mv_fiscal_year }, period={ mv_fiscal_period }, allocation={ mv_allocation_id }.\|` | 410 | CORR_BCHE rollback: Changes cannot be undone. Manual correction may be required for &1/&2/&3/&4 | CORR_BCHE rollback: Změny nelze vrátit. Může být nutná manuální oprava pro &1/&2/&3/&4 | W | &1=company_code, &2=fiscal_year, &3=fiscal_period, &4=allocation_id |
| 192 | `value = 'CORR_BCHE: serial step, execute_substep not supported'` | 411 | CORR_BCHE: serial step, execute_substep not supported | CORR_BCHE: sériový krok, execute_substep není podporováno | E | - |

---

## Summary Statistics

| Step Class | Total Messages | Info | Success | Warning | Error |
|------------|----------------|------|---------|---------|-------|
| **INIT** (001-099) | 8 | 1 | 2 | 0 | 5 |
| **PHASE1** (100-199) | 25 | 13 | 3 | 0 | 9 |
| **PHASE2** (200-299) | 27 | 11 | 4 | 0 | 12 |
| **PHASE3** (300-399) | 25 | 14 | 2 | 0 | 9 |
| **CORR_BCHE** (400-499) | 12 | 0 | 3 | 1 | 8 |
| **TOTAL** | **97** | **39** | **14** | **1** | **43** |

---

## Message Reuse Opportunities

Several messages from PHASE1 are duplicated in PHASE2 and PHASE3. We could consider reusing these:

1. **Company code/fiscal year/period logging** (104-107 in PHASE1, duplicated in PHASE2/PHASE3)
   - Could reuse PHASE1 messages 104-107
   
2. **Data overwrite mode message** (110 in PHASE1, duplicated in PHASE2/PHASE3)
   - Could reuse PHASE1 message 110

3. **Separator line** (111 in PHASE1, duplicated in PHASE2/PHASE3)
   - Could reuse PHASE1 message 111

4. **Validation error messages** (similar patterns across all steps)
   - Each step has its own validation messages for clarity

5. **Rollback error messages** (identical text in multiple steps)
   - Could create a single shared message for "instance variables not initialized"

---

## Notes

1. **Existing Messages (001-012)**: Already created, used primarily in ZCL_FI_ALLOCATIONS helper class
   - e005: UUID generation error
   - e006: Database insert/update error
   - e008: Processing failed with errors
   - e010: Base header not found
   - e011: AMDP/SQL error
   - s009: Processing completed successfully
   - w012: Processing completed with warnings

2. **Message Class Standard**: All messages follow SAP standard:
   - English text is max 73 characters (displayed in UI)
   - Czech translation in SE63
   - Variables use &1, &2, &3, &4 placeholders

3. **Severity Codes**:
   - **I** (Information): Progress messages, status updates
   - **S** (Success): Completion messages
   - **W** (Warning): Non-critical issues
   - **E** (Error): Validation failures, exceptions

4. **Implementation Notes**:
   - Messages 188, 195-196, 205-206 in PHASE1 use existing messages e005/e006
   - Messages 378, 417 in PHASE2 use existing messages e008/e009
   - Messages 189-190, 198-199, 240, 260, 268, 281-282 in PHASE3 use existing messages e005/e006/e008/e009/e010/e011/w012/s009
   - All `MESSAGE sXXX(zfi_alloc) WITH ... INTO lv_dummy` patterns need conversion
   - All `RAISE EXCEPTION ... value = '...'` patterns need conversion
   - All `rs_result-message = |...|` patterns need conversion

---

## Recommendations

1. **Create all messages 013-411** in T100 message class ZFI_ALLOC
2. **Replace hard-coded strings** with MESSAGE statements throughout step classes
3. **Use helper method** for rs_result-message assignment:
   ```abap
   MESSAGE sXXX(zfi_alloc) WITH param1 param2 INTO rs_result-message.
   ```
4. **For exceptions**, use MESSAGE statement before RAISE:
   ```abap
   MESSAGE eXXX(zfi_alloc) WITH param1 param2 INTO DATA(lv_msg).
   RAISE EXCEPTION TYPE zcx_fi_process_error
     EXPORTING textid = zcx_fi_process_error=>invalid_parameters
               value  = lv_msg.
   ```
5. **For BAL logging**, use MESSAGE statement with log_sy_msg():
   ```abap
   MESSAGE iXXX(zfi_alloc) WITH param1 INTO lv_dummy.
   mo_log->log_sy_msg( ).
   ```

---

**Document Version**: 1.0  
**Author**: OpenCode AI Agent  
**Created**: 2026-03-12  
**Status**: Ready for T100 Message Creation
