# System Values Transfer / Setup – Business Rules

**Module prefix:** RX
**System:** ASOKON
**Focus:** What blocks or prevents data-transfer jobs from running, and what conditions govern each data-correction routine

---

## Prerequisites / Master Data Requirements

- RX000R (transfer job resolver): The webshop/file-group configuration record must exist in `RSCDPF` (logical view `RSCDL1`, key `STSKEY`). If not found, no IFS path is returned and the calling job has no transfer target.
- RX000R (FTP path): An IFS path entry for the firm must exist in `AFTPPF` (logical view `AFTPL1`, key firm + name). If not found, or if the path field `aflocd` is a single space, the program calls `AA005R` and writes the error message `'Lokal bane mangler i FTP-Prameter'` — effectively blocking the FTP transfer.
- RX002R (repeat scheduler): The configuration record in `RSCDPF` must carry a non-zero repeat interval in field `rcrepp`. If the current time plus the interval would exceed the end time `rcslut`, the output parameter `p_min` is cleared to zero, which signals the calling job scheduler not to re-queue the job.
- RX739R (contact-person migration): The next contact-person number is obtained by calling `RS003R`. If `RS003R` cannot allocate a number the migration row is not inserted.

---

## Validation Rules

| Program | Condition | Effect |
|---------|-----------|--------|
| RX000R | `RSCDL1` key not found | No path returned; transfer job has no target |
| RX000R | `AFTPL1` not found OR `aflocd = ' '` | Error message logged; FTP transfer blocked |
| RX001R | First character of customer-number field is not a digit 0–9 | Record skipped entirely |
| RX001R | Customer-number field length > 6 characters (version 8.01) | Record skipped entirely |
| RX001R | Balance (`saldo`) contains the string `'E-'` (scientific notation) | Record skipped; value cannot be processed |
| RX002R | `current_time + rcrepp > rcslut` | `p_min` cleared; job not re-scheduled |
| RX732R | `rbraar < 2004` | VAT-basis correction not applied |
| RX732R | Signs of `rbneto`, `rbmvab`, `rbmvag` do not match mismatch pattern | No change |

---

## Configuration and Authorization Rules

- All RX programs read the firm number from the Local Data Area (LDA positions 944–946). All table reads and updates are scoped to that firm.
- RX001R reads from `VEINPF` (external balance import file). Records are accepted only if the customer-number field starts with a numeric digit; non-numeric prefixes indicate header or trailer records and are silently skipped.
- RX001R enforces a credit-block rule: if the source flag `w_sprk = '1'` and the customer's credit-result field `rkkres <> 2` in `RKUNPF`, the routine sets `rkkres = 2` (credit block). This is a one-way gate — the customer is blocked, never unblocked, by this routine.
- RX737R sets `rbautk = '3'` in `RHTRPF` (GL transaction file) for entries where the corresponding record in `RZUMPF` has `rztell = 1` (sum post). This is applied unconditionally to all matching records for the firm; no interactive confirmation is required.

---

## Financial / Transactional Rules

- **RX732R – VAT basis correction:** Targets `RHTRPF` records where `rbraar >= 2004`. Corrects sign mismatch on the VAT gross amount: when `rbneto < 0` AND `rbmvab < 0` AND `rbmvag > 0`, the program negates `rbmvag`. This is a one-time data-quality fix.
- **RX733R – Factoring date reset:** Sets `rnfada = *loval` (lowest date value) on all records in `RKTRPF` (customer transaction file) for the firm. No conditions other than firm membership; all factoring dates are erased.
- **RX735R – Supplier payment date reset:** Sets `rofoda`, `roftim`, `rordat`, `rortim` to their default/low values on all records in `RLTRPF` (supplier transaction file) for the firm. Unconditional.
- **RX738R – Note timestamp reset:** Sets `rxtist = *loval` on all records in `RKLAPF` (customer note file) for the firm. Unconditional.
- **RX740R – Customer side register copy:** Copies records from `RKUNPF` to `RKU2PF`. The combination of parameter `p_fast = '1'` AND (`rkfaty = 24` OR `rkfaty = 28`) results in `rk2fas = 1` being set on the target record, flagging it as a fixed-concept customer in the side register.

