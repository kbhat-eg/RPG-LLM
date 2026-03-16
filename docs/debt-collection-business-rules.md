# Business Rules: Debt Collection (RC Module)

**System:** ASOFAK
**Module Prefix:** RC
**Programs Analyzed:** RC100R, RC110R, RC400R, RC401R, RC600R, RC601R, RC650R, RC651R, RC653R, RC700R (RC701R), RC720R
**NXKORR Overrides:** None found
**Date:** 2026-03-16

---

## 1. Prerequisites / Master Data Requirements

- The debt collection configuration record must exist in `RCA0PF` for the active firm. RC651R, RC653R, and RC720R all CHAIN to `RCA0PF` by firm. If not found, the agency name field `RC0CNK` is blank and RC720R returns 'Ukjent' — no debt collection processing is possible.
- The debt collection agency name (`RCA0PF.RC0CNK`) must be one of three recognised values: 'Lindorf', 'Svea', or 'Collectio'. Any other value causes RC100R to call `AA005R` and display a blocking error message ('Ukjent Inkassobyrå'). The save is refused.
- Customer records must exist in `RKUNPF` for individual-customer validation in RC650R. If a customer number is entered individually and not found in `RKUNPF`, `*in47` is set and the run is blocked.
- The customer receivables transaction file `RKTPF` / `RKTRPF` must contain open transactions for debt collection to operate. RC651R iterates `RKTRPF`; no transactions means no notices are generated — not an error, but no output.
- For SVEA payment export (RC700R/RC701R), the customer category in `RKUNPF.RKKATG` must be 701 or 702. Customers outside these categories are skipped entirely.

---

## 2. Validation Rules

### RC100R — Debt Collection Configuration Maintenance
- Agency name (`c2cnk`): if not 'Lindorf', 'Svea', or 'Collectio' → calls `AA005R` with message 'Ukjent Inkassobyrå' and blocks save.
- Description (`c2txt`): if blank → `*in31` blocks save.
- Copy to target firm: if a configuration record already exists for the target firm in `RCA0PF` → `*in32` blocks the copy operation.

### RC110R — Debt Collection Case Register Maintenance
- Cases are filtered: only cases where `RCNFPF.RCNCNK = RCA0PF.RC0CNK` (active agency match) are shown. Cases for other agencies are skipped (`goto neste`).
- Cases with status 'A' (closed) are hidden when `w_vise = 'A'` (show-open-only mode). Skipped via `goto neste`.
- File filter: cases where `RCNFPF.w_file <> l_file` are skipped. This limits the display to cases belonging to the current user's file group.
- Only option codes 2 (change) and 5 (view) are permitted. No creation of new cases through this program.

### RC400R — Case Deletion Parameters
- End date (`a2edat`): must be a valid date in DMY format (`*dmy test(d)` → if `*in31` is set, the date is invalid and the run is blocked).
- Type (`a2etyp`): must be 'J' (yes, closed cases) or 'N' (no, open cases). Any other value → `*in32` blocks.

### RC401R — Case Deletion Batch
- Status check: if `RCNFPF.RCNSTA = ' '` (case not closed), the record is skipped — open cases are never deleted.
- Date filter: if `RCNFPF.RCNDAT > w_edat` (closure date later than cutoff), the record is skipped.
- Type check: deletion only occurs when `w_type = 'N'`. Type 'J' processes (reads) but does not delete.

### RC600R — Case Print Parameters
- Same validation as RC400R: end date must be valid DMY format (`*in31`), type must be 'J' or 'N' (`*in32`).

### RC601R — Case Print Batch
- File filter: if `w_file <> l_file` → skip record.
- Type 'J' (closed): if `RCNFPF.RCNSTA = ' '` (open case) → skip.
- Type 'N' (open): if `RCNFPF.RCNSTA = 'A'` (closed case) → skip.
- Date filter: if `RCNFPF.RCNDAT > w_edat` → skip.

