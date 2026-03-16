# System Values / Logistics Setup — Business Rules

**Module:** 35 (LA prefix)
**Focus:** What blocks or prevents logistics system initialisation and configuration operations

---

## 1. Prerequisites / Master Data

Before logistics setup operations can proceed, the following conditions must be satisfied:

- **Company number in LDA** (`l_firm` at pos 944–946) must be set. LA101R, LA400R, LA900R, LA910R, and LA940R all read `l_firm` from the LDA to scope database operations to the correct firm.
- **LSTSPF (warehouse status codes)** must be accessible as an update-eligible file for LA101R to perform UPSERT. If the file is unavailable, status code maintenance cannot proceed.
- **LA400R requires a target company number** (`p_firm`) to be passed as a parameter. Without a valid p_firm, all 22 bulk UPDATE statements operate with a zero or blank key, producing unpredictable results.
- **AUSRPF (user register), ALIBPF (library list), AFORPF (print routines), ANUMPF (number series), VLTYPF (line types), VOTYPF (order types)** must all be writable for LA900R to create the standard initialisation records.
- **LA910R requires a confirmed choice** (`c1valg <> 0`) before any year-end procedure runs. If c1valg = 0, the program exits immediately without calling LA912C.
- **LA940R requires LOHEPF (order header)** to be accessible. For each LODTPF (order line) row, LA940R chains LOHEPF by firm + order number + suffix. If LOHEPF is not accessible, all lines will be silently skipped since the `if not %found` condition will always be true.

---

## 2. Validation Rules

### LA101R — General warehouse status codes maintenance

| Condition | Effect |
|-----------|--------|
| No blocking validation is defined | Screen reads and displays current LSTSPF values; always proceeds to UPSERT on ENTER |
| LSTSPF row found for firm | Field values are **updated** (UPDATE) |
| LSTSPF row not found for firm | New row is **created** (WRITE) |

### LA400R — Company number reassignment (bulk update)

| Condition | Effect |
|-----------|--------|
| `p_firm` parameter not provided (zero or blank) | All 22 UPDATE statements execute with zero/blank firm key — potentially updating wrong rows |
| Any of the 22 target tables is not accessible | SQL UPDATE raises an exception; processing may halt depending on error handling |
| No blocking conditions defined in the program | All updates run unconditionally — there is no rollback if a partial failure occurs |

### LA900R — Standard values initialiser for new warehouse company

| Condition | Effect |
|-----------|--------|
| User enters choice `1` (overwrite existing) | All standard value records are written unconditionally, overwriting any existing setup |
| User enters choice other than `1` | Standard records are only written if they do not already exist (insert-only) |
| Default supplier number (`b1ldor`) is zero | Standard user and number series records are created with a zero default supplier — no blocking but may cause downstream errors |
| AA030C (IP program generator) fails | The standard IP program is not created; no blocking of the LA900R write operations |
| AA034C (OS/400 user profile generator) fails | The standard user profile is not created; warehouse users will have no OS/400 profile |

### LA910R — Year-end procedures parameter screen

| Condition | Effect |
|-----------|--------|
| `c1valg = 0` (confirmation not entered) | Program exits without running any year-end procedure — **no operation** |
| `c1valg <> 0` and no checkboxes selected (`val1`, `val2`, `val3` all zero) | LA912C is called with empty procedure flags; LA912C may run with no-op effects |
| F10 pressed | `w_vlg1 = '1'` — procedures are submitted to a job queue via LA912C |
| ENTER pressed | Procedures run interactively through LA912C |

### LA940R — Purchase order line update from order header

| Condition | Effect |
|-----------|--------|
| LOHEPF row not found for `LODTPF.LDNUMM + LDSUFF` | Order line is **skipped** — no update performed for that line |
| `LOHEPF.LOLELE <> 0` | `LODTPF.LDLDOR` is updated from `LOLELE` (alternate supplier), not from `LOLDOR` |
| `LOHEPF.LOLELE = 0` | `LODTPF.LDLDOR` is updated from `LOLDOR` (primary supplier) |
| All LODTPF rows processed | LODTPF fields LDOTYP, LDLAGE, LDAVDE, LDLDOR are updated from the corresponding LOHEPF header values |

---

## 3. Configuration and Authorization Rules

- **LDA `l_user`** (pos 911–920): Used in LA900R to populate audit fields (`RKESIG`, `RKNSIG`, etc.) on the created standard records. Without a valid user, audit fields will be blank.
- **LA900R CTDATA arrays**: The standard values for users (AUSRPF), library list entries (ALIBPF), print routines (AFORPF), and number series (ANUMPF) are defined as compile-time arrays embedded at the end of LA900R using `CTDATA`. These arrays are compiled into the program object and cannot be changed without recompiling LA900R.
- **LA900R standard record set**: Creates exactly 1 standard user, 3 library list entries, 7 print routines, 9 number series, a set of line types (VLTYPF), a set of order types (VOTYPF), and 1 status code record in LSTSPF. The exact values are determined by the CTDATA arrays.
- **LA400R authorization**: LA400R runs bulk UPDATE statements across 22 tables with no user-level authorization check in the RPG logic. Access control must be enforced at the OS/400 object authority level on the menu option or submitted job that calls LA400R. Unrestricted access to LA400R would allow any user to reassign all warehouse data to a different firm number.
- **LA910R job queue submission**: When F10 is pressed, LA912C is called with `w_vlg1 = '1'`, which causes the year-end procedures to be submitted as a batch job. The job queue must be configured and available for batch submission. If the job queue is held or not found, LA912C will fail to submit.

---

## 4. Financial / Transactional Rules

