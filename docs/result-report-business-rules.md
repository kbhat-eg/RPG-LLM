# Result Report Business Rules

**Module**: Result Report (RR prefix)
**System**: ASOFAK / ASRAPP
**Source files analyzed**: RR108R, RR109R, RR500R, RR510R, RR611R, RR612R, RR621R, RR622R, RR630R, RR634R, RR635R

---

## 1. Prerequisites / Master Data Requirements

The RR module calculates and presents result reports by reading accounting balances from RHSAPF and writing calculated results to RRF9PF. The following must exist before a report can be calculated or printed:

| Requirement | Rule Name | Logic | File | Field | Condition |
|---|---|---|---|---|---|
| Report header must exist in RRF0PF | Report Header Required | RR500R, RR621R, RR630R all chain RRF0PF; if not found: error displayed or loop back | RRF0PF | Report key fields | Not found → parameter screen loops back |
| Report lines must exist in RRF5PF | Line Definitions Required | RR109R reads RRF5PF to know which accounts/ranges contribute to each line; no lines = empty result | RRF5PF | Line key fields | No lines → RRF9PF written as zero |
| Report line ranges in RRF8PF | Range Definitions Required | RR108R maintains from/to account ranges per line; RR109R reads these to sum RHSAPF balances | RRF8PF | Line+range key | No ranges → line balance = zero |
| Accounting balances in RHSAPF | Balance Data Required | RR109R reads RHSAPF for each account range in each period; if no balances: result = zero | RHSAPF | Account+period key | No data → zero result lines |
| Report must NOT exist for auto-build | No-Overwrite Check (RR634R) | RR634R blocks if the report IS found in RRF0PF: *in31=*on — prevents overwriting built reports | RRF0PF | Report key | Found → RR634R blocked |

---

## 2. Validation Rules

### RR621R — Result Report Print Parameter Screen