### RC650R — Debt Notice Parameter Screen
- Run date (`a2pdad`): must be valid DMY date (`test(d)` → `*in31` if invalid). Blocks run.
- Notice date (`a2pdat`): must be valid DMY date (`test(d)` → `*in32` if invalid). Blocks run.
- Running days (`a2plop`): if `> 50` → `*in34` blocks. Maximum 50 running days allowed.
- Notice type (`a2pkod`): must be 'I' (inkasso/collection) or 'P' (purring/reminder). Any other value → `*in36` blocks.
- Send flag (`a2pskr`): must be 'J' (yes) or ' ' (blank/no). Any other value → `*in37` blocks.
- Department range: `a2pavf > a2pavt` → `*in43` blocks (from-department greater than to-department).
- Customer category range: `a2pkaf > a2pkat` → `*in42` blocks.
- Salesperson range: `a2psef > a2pset` → `*in44` blocks.
- Customer number range: `a2pknf > a2pknt` → `*in48` blocks.
- Individual customer numbers: each entered customer number is validated against `RKUNPF`. If not found → `*in47` blocks.

### RC651R — Debt Notice Selection Batch (Core Blocking Logic)
The following conditions cause a customer to be skipped entirely (`goto nykund`):
- `RKUNPF.RKRENK = 2` → customer excluded from debt collection processing.
- Notice type is 'I' (inkasso) and `RKRENK = 2 or 3` → skip.
- Customer found in `RKBPPF` (exclusion list) with `RKINKA = 1` → skip.
- CO402R switch `RC651C_Ikke_oppd_Kunde_m_INKK = N` → if `u_Inkk = *on` and `RKUNPF.RKINKK = 'N'` → skip.
- `RKUNPF.RKSALD <= 0` → customer balance is not positive, skip.
- `RKUNPF.RKKATG < pkatf or > pkatt` → customer category outside allowed range, skip.
- `RKUNPF.RKSLGR < pself or > pselt` → salesperson outside allowed range, skip.

Per-transaction conditions causing a transaction to be skipped (`goto neste`):
- `RKTRPF.RNINKO = 'I' or 'J'` → notice already sent (inkasso or purring flag set). Transaction not eligible.
- `RKTRPF.RNBILK = 89` → complaint voucher code. Excluded from debt collection.
- `RKTRPF.RNREKL = 'J'` → complaint flag set. Excluded.
- `RKTRPF.RNREST = 0` → zero remaining balance. Skip.
- `RKTRPF.RNANTP < ppunr` → number of previous reminders is less than the required minimum (`ppunr`). Transaction has not received enough reminders. Skip.
- Voucher code matches any of `RCA0PF.RCUBK1`–`RCUBK5` (configured excluded voucher codes). Skip.
- `RKTRPF.RNBILK = 65` → hard-coded excluded voucher code. Skip.
- Credit line with due date not yet reached: if `RNREST >= 0 and w_forf >= w_fdat` (due date not past) → skip.

### RC653R — Debt Notice Print
- Customer balance check: `RKUNPF.RKSALD <= 0 or RKUNPF.RKPNEG = 'N'` → skip this customer entirely.
- Transaction filter: `RKTRPF.RNBILK = 89 or RNBELØ = 0` → skip this transaction line.
- Debt collection case number is incremented from `RCA0PF` on each new notice. `RCA0PF` must be updatable.

### RC700R/RC701R — SVEA Payment Export
- Only processes payment voucher codes 10, 14, and 87. All other voucher codes are skipped.
- Only customers with `RKUNPF.RKKATG = 701 or 702` are included. All others skipped.
- If the combination of customer + voucher is already found in `RKTSPF` (SVEA export register) → skip (already exported).

### RC720R — Agency Name Lookup
- If `RCA0PF` not found for firm, or if `RC0INF` is blank → returns 'Ukjent' to caller.

---

## 3. Configuration and Authorization Rules

- `RCA0PF` is the master configuration record for the debt collection module. It must exist and be correctly configured before any debt collection run.
- Excluded voucher codes: `RCA0PF.RCUBK1`–`RCUBK5` define up to 5 voucher codes to exclude from all debt collection notices. These are checked in RC651R for every transaction.
- Credit note codes: `RCA0PF.RCBKK1`–`RCBKK5` define voucher codes treated as credit notes. These inform balance calculations.
- CO402R switch `RC651C_Ikke_oppd_Kunde_m_INKK`: if active ('N' configured), customers with `RKUNPF.RKINKK = 'N'` are excluded from debt collection runs. This is a per-installation override.
- CO402R switch 'INKASSOFORSLAGTILEXCEL' (RC653R): if active, debt collection proposal is additionally exported to Excel format via `ASPTOXLSX`. Does not affect the standard print path.
- LDA position 944–946 (`l_firm`) scopes all processing. LDA file group position (for RC110R `l_file`) limits case display to the current file context.

