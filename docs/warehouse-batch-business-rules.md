# Business Logic for Warehouse Batch Operations

This module covers batch maintenance, data migration, and EDI integration programs in the ASLAGR system. The programs covered are LX100R (EDI message code update — LEDIPF batch), LX101R (EDI message number link to purchase orders and history), LX400R (warehouse unit transfer to historical stock ledger), LX401R (warehouse unit transfer from item master to warehouse), LX700R (external file / Excel import for warehouse parameters by EAN), LX710R (move warehouse parameters between warehouses — 99→02), and LX720R (copy item-level parameters to warehouse records).

These are batch programs with no interactive user screen. They run without operator input and apply their transformations unconditionally across entire files, subject to the specific filtering conditions documented below.

---

## Prerequisites / Master Data Requirements

1. **LX401R: Item Must Exist in VVARPF (Implicit — Unit Copy)**
   - Logic: LX401R reads all VLAGPF records sequentially and chains VVARPF for each to get the lagerenhet (VVENHL — warehouse unit). The chain is unconditional. If VVARPF is not found for an item (indicator `*in91 = *on`), the warehouse unit field (VLENHL) is still updated — it is set to whatever was already in VVENHL before the chain, which may result in a blank unit. No explicit skip or error is raised for missing VVARPF records.
   - File: VVARPF, VLAGPF
   - Fields: VVENHL → VLENHL
   - Condition: Best practice requires VVARPF to exist for all VLAGPF items prior to running LX401R.

2. **LX400R: Item Must Exist in VVARPF (Implicit — Unit Copy to History)**
   - Logic: LX400R reads all VHISPF (historical stock ledger) records sequentially and chains VVARPF by firm + item number to get VVENHL. The unit is written to VHENHL in the history record. No blocking or skip occurs on VVARPF not-found — the value in the chain destination is used as-is.
   - File: VVARPF, VHISPF
   - Fields: VVENHL → VHENHL

3. **LX700R: EAN Number Must Resolve to Item via VE710R**
   - Logic: LX700R reads LWEXPF (external work file loaded from Excel/CSV). For each row, the EAN number is passed to VE710R (EAN barcode lookup). VE710R returns a status code w_east. Only rows where `w_east = '1'` (resolved successfully) are processed. Rows where EAN resolution fails are silently skipped.
   - Program: VE710R
   - Files: LWEXPF (input work file), VVARPF (item lookup)
   - Condition: `w_east <> '1'` → row skipped, no VLAGPF update.

4. **LX700R: Item Must Exist in VVARPF After EAN Resolution**
   - Logic: After VE710R resolves the EAN to an item number (w_eava), VVARPF is chained by firm + item number. If not found (indicator `*in92 = *on`), the row is skipped.
   - File: VVARPF
   - Condition: `*in92 = *on` → row skipped.

---

## Validation Rules

5. **LX100R: Only Records with Status 7 or 9 Are Processed**
   - Logic: LX100R uses embedded SQL to count and update records in LEDIPF (EDI message file). Only records where LISTAT = '7' (deleted in NOBB) or LISTAT = '9' (approved) are selected. Records with other status codes are not affected.
   - File: LEDIPF
   - Field: LISTAT
   - SQL: `WHERE listat = '7' OR listat = '9'`

6. **LX100R: Aborts If No Matching Records Exist**
   - Logic: LX100R first counts matching records. If `w_anta < 1` (no records with status 7 or 9), the program sets `*inlr = *on` and returns immediately without any updates.
   - Condition: `w_anta < 1` → program exits without action.

7. **LX100R: Aborts If SQL Update Returns No Rows (SQLCOD = 100)**
   - Logic: After each UPDATE statement (for status 7→'S' and status 9→'G'), the SQL return code is checked. If `sqlcod = 100` or `sqlstt = '02000'` (no rows updated), the program exits.
   - Condition: `sqlcod = 100 or sqlstt = '02000'` → program exits after partial update.

8. **LX100R: Old LEDIPF Records Are Deleted**
   - Logic: At the end of processing, LX100R deletes LEDIPF records where LIMDAT (message date) is earlier than 2008-12-31, provided LIKODE (message code) is not blank. This is a hardcoded purge of old processed records.
   - SQL: `DELETE FROM ledipf WHERE limdat < '2008-12-31' AND likode <> ' '`

9. **LX101R: Only EDI Records with LINUMM > 0 and LIMLIN = 100001 Are Linked**
   - Logic: LX101R reads LEDIPF and processes only records where the purchase order number (LINUMM) is greater than zero and the source line indicator (LIMLIN) equals 100001. All other records are skipped.
   - File: LEDIPF
   - Fields: LINUMM, LIMLIN
   - Condition: `LINUMM > 0 and LIMLIN = 100001` → process; otherwise skip.