---

## Status and Lifecycle Rules

- **RX734R – Customer name cleanup:** Removes all commas from `RKUNPF.rknavn` (customer name) and `RKUNPF.rkalfa` (sort name). Phone number fields are reformatted to the pattern `'nn nn nn nn'` (space-separated pairs). These changes are permanent and immediately written back to `RKUNPF`.
- **RX736R – Supplier name cleanup:** Identical comma-removal logic as RX734R but applied to `RLEVPF.rlnavn` and `RLEVPF.rlalfn`.
- **RX739R – Contact person migration:** Reads `RKUNPF.rkpers` and `RLEVPF.rlpers` contact-person free-text fields and creates structured records in `RUKPPF` (contact person register). A new contact-person number is allocated via `RS003R` for each migrated entry. Already-migrated contacts are not duplicated (the migration is typically a one-time upgrade step).
- **RX001R – Balance import:** The balance value imported from `VEINPF` is written to `RKUNPF.rksald`. Scientific notation values (containing `'E-'`) are skipped because they cannot be safely converted to a numeric balance field.

---

## Special Conditions

- RX000R is a resolver/router, not an interactive program. It is called by batch transfer jobs to obtain the IFS path for file output. A missing path configuration is a silent block — the calling job receives a blank path and must handle the absence.
- RX002R controls job-repeat scheduling entirely through the output parameter `p_min`. Setting `p_min = 0` is the only signal the scheduler reads; the program itself does not submit or cancel any jobs.
- RX732R is designed for post-upgrade data correction and carries a hard year-guard (`rbraar >= 2004`) to limit scope. Applying it more than once is safe because the sign-correction logic is idempotent: once `rbmvag` has the correct sign the condition `rbmvag > 0` is no longer true.
- RX737R marks GL sum-post entries (`rztell = 1` in `RZUMPF`) with `rbautk = '3'`. This flag is used downstream by posting and reconciliation programs; setting it incorrectly on non-sum entries would corrupt GL reconciliation.

---

## Subprogram Calls Affecting Logic

| Caller | Called Program | Purpose | Blocking Effect |
|--------|---------------|---------|-----------------|
| RX000R | AA005R | Logs error for missing IFS path | Called when `aflocd = ' '` or `AFTPL1` not found; blocks transfer |
| RX739R | RS003R | Allocates next contact-person number | Failure to allocate prevents insertion of migrated contact record |

---

## Reference Tables

| Table | Logical View | Key Fields | Usage |
|-------|-------------|-----------|-------|
| RSCDPF | RSCDL1 | STSKEY | Webshop / transfer job configuration; path lookup in RX000R |
| AFTPPF | AFTPL1 | firm + name | IFS FTP path; missing or blank blocks transfer in RX000R |
| RKUNPF | — | firm + customer | Customer master; balance update (RX001R), name cleanup (RX734R), migration (RX739R), side-register copy (RX740R) |
| RLEVPF | — | firm + supplier | Supplier master; name cleanup (RX736R), contact migration (RX739R) |
| VEINPF | — | — | External balance import source read by RX001R |
| RKTRPF | — | firm + customer | Customer transactions; factoring date reset in RX733R |
| RLTRPF | — | firm + supplier | Supplier transactions; payment date reset in RX735R |
| RHTRPF | — | firm + GL entry | GL transaction file; VAT correction (RX732R), sum-post flag (RX737R) |
| RZUMPF | — | firm + GL entry | GL sum-post register; `rztell = 1` triggers rbautk update in RX737R |
| RKLAPF | — | firm + customer | Customer note file; timestamp reset in RX738R |
| RUKPPF | — | firm + contact | Contact person register; target of migration in RX739R |
| RKU2PF | — | firm + customer | Customer side register; target of copy in RX740R |