---

## 4. Financial / Transactional Rules

- Debt notice eligibility requires `RKUNPF.RKSALD > 0` (positive customer balance). A zero or negative balance completely prevents notice generation.
- Transaction-level eligibility requires `RKTRPF.RNREST <> 0` (non-zero remaining balance) and `RNBILK <> 89` (not a complaint) and `RNREKL <> 'J'` (no complaint flag).
- The minimum reminder count (`ppunr`, parameter field) must be satisfied: each eligible transaction must have `RNANTP >= ppunr` reminders sent before being included in a debt collection run.
- Credit lines (`RNREST >= 0`) bypass the due-date check only when due date has passed (`w_forf < w_fdat`). Credit lines with future due dates are excluded.
- SVEA export: payment amount sign is calculated based on debit/credit account polarity. Only voucher codes 10, 14, 87 carry payment data for SVEA.

---

## 5. Status and Lifecycle Rules

- Debt collection case status in `RCNFPF`: blank = open, 'A' = closed. RC401R and RC601R use this field to differentiate between open and closed case processing.
- Once a notice is sent, `RKTRPF.RNINKO` is set to 'I' (inkasso) or 'J' (purring). This field prevents the same transaction from being included in subsequent notice runs.
- Case deletion (RC401R): only closed cases (`RCNSTA = 'A'`) with closure date on or before the cutoff (`RCNDAT <= w_edat`) are deleted. Active cases are never deleted.

---

## 6. Special Conditions

- Notice type 'I' (inkasso) applies stricter customer exclusion: customers with `RKRENK = 2 or 3` are skipped (whereas for type 'P' only `RKRENK = 2` is excluded).
- Hard-coded voucher code 65 is always excluded from debt collection regardless of `RCA0PF` configuration.
- RC651R checks `RKBPPF` (customer exclusion list): if the customer is found in this table with `RKINKA = 1`, they are excluded even if their balance and category would otherwise qualify.
- Running days (`a2plop`) maximum of 50 is a hard validation limit — it cannot be overridden by configuration.
- Parameters entered in RC650R are saved to `RKWAPF` (version 9.03) for reuse in subsequent runs within the same session.

---

## 7. Subprogram Calls Affecting Logic

| Called Program | Called From | Purpose / Effect on Logic |
|---|---|---|
| `AA005R` | RC100R | Displays error message ('Ukjent Inkassobyrå'); blocks save on unrecognised agency name |
| `AA007R` | RC650R (validation) | Displays field-level error messages for parameter validation failures |
| `CO402R` | RC651R, RC653R | Reads switches controlling customer exclusion and Excel export |
| `ASPTOXLSX` | RC653R | Excel export of debt collection proposal (when CO402R switch active) |
| `RC400C` | RC400R | CL batch driver for case deletion |
| `RC600C` | RC600R | CL batch driver for case print |

---

## 8. Reference Tables

| Table | Role in Module |
|---|---|
| `RCA0PF` | Debt collection configuration — agency name, excluded voucher codes, case number counter |
| `RCNFPF` | Debt collection case register — individual cases with status and closure date |
| `RKUNPF` | Customer register — balance (`RKSALD`), category (`RKKATG`), salesperson (`RKSLGR`), exclusion flags (`RKRENK`, `RKINKK`, `RKPNEG`) |
| `RKTRPF` / `RKTPF` | Customer transaction register — individual open receivables; notice-sent flags (`RNINKO`), voucher code (`RNBILK`), complaint flag (`RNREKL`) |
| `RKBPPF` | Customer debt-collection exclusion list — `RKINKA = 1` excludes customer |
| `RKWAPF` | Debt notice parameter save file — stores RC650R parameters for reuse |
| `RKTSPF` | SVEA export register — tracks already-exported payment lines |
| `RCSVPF` | SVEA payment output file — receives payment data from RC700R |