10. **LX101R: Purchase Order Must Exist in LOHEPF**
    - Logic: After selecting eligible LEDIPF records, LOHEPF (purchase order header) is chained by firm + order number + suffix. Only if found (`%found`) is the EDI message number (LOMNUM) updated.
    - File: LOHEPF
    - Condition: `not %found(LOHEPF)` → no update to LOHEPF.

11. **LX101R: History Record Suffix Must Match**
    - Logic: SIHEPF (purchase order history) is read for the same order number. For each history record, the suffix (SHSUFF) is compared to the EDI record suffix (LISUFF). Only records where `LISUFF = SHSUFF` have their EDI message number updated.
    - File: SIHEPF
    - Condition: `LISUFF <> SHSUFF` → SHMNUM not updated for that record.

---

## Configuration and Authorization Rules

12. **LX710R: Source Warehouse Is Hardcoded to 99**
    - Logic: LX710R reads VLAGPF for warehouse 99 only (vlagl2_lage = 99). It is a one-time migration utility that moves data from warehouse 99 to warehouse 02. The target warehouse (02) is also hardcoded. There are no user-configurable parameters.
    - File: VLAGPF
    - Condition: Source = warehouse 99; target = warehouse 02. Both hardcoded.

13. **LX710R: Source Record Must Have At Least One Non-Zero Parameter**
    - Logic: Source records from warehouse 99 are skipped if all of the following fields are zero: VLMINI, VLMAKS, VLLTID, VLBPUN, VLBMIN, VLBMEN, VLSIKL. Records with all-zero parameters carry no meaningful data and are excluded from migration.
    - Condition: `VLMINI = 0 and VLMAKS = 0 and VLLTID = 0 and VLBPUN = 0 and VLBMIN = 0 and VLBMEN = 0 and VLSIKL = 0` → goto les_neste (skip).

14. **LX710R: Target Warehouse 02 Record Only Updated If It Has All-Zero Parameters**
    - Logic: If a VLAGPF record already exists for the item in warehouse 02, it is updated only if all its parameter fields are also zero (no existing meaningful data). If the target already has non-zero values, it is not overwritten.
    - Condition: `%found(VLAGPF for warehouse 02) and all target parameters ≠ 0` → no update performed.

15. **LX720R: Item Must Have At Least One Non-Zero Parameter in VVARPF**
    - Logic: LX720R reads all VVARPF records. For each item, it checks whether any of VVMINI, VVMAKS, VVLTID, VVBPUN, VVBMEN, VVSIKL, or VVPLAS (shelf location) is non-zero or non-blank. If all are zero/blank, the item is skipped — no warehouse records are updated.
    - File: VVARPF
    - Condition: All parameters zero and VVPLAS blank → `upd_lager` not called, item skipped.

16. **LX720R: Each Warehouse Field Is Only Copied If the Target Field Is Currently Zero**
    - Logic: For each VLAGPF record found for the item, individual parameter fields are updated only if the existing warehouse value is zero (non-destructive copy). Existing non-zero values are preserved. This applies to: VLMINI, VLMAKS, VLLTID, VLBPUN, VLBMEN, VLSIKL, and VLPLAS (blank check for shelf location).
    - File: VLAGPF
    - Conditions:
      - `VLMINI = 0` → set to VVMINI; else leave unchanged.
      - `VLMAKS = 0` → set to VVMAKS.
      - `VLLTID = 0` → set to VVLTID.
      - `VLBPUN = 0` → set to VVBPUN.
      - `VLBMEN = 0` → set to VVBMEN.
      - `VLSIKL = 0` → set to VVSIKL.
      - `VLPLAS = *blank` → set to VVPLAS.

---

## Financial / Transactional Rules

17. **LX700R: Min/Max/Order Parameters Set from CSV Columns**
    - Logic: LX700R parses the external CSV/Excel file using fixed column positions from the data structure: d_minl (columns 77–83) → VLMINI and VLBPUN (order point); d_maxl (columns 70–76) → VLMAKS; d_bmen (columns 87–93) → VLBMEN (order quantity). These values are written to both new and existing VLAGPF records.
    - File: LWEXPF → VLAGPF
    - Fields: VLMINI, VLMAKS, VLBPUN, VLBMEN
    - Note: VLBPUN (bestillingspunkt) is set to the same value as VLMINI (minimum stock level) — they share the same source column d_minl.

18. **LX700R: New VLAGPF Record Created If Item Not in Warehouse**
    - Logic: If the resolved item is not found in VLAGPF for the target warehouse (indicator `*in93 = *on`), a new VLAGPF record is written with the firm, item number, warehouse (w_lage = 00), unit, and the four parameter values from the CSV. Existing records are updated in-place.
    - Condition: `*in93 = *on` → WRITE new VLAGPF record; otherwise UPDATE existing.

---

## Status and Lifecycle Rules