| Rule Name | Logic | File | Field | Condition |
|---|---|---|---|---|
| Report Must Exist | Chain RRF0PF with entered report number; if not found: loop back silently | RRF0PF | Report key | Not found → screen re-displays |
| Department From Must Be <= To | a2pavf > a2pavt: *in31 = *on | Input | a2pavf/a2pavt | From > To → save blocked |
| Run Date Must Be Valid | Invalid date format detected: *in33 = *on | Input | Run date field | Invalid date → save blocked |
| Period From Must Be <= To | a2pfrf > a2pfrt: *in35 = *on | Input | a2pfrf/a2pfrt | From > To → save blocked |
| Department Lock When User Has avd Restriction | If AUSRPF.abbavd='1': *in30=*on (protect fields), a2pavf and a2pavt are forced to equal absavd (user's department) | AUSRPF | abbavd/absavd | Department-restricted user cannot select other departments |

### RR630R — Report Print Parameter Screen

| Rule Name | Logic | File | Field | Condition |
|---|---|---|---|---|
| Report Number Must Exist | Chain RRF0PF; if not found: *in31 = *on | RRF0PF | Report key | Not found → error indicator |

### RR634R — Auto-Build Report Parameter

| Rule Name | Logic | File | Field | Condition |
|---|---|---|---|---|
| Report Must NOT Already Exist | If report IS found in RRF0PF: *in31 = *on — auto-build refused | RRF0PF | Report key | Found → blocked (prevents overwrite) |

### RR108R — From/To Range Maintenance (Preview only — 58KB)

| Rule Name | Logic | File | Field | Condition |
|---|---|---|---|---|
| Subfile maintenance validation | Indicators *in31–*in79 used for validation errors on from/to account ranges; from > to is expected to be validated | RRF8PF | Range from/to | From > To blocked |

---

## 3. Configuration and Authorization Rules

| Rule Name | Logic | File | Field | Condition |
|---|---|---|---|---|
| Department Restriction | RR621R checks AUSRPF.abbavd='1'; if true, the user is locked to their own department (absavd) and cannot enter other department ranges | AUSRPF | abbavd/absavd | User-level department restriction enforced |
| Print Routing via AP600R/AP601C | RR630R calls AP600R/AP601C for printer routing before calling RR631R for actual print | AP600R/AP601C | Printer parameters | Routing must succeed for print to proceed |
| Period Configuration from ASOKON | RR109R uses ASOKON (accounting period configuration) to know the period structure; periods mapped to RHSAPF balance fields | ASOKON | Period structure | Period setup determines which balance fields are read |
| Extended Balance Fields (v6.30) | v6.30: RRF9PF extended to 14 balance fields (saldo fields); RR109R writes all 14 periods; earlier versions had fewer | RRF9PF | saldo1–saldo14 | 14-period model from v6.30 |

---

## 4. Financial / Transactional Rules

| Rule Name | Logic | File | Field | Condition |
|---|---|---|---|---|
| Balance Source is RHSAPF | RR109R reads RHSAPF (accounting balance file) for each account range defined in RRF8PF; balances summed per period | RHSAPF | Account+period | Only accounts within defined ranges are included |
| Results Written to RRF9PF | After summing, RR109R writes or updates RRF9PF with the calculated result per line per period | RRF9PF | Line+period key | Overwritten on each recalculation |
| RR500R Option 8 Triggers Recalculation | Selecting option 8 on a report in RR500R calls RR109R to recalculate, then RR510R to display; options 1/2/3 call print programs directly | RR500R | User option | Option 8 = calculate + display |
| Print Programs Use Pre-Calculated RRF9PF | RR621C, RR623C, RR627C read RRF9PF for display; they do not recalculate; stale data is possible if RR109R has not been run | RRF9PF | All result fields | Caller must ensure RR109R run before print |
| RR627C — Expected Result Print | Option 3 in RR500R calls RR627C for expected/budget result comparison; reads same RRF9PF | RRF9PF | Budget vs actual | Depends on RRF9PF being current |

---

## 5. Status and Lifecycle Rules

| Rule Name | Logic | File | Field | Condition |
|---|---|---|---|---|
| Sequential Text Reading (RR611R) | RR611R reads RRF5PF sequentially for a report; p_retk='S' starts, blank advances, 'F' finishes; returns p_retk='N' at EOF or firm/report boundary change | RRF5PF | Firm+report key | EOF or boundary = 'N' return code |
| Report Recalculation On Demand | RR109R is called explicitly (via RR500R option 8 or directly); no automatic trigger | RRF9PF | — | Manual recalculation required |
| Auto-Build Creates New Reports | RR634R is called to create a new report structure; blocks if report already exists to avoid accidental overwrite | RRF0PF | Report key | One-time creation guard |
| Print Parameters Written to LDA | RR621R writes parameters to LDA positions 300+ (rhprap, a2p-fields); these are read by the print programs | LDA | Positions 300+ | LDA is the inter-program communication path for print params |

---

## 6. Special Conditions (Program-Specific)

### RR500R — Result Report Inquiry/Selection

- Displays list of reports from RRF0PF.
- Option 1 → RR621C (standard period print).
- Option 2 → RR623C (13-period comparison print).
- Option 3 → RR627C (expected/budget result print).
- Option 8 → Calls RR109R (calculate) then RR510R (display balances on screen).
- Positioning by report number or description.

### RR109R — Calculation Engine (52.9KB — preview only)

- Reads all RRF5PF lines for the report, then for each line reads all RRF8PF ranges to sum RHSAPF account balances.
- v6.30: Extended to handle 14 balance periods.
- Output written to RRF9PF; existing records updated, new records written.

### RR108R — From/To Range Maintenance (58KB — preview only)

- Subfile maintenance screen for RRF8PF.
- Users define which account ranges (from/to) contribute to each result report line.
- Indicators *in31–*in79 used for field-level validation; from > to range is blocked.

### RR621R — Print Parameters with Department Restriction

- The most security-sensitive program in the module.
- AUSRPF.abbavd='1' locks the user to their department; *in30 protects department fields from editing.
- Parameters written to LDA are consumed by RR621C without further validation.

### RR634R — Auto-Build Guard

- Unusual reversal: the program blocks when the report IS found, not when it is missing.
- This is an explicit "create-only" guard; calling RR635R only when report does not exist.

---

## 7. Subprogram Calls Affecting Logic

| Caller | Callee | Purpose | Effect on Blocking |
|---|---|---|---|
| RR500R | RR109R | Calculate result report | Must complete before display is meaningful |
| RR500R | RR510R | Display calculated balances | Depends on RRF9PF being populated |
| RR500R | RR621C | Print standard period report | Reads LDA + RRF9PF |
| RR500R | RR623C | Print 13-period report | Reads LDA + RRF9PF |
| RR500R | RR627C | Print expected result report | Reads LDA + RRF9PF |
| RR621R | (LDA write) | Write print parameters to LDA 300+ | LDA contents passed to print callee |
| RR630R | AP600R/AP601C | Printer routing | Must succeed before print |
| RR630R | RR631R | Actual print execution | Called after routing |
| RR634R | RR635R | Build report structure | Called only when report does not exist |
| RR611R | (reads RRF5PF) | Sequential line reader | Returns 'N' at EOF; caller uses to loop |

---

## 8. Reference Tables

| Table | Description | Key Fields |
|---|---|---|
| RRF0PF | Result report header definitions | Firm, report number |
| RRF5PF | Result report line definitions | Firm, report, line number |
| RRF8PF | Account range definitions per line | Firm, report, line, range sequence |
| RRF9PF | Calculated result balances | Firm, report, line, period |
| RHSAPF | Accounting balances (source data) | Firm, account, period |
| AUSRPF | User authorization / department restrictions | Firm, user |
| ASOKON | Accounting period configuration | Firm, period structure |