- **LA400R firm number cascade**: The bulk UPDATE covers the following 22 tables: LBCHPF, LBDTPF, LBHEPF, LDELPF, LEDIPF, LFDTPF, LFHEPF, LFTXPF, LLDIPF, LLDTPF, LLHEPF, LODTPF, LOHEPF, LOTXPF, LPLLPF, LPLOPF, LPM1PF, LSTSPF, LTELPF, LUSRPF, SIDTPF, SIHEPF. All rows in each table that have the old firm number are updated to the new firm number in a single SQL UPDATE statement. No partial-update or rollback logic is implemented.
- **LA940R propagation fields**: For each LODTPF order line with a matching LOHEPF header, the following fields are overwritten from the header: `LDOTYP` ← `LOOTYP` (order type), `LDLAGE` ← `LOLAGE` (warehouse), `LDAVDE` ← `LOAVDE` (department), `LDLDOR` ← `LOLDOR` or `LOLELE` (supplier). These fields on the order line are treated as derived from the order header and can be re-synchronized at any time by running LA940R.
- **LA900R number series initialisation**: The 9 number series created by LA900R are assigned sequential starting numbers from the CTDATA array. The ANUMPF table stores the current counter; if it already exists and overwrite is not chosen, the existing counter is preserved.

---

## 5. Status and Lifecycle Rules

- **LA101R UPSERT pattern**: Every call to LA101R results in exactly one database operation on LSTSPF: either UPDATE (if the firm row exists) or WRITE (if it does not). The screen always displays the current state first; any field the user leaves blank will overwrite the stored value with blank.
- **LA400R irreversibility**: Once LA400R completes, the firm number change is permanent. There is no undo functionality. The old firm number no longer appears on any of the 22 tables. A re-run of LA400R with swapped source/target numbers would be required to reverse the operation.
- **LA900R idempotency**: When overwrite is not selected (choice <> 1), LA900R checks for existing records before writing. If a record already exists (e.g. the number series `ANUMPF` row), it is skipped. This makes LA900R safe to re-run without destroying existing configuration, as long as choice 1 is not selected.
- **LA910R year-end procedure sequence**: LA912C implements the actual year-end logic based on the three checkbox flags (`val1`, `val2`, `val3`). The checkboxes control which sub-procedures run (e.g. carry-forward of open orders, clearing of period statistics, resetting of order counters). Without LA912C being called (i.e. when c1valg = 0), no year-end state changes occur.

---

## 6. Special Conditions

- **LA400R no blocking conditions**: The program is specifically designed as an unconditional bulk update. It contains no validation of the target firm number and no row-count check. If the new firm number already has rows in any of the 22 tables, those rows will coexist with the newly renumbered rows, potentially creating duplicates.
- **LA900R AA030C and AA034C calls**: After writing all standard data records, LA900R calls AA030C to generate an IP (Interactive Print) program and AA034C to create a corresponding OS/400 user profile. These are administrative setup steps. A failure in either call is not propagated back as a blocking error to the LA900R main program; the data writes have already been committed.
- **LA940R LOLELE override logic**: The alternate supplier field `LOLELE` on the order header takes precedence over the primary supplier `LOLDOR` when `LOLELE <> 0`. This allows the order header to redirect all its lines to an alternate supplier without having to update each line individually. LA940R propagates this decision to all matching LODTPF lines in a single pass.
- **LA910R interactive vs. batch**: The distinction between interactive (ENTER) and batch (F10) execution is controlled by the `w_vlg1` flag passed to LA912C. This single flag determines whether LA912C submits the year-end job or runs it in-stream. There is no confirmation dialog for the batch path; pressing F10 immediately submits the job.

---

## 7. Subprogram Calls

| Caller | Called Program | Purpose |
|--------|---------------|---------|
| LA900R | AA030C | Generate standard IP program for the new company |
| LA900R | AA034C | Create OS/400 user profile for the standard warehouse user |
| LA910R | LA912C | Execute year-end procedures (interactive or batch) |
| LA940R | (none — direct file I/O) | Updates LODTPF from LOHEPF directly |

---

## 8. Reference Tables

| Table | Key Fields | Purpose |
|-------|-----------|---------|
| LSTSPF | Firm | Warehouse status codes |
| AUSRPF | Firm + user code | User register for warehouse users |
| ALIBPF | Firm + sequence | Library list entries |
| AFORPF | Firm + routine code | Print routines (report definitions) |
| ANUMPF | Firm + series code | Number series counters |
| VLTYPF | Firm + line type | Order line type definitions |
| VOTYPF | Firm + order type | Order type definitions |
| LOHEPF | Firm + order number + suffix | Purchase order header |
| LODTPF | Firm + order number + suffix + line | Purchase order lines |
| LBCHPF | Firm | Batch header file (logistics) |
| LBDTPF | Firm | Batch detail file (logistics) |
| LBHEPF | Firm | Batch header extra (logistics) |
| LDELPF | Firm | Delivery file |
| LEDIPF | Firm | EDI processing file |
| LFDTPF | Firm | Freight detail |
| LFHEPF | Firm | Freight header |
| LFTXPF | Firm | Freight text |
| LLDIPF | Firm | Goods receipt detail |
| LLDTPF | Firm | Goods receipt line |
| LLHEPF | Firm | Goods receipt header |
| LOTXPF | Firm | Order text |
| LPLLPF | Firm | Pick list lines |
| LPLOPF | Firm | Pick list operations |
| LPM1PF | Firm | Purchase order master 1 |
| LTELPF | Firm | Telephone directory |
| LUSRPF | Firm + user | Warehouse-specific user settings |
| SIDTPF | Firm | Supplier invoice detail |
| SIHEPF | Firm | Supplier invoice header |