19. **LX100R: Message Code Updated Based on NOBB Status**
    - Logic: The business purpose of LX100R is to translate NOBB status codes to internal processing codes in LEDIPF: status '7' (slettede poster — deleted in NOBB) → LIKODE = 'S'; status '9' (godkjente poster — approved) → LIKODE = 'G'. This enables downstream programs to identify which EDI messages have been processed.
    - File: LEDIPF
    - Fields: LISTAT (input filter), LIKODE (output code)

20. **LX101R: EDI Message Number Written to Purchase Order and History**
    - Logic: LX101R links the EDI message number (LIMNUM from LEDIPF) back to the purchase order header (LOHEPF.LOMNUM) and the corresponding history record (SIHEPF.SHMNUM). This creates a traceable link between the EDI transmission and the internal purchase document for reconciliation purposes.
    - Files: LEDIPF, LOHEPF, SIHEPF
    - Fields: LIMNUM → LOMNUM, SHMNUM

21. **LX401R: All VLAGPF Records Updated — No Filter Applied**
    - Logic: LX401R reads the entire VLAGPF file sequentially (no firm or warehouse filter in the read loop — it reads from an open cursor). Every record has its unit (VLENHL) overwritten with the value from the corresponding VVARPF record (VVENHL). The audit trail fields (VLEDAT, VLETIM, VLEUSR) are updated with the current timestamp and user.
    - Scope: All VLAGPF records across all firms and warehouses.

22. **LX400R: All VHISPF Records Updated — No Filter Applied**
    - Logic: LX400R reads the entire VHISPF (historical stock ledger) sequentially and updates the unit field (VHENHL) from VVARPF.VVENHL. The audit trail (VHEDAT, VHETIM, VHEUSR) is updated.
    - Scope: All VHISPF records.

---

## Special Conditions (Program-Specific)

23. **LX700R: Header Rows Are Skipped**
    - Logic: LX700R skips rows in LWEXPF where the first two characters of the EAN column (d_ean1) equal 'ID' (column header) or the first three characters equal spaces. This guards against processing the CSV header row as data.
    - Condition: `%subst(d_ean1:1:2) = 'ID' or %subst(d_ean1:1:3) = '   '` → row skipped.

24. **LX700R: EAN Column Fallback — Uses Second EAN Column If First Is '?'**
    - Logic: If the first EAN column (d_ean1) begins with '?', the program attempts to use the second EAN column (d_ean2). If d_ean2 also begins with '?', the row is skipped entirely (goto behand_ut).
    - Condition: `d_ean1 starts with '?'` → try d_ean2; `d_ean2 starts with '?'` → skip row.

25. **LX700R: Target Warehouse Hardcoded to 00**
    - Logic: The warehouse number used for all VLAGPF operations in LX700R is hardcoded to `w_lage = 00`. All records are created or updated in warehouse 00 regardless of the source data.
    - Condition: Always warehouse 00.

26. **LX710R: Firm Hardcoded to 000**
    - Logic: The firm number used for VLAGPF operations in LX710R is hardcoded to `w_firm = 000`. This program does not read the firm from the LDA; it operates only on firm 000 data.
    - Condition: Always firm 000.

27. **LX720R: Firm Hardcoded to 000**
    - Logic: Similarly, LX720R reads VVARPF and VLAGPF using `w_firm = 000`. Both the source scan and the target update are scoped to firm 000.
    - Condition: Always firm 000.

28. **LX100R: SQL Date Format Set to ISO**
    - Logic: LX100R sets SQL date format to ISO (`Set Option DatFmt=*ISO`) at the start of processing. This ensures date comparisons in the DELETE statement (LIMDAT < '2008-12-31') work correctly regardless of system default date format.
    - SQL: `Set Option DatFmt=*ISO`

---

## Subprogram Calls Affecting Logic

| Program | Trigger | Logic | Impact |
|---------|---------|-------|--------|
| VE710R | LX700R — each CSV row | Resolve EAN number to item number and unit | Rows skipped if w_east ≠ '1' (resolution failed) |

---

## Reference Tables

| Table | Key | Purpose in Warehouse Batch Processing |
|-------|-----|---------------------------------------|
| VLAGPF | VLFIRM + VLLAGE + VLVARE | Warehouse item master — target for LX401R, LX700R, LX710R, LX720R updates |
| VVARPF | VVFIRM + VVVARE | Item master — source of VVENHL (unit) for LX400R/LX401R; parameter source for LX720R |
| VHISPF | VHFIRM + VHLAGE + VHVARE | Historical stock ledger — target for LX400R unit update |
| LEDIPF | LIFIRM + LISTAT | EDI message file — source and target for LX100R (message code update) and LX101R (message number link) |
| LOHEPF | LOFIRM + LONUMM + LOSUFF | Purchase order header — updated by LX101R with EDI message number (LOMNUM) |
| SIHEPF | SHFIRM + SHNUMM + SHSUFF | Purchase order history — updated by LX101R with EDI message number (SHMNUM) |
| LWEXPF | Sequential | External work file (CSV/Excel import) — input for LX700R; loaded by ASPFRSTMF or equivalent |
