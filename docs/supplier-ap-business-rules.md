# Business Rules: Supplier / Accounts Payable (RL Module)

**System:** ASOFAK
**Module Prefix:** RL
**Programs Analyzed:** RL100R, RL101R, RL108R, RL110R, RL120R, RL400R, RL500R, RL501R, RL510R, RL600R, RL620R
**Date:** 2026-03-16

---

## 1. Prerequisites / Master Data Requirements

- Supplier master records are stored in `RLEVPF`. All AP transactions, inquiry, and reporting programs require suppliers to exist in `RLEVPF` before any operation can be performed on them.
- Currency codes used in supplier locked-currency periods (`RL120R`) must be defined in the system currency table. The exchange rate (`c2vkur`) must be non-zero for the period to be saved.
- Billing codes validated in RL110R must exist in the billing code table (accessed via RS701R). Invalid billing codes block posting line saves.
- Department codes validated in RL110R must exist in the department table (accessed via RS707R). Invalid department codes block posting line saves.
- For supplier movement/merge (RL108R), the target supplier number must not already exist in `RLEVPF`. If it exists, the move is blocked. The existence check is performed before any data is moved.
- RL109R is the validation subprogram used by RL108R to verify that all tables affected by a supplier number change can be cleanly updated. RL109R must return `w_stat = '0'` (no conflict) for the move to proceed.
- The `TESTSPERRE` mechanism (accessed via a configuration check for `'SPERRE_FLYTTING_KUND/LEV'`) controls whether supplier movement is globally locked. If this sperre (lock) is active, RL108R exits immediately without displaying any screen.
- For supplier locked-currency periods (RL120R), the SQL-accessible table `RA60ST` must be available for both insert and select operations (period overlap checking).

---

## 2. Validation Rules

### RL100R – Supplier Maintenance (partial analysis)
- Passive flag `RLEVPF.RLPASS = 'J'` causes the supplier to be filtered from standard supplier lists. Passive suppliers can still be accessed directly but do not appear in inquiry programs by default.
- Deletion of a supplier is blocked if the supplier has transactions in the AP ledger. The program checks for existing postings before allowing deletion.

### RL108R – Moving / Merging Suppliers
- If the `TESTSPERRE` check finds `'SPERRE_FLYTTING_KUND/LEV'` active → immediately exit. No screen is displayed and no movement is possible while this lock is set.
- If the target supplier number already exists in `RLEVPF` → `*in32` blocks the copy/move operation.
- If notes exist on the supplier being moved (checked in a notes table) → a warning message C2MSG is displayed. The user can cancel the operation at this point, but the move is not automatically blocked — it is a confirmation warning.
- RL109R is called to validate all tables that will be affected by the supplier number change. If RL109R sets `w_stat = '1'` (conflict detected) → the message "Leverandør kan ikke byttes" (supplier cannot be changed) is displayed and the move is blocked.

### RL110R – Supplier Posting Maintenance
- Due date must be a valid calendar date (validated using `*dmy test(d)`). Invalid due date → `*in32` blocks the posting line save.
- Billing code is validated via RS701R with `p_stat = 'N'` (inactive check). Invalid or inactive billing code → `*in33` and `*in81` are set, blocking the save and displaying an error message.
- Department code is validated via RS707R with `p_stat = 'N'`. Invalid or inactive department → `*in34` and `*in81` are set, blocking the save.

### RL120R – Locked Currency Codes Per Supplier
- From date (`c2fdat`) must not be 0. Zero from date → message `xc2msg1` is displayed and save is blocked.
- To date (`c2tdat`) must not be 0. Zero to date → message `xc2msg1` is displayed and save is blocked.
- An SQL query against `RA60ST` checks whether the new period overlaps with any existing locked-currency period for the same supplier and currency. If overlap detected → message `xc2msg2` is displayed and save is blocked.
- Currency exchange rate (`c2vkur`) must not be 0. Zero rate → message `xc2msg4` is displayed and save is blocked.

### RL400R – Supplier Deletion Parameters
- The "older than" date (`a2edat`) must be a valid date. Invalid date → `*in31` blocks the deletion run.
- The "added after" date (`a2ndat`) must be a valid date. Invalid date → `*in33` blocks the deletion run.
- Output type must be either `'J'` (yes/execute) or `'N'` (no/simulate). Any other value → `*in32` blocks the deletion run.

### RL620R – Supplier Register Print Parameters
- Supplier from must be ≤ supplier to. If from > to → `*in32` blocks the print run.
- Department from must be ≤ department to. If from > to → `*in33` blocks the print run.
- Category from must be ≤ category to. If from > to → `*in34` blocks the print run.
- District from must be ≤ district to. If from > to → `*in36` blocks the print run.
- Print type must be `'E'` (individual) or `'F'` (firm-level). Any other value → `*in37` blocks the print run.
- Only one extra information field can be selected at a time. If multiple extra fields are selected → `*in38` and `*in45` block the print run.
- Sort order must be a valid sort code. Invalid sort → `*in41` blocks the print run.

---

## 3. Configuration and Authorization Rules

- The `TESTSPERRE` configuration key `'SPERRE_FLYTTING_KUND/LEV'` is a global lock that prevents both customer and supplier moves/merges. When set, RL108R exits immediately. This lock is typically activated during system upgrades or data migrations to prevent concurrent structural changes.
- The passive flag (`RLEVPF.RLPASS = 'J'`) is a soft filter. Passive suppliers are excluded from RL500R's default inquiry view. Users can toggle visibility of passive suppliers in RL501R using the `w_pass` filter. The `*in55` indicator in RL500R controls the active/passive display toggle.
- RL510R supports multiple sort modes for supplier posting inquiry: by document year/number, by document number only, by supplier invoice number, by amount, and by date. The active accounting year filter limits results to one fiscal year.
- RL600R stores comparison field flags (name, address, postal number, telephone, mobile) to the LDA for the duplicate supplier detection report. These flags are set per report run and do not persist beyond the session.
- RL501R's F11 key cycles through four display formats (4 view modes), allowing users to see different combinations of supplier data fields without leaving the inquiry screen.

---

## 4. Financial / Transactional Rules

### Supplier Posting (RL110R)
- Each posting line requires a valid billing code and a valid department code. Both are validated against their respective tables (RS701R and RS707R) with inactive-code checking.
- Due dates are mandatory and must be valid calendar dates. The due date drives the aging analysis and payment selection in the AP payment run.
- Posting lines with invalid billing codes or departments are individually blocked; other lines in the same session are not affected.

### Locked Currency Periods (RL120R)
- A locked currency period defines a fixed exchange rate for a specific supplier and currency between a from-date and a to-date. This overrides the system exchange rate table (`RA16PF`) for transactions with this supplier during the period.
- Period overlap is detected by an SQL query against `RA60ST`. The overlap check uses the period dates of the new entry against all existing periods for the same supplier + currency combination.
- A zero exchange rate (`c2vkur = 0`) is not permitted. This prevents a configuration where the locked rate effectively zeroes out all currency amounts for a supplier.

### Supplier Deletion (RL400R)
- The deletion run removes supplier master records from `RLEVPF` based on age (older than `a2edat`) and creation date (added after `a2ndat`). Only suppliers matching both criteria are candidates.
- Output type `'N'` (simulate) runs the deletion logic without committing changes, producing a report of what would be deleted. Output type `'J'` executes the deletion.
- Suppliers with AP ledger transactions are not deleted (protected by the blocking logic in RL100R). RL400R only operates on suppliers that can be safely deleted.

---

## 5. Status and Lifecycle Rules

- Active suppliers have `RLEVPF.RLPASS <> 'J'`. Passive suppliers (`RLPASS = 'J'`) remain in the system but are hidden from standard inquiry and selection screens. No supplier is deleted solely because it is passive.
- The supplier number is the primary key for all AP operations. Changing a supplier number (RL108R) is a major operation that updates references across multiple tables (validated by RL109R). The operation is protected by both the `TESTSPERRE` lock and RL109R's conflict check.
- Once a supplier has AP ledger postings, deletion of the master record is blocked in RL100R. The supplier must be set to passive if it is no longer active but cannot be deleted due to transaction history.
- Locked-currency periods in `RA60ST` have a defined start and end date. Expired periods remain in the table but no longer affect transactions (date-range checking in the transaction processing programs handles this).

---

## 6. Special Conditions

- **TESTSPERRE global lock:** This mechanism (`SPERRE_FLYTTING_KUND/LEV`) blocks all supplier and customer moves simultaneously. It is a safety lock for bulk data operations. When active, RL108R silently exits — the user sees nothing, which may be confusing if the lock is unexpectedly set.
- **Notes warning on supplier move:** When notes exist on a supplier being moved, the C2MSG warning is shown but the user can choose to proceed. Notes are not automatically transferred to the new supplier number; they may be lost after the move depending on the notes table's key structure. The warning is designed to prompt manual review.
- **RL109R conflict detection:** RL109R checks multiple tables for supplier number dependencies. The exact tables checked by RL109R are defined within that program. If any table contains records that cannot be cleanly re-keyed to the new supplier number, `w_stat = '1'` is returned and the move is blocked. This is the primary safeguard for supplier merges.
- **Duplicate supplier detection (RL600R):** The comparison fields stored to LDA determine which fields are used to identify duplicate supplier records. Only one combination of fields is stored per report run. Multiple report runs with different field combinations can be used to find different types of duplicates.
- **RL501R minimum balance filter:** The `v1sald` variable in RL501R provides a minimum balance filter for the supplier posting inquiry. Suppliers with a balance below this threshold are excluded from the display. This is useful for filtering out settled accounts.
- **Period overlap SQL in RL120R:** The period overlap detection uses a direct SQL SELECT against `RA60ST` rather than RPG file I/O. This is one of the few programs in the RL module that uses embedded SQL for validation logic.

---

## 7. Subprogram Calls Affecting Logic

| Caller | Subprogram | Purpose | Blocking Effect |
|--------|-----------|---------|-----------------|
| RL108R | RL109R | Validate all tables for supplier number change | w_stat='1' blocks the move |
| RL108R | (TESTSPERRE check) | Check global move lock | Active lock → immediate exit |
| RL110R | RS701R | Billing code validation | *in33/*in81 if code invalid/inactive |
| RL110R | RS707R | Department code validation | *in34/*in81 if dept invalid/inactive |
| RL500R | AS700R | Free-text search via RLSOPF | Returns matching supplier list |
| RL501R | (internal) | Active/passive filter toggle | *in55 controls passive supplier visibility |

---

## 8. Reference Tables

| Table | Description | Key Fields Used |
|-------|------------|-----------------|
| `RLEVPF` | Supplier master | `RLFIRM`, `RLLEV`, `RLPASS` |
| `RLOSPF` | Supplier AP ledger postings | `RLFIRM`, `RLLEV` |
| `RA60ST` | Supplier locked currency periods | `RAFIRM`, `RALEV`, `RAVALUTA`, `RAFDAT`, `RATDAT` |
| `RA16PF` | Currency exchange rates | `RAFIRM`, `RAVALUTA` |
| `RA03PF` | Payment terms | `RAFIRM`, `RABETA` |
| `AUSRPF` | User authorizations | `AUFIRM`, `AUUSER` |
| `RLSOPF` | Supplier search index | `RLFIRM`, `RLLEV` |
