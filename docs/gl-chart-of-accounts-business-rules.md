# Business Rules: GL Chart of Accounts (RH Module)

**System:** ASOFAK
**Module Prefix:** RH
**Programs Analyzed:** RH100R, RH102R, RH103R, RH108R, RH110R, RH190R, RH400R, RH500R, RH610R, RH620R
**Date:** 2026-03-16

---

## 1. Prerequisites / Master Data Requirements

- The chart of accounts is stored in `RHOVPF`. Every GL account used in journal postings, fixed accounts (`RA04PF`), and balance sheet reports must exist in `RHOVPF` before those operations can succeed.
- Standard chart of accounts entries are stored in `RHSTPF`. RH102R can copy standard chart entries (`RHSTPF.SHKONT`) into the active chart (`RHOVPF.RHREFN`) as a reference number, enabling alignment with national accounting standards.
- The system accounting year parameter `RASTPF.RA1AAR` must be set. RH400R uses this value to block deletion of current or future year postings. RH620R uses it to warn when a report year exceeds the current year.
- VAT code validation requires that the VAT code table is populated. RH100R validates VAT codes on accounts during maintenance.
- For balance distribution (RH190R) and print (RH610R, RH620R), department and account range boundaries must be established — the from/to parameters must resolve to valid ranges in `RHOVPF`.
- The user authorization table `AUSRPF` controls department-level field protection in RH610R. If `AUSRPF.ABBAVD = 'J'`, the user is limited to a specific department and the department filter fields are protected (read-only) in the parameter screen.

---

## 2. Validation Rules

### RH190R – Balance Distribution Parameters
- From department (`a2pavf`) must be ≤ to department (`a2pavt`). If `a2pavf > a2pavt` → `*in31` blocks save.
- From account (`a2pktf`) must be ≤ to account (`a2pktt`). If `a2pktf > a2pktt` → `*in32` blocks save.
- Date entry must be valid (valid calendar date). Invalid date → `*in33` blocks save.
- Accounting year must not exceed current year (`*year`). Year > `*year` → `*in34` blocks save.
- From period (`a2ppef`) must be ≤ to period (`a2ppet`). If `a2ppef > a2ppet` → `*in35` blocks save.

### RH400R – Deletion of Postings Parameters
- Year must not be 0. Year = 0 → `*in31` blocks the deletion run.
- Year must not exceed the current calendar year (`*year`). Year > `*year` → `*in32` blocks the deletion run.
- Year must be strictly less than the current accounting year (`RASTPF.RA1AAR`). If year ≥ `RA1AAR` → `*in33` blocks the deletion run. This prevents deletion of current-year or future-year postings.

### RH610R – Chart of Accounts Print Parameters
- From department (`a2pavf`) must be ≤ to department (`a2pavt`). If `a2pavf > a2pavt` → `*in31` blocks the print run.
- From account (`a2pktf`) must be ≤ to account (`a2pktt`). If `a2pktf > a2pktt` → `*in32` blocks the print run.
- From group/section (`a2pbØf`) must be ≤ to to-group (`a2pbØt`). If `a2pbØf > a2pbØt` → `*in33` blocks the print run.
- If the user is department-limited (`AUSRPF.ABBAVD = 'J'`), the department filter fields are protected and cannot be changed. The user can only run the report for their assigned department.

### RH620R – Balance Sheet Print Parameters
- From department (`a2pavf`) must be ≤ to department (`a2pavt`). If `a2pavf > a2pavt` → `*in31` blocks the print run.
- From account (`a2pktf`) must be ≤ to account (`a2pktt`). If `a2pktf > a2pktt` → `*in32` blocks the print run.
- Date entry must be valid. Invalid date → `*in33` blocks the print run.
- If year > `*year` (current calendar year) → a warning dialog is displayed. The user must explicitly confirm with `'J'` (yes). If the user does not confirm, `*in34` blocks the print run. This is a soft block — a warning that can be overridden.
- From period (`a2ppef`) must be ≤ to period (`a2ppet`). If `a2ppef > a2ppet` → `*in35` blocks the print run.
- **Known code defect in RH620R:** The comparison `a2pbØf > a2pbØf` (self-comparison of the "from group" field against itself) sets `*in36`. This condition can never be true, meaning `*in36` is never triggered. The intended logic was likely `a2pbØf > a2pbØt` (comparing from-group to to-group), but the bug means this validation never fires.

### RH100R – Chart of Accounts Maintenance (partial analysis)
- Deleting an account that has existing postings in `RFORPF` is blocked. The program checks for transaction history before allowing deletion.
- VAT code on an account must be a valid code from the VAT code table. Invalid VAT code blocks save.
- Project flag (`RHPROJ`) and department flag (`RHAVDK`) must be valid flag values (`'J'`/`'N'`). Invalid flag values block save.

### RH108R – Moving Accounts / Postings to Cost Bearer (partial analysis)
- The cost bearer account (`konto`) must be specified. A blank `konto` sets `*in38` and blocks the operation.

### RH110R – Ledger Postings Maintenance (partial analysis)
- Billing codes entered on postings are validated. Invalid billing codes block the posting line save.
- Due dates are validated. An invalid due date blocks the posting line save.

### RH500R – Chart of Accounts Inquiry (Subprogram)
- RH500R is a read-only inquiry program used by other programs (e.g., RA104R) to validate or select accounts. It does not modify `RHOVPF`.
- When called with `p_text = '*Prosjekt'`, the inquiry is filtered to show only accounts where `RHOVPF.RHPROJ = 'J'` (project-capable accounts). This filter is optional; without it, all accounts are shown.
- Supports positioning by account number (numeric entry) or by text description (alpha entry). Both entry modes return a selected account to the calling program.

---

## 3. Configuration and Authorization Rules

- `AUSRPF.ABBAVD = 'J'` restricts a user to department-level reporting in RH610R. The department filter is set from the user's assigned department and cannot be overridden by the user. This ensures department-segregated reporting.
- The accounting year `RASTPF.RA1AAR` is the system's current active accounting year. RH400R uses this as a hard boundary: postings in the current or future year cannot be deleted under any circumstances.
- Standard chart reference numbers (`RHSTPF.SHKONT`) are copied to `RHOVPF.RHREFN` via RH102R. This is a one-time setup or periodic alignment operation. The copy does not overwrite other account fields — only the reference number is updated.
- The project flag (`RHOVPF.RHPROJ`) and department flag (`RHOVPF.RHAVDK`) are per-account configuration. They are set in RH100R and are used by RF105R (journal distribution) to clear dimensions on postings. These flags must be explicitly set; there is no default.

---

## 4. Financial / Transactional Rules

### Posting Deletion (RH400R)
- The deletion run removes all postings from `RFORPF` (and associated sub-ledger tables) for the specified year, subject to the year validation rules.
- Only postings from years strictly before the current accounting year (`RA1AAR`) can be deleted. This is an archival/purge operation for historical data.
- The year must be a positive integer (year > 0). Year = 0 is treated as an invalid entry, not as "all years."

### Balance Distribution (RH190R)
- Balance distribution copies closing balances from one accounting period to opening balances of the next. Parameters specify the department range, account range, date, year, and period range to process.
- All five range parameters must form valid (from ≤ to) ranges. A single invalid range blocks the entire distribution run.

### Balance Sheet Print (RH620R)
- The balance sheet report covers the account range, department range, and period range specified in the parameters.
- Running a balance sheet for a future year is discouraged (warning dialog) but not absolutely blocked — the user can confirm to proceed. This accommodates budget scenarios where future-year accounts may be pre-populated.

### Account Moving (RH108R)
- Moving postings to a cost bearer re-allocates existing postings from their original account to the specified cost-bearer account. The cost-bearer account must be specified; blank is not permitted.
- This is a bulk re-allocation operation. The program reads all postings in the specified range and updates the account reference.

---

## 5. Status and Lifecycle Rules

- GL accounts in `RHOVPF` do not have an explicit active/inactive status flag in the programs analyzed. Deletion is the primary lifecycle action. Deletion is blocked if postings exist for the account.
- Once a posting year is closed (year < `RA1AAR`), its postings become eligible for deletion via RH400R. Prior to that, they are protected.
- The current accounting year (`RA1AAR`) acts as the system's "open year." It can be advanced (year-end close) but the programs in this module do not perform that advancement — it is managed by a separate year-end close process.
- Standard chart entries (`RHSTPF`) are reference data only. They are never posted against. Their lifecycle is managed independently of the active chart.

---

## 6. Special Conditions

- **RH620R self-comparison bug:** The condition `a2pbØf > a2pbØf` at indicator `*in36` is a code defect. The from-group boundary is compared against itself, which is always false. The group range validation is therefore never enforced in RH620R. Any from-group value is accepted regardless of the to-group value.
- **Future-year balance sheet override:** RH620R allows a future accounting year if the user explicitly confirms the warning dialog. This is designed for budget reporting or pre-close validation. The confirmation dialog uses a single-character `'J'` input — any other character defaults to blocking (`*in34`).
- **Project-only account filter (RH500R):** When called with `p_text = '*Prosjekt'`, RH500R shows only project-capable accounts. This is used by project management screens to ensure that only accounts configured for project posting can be selected.
- **Standard chart copy (RH102R):** The copy from `RHSTPF` to `RHOVPF.RHREFN` is an alignment operation. It does not create new accounts; it only sets the reference number on existing accounts. Accounts without a standard chart equivalent retain their existing `RHREFN` value.
- **Posting deletion irreversibility:** RH400R deletes postings permanently. There is no soft-delete, recycle bin, or undo mechanism in the program. The year validation is the only safeguard.

---

## 7. Subprogram Calls Affecting Logic

| Caller | Subprogram | Purpose | Blocking Effect |
|--------|-----------|---------|-----------------|
| RH100R | (internal VAT validation) | Validates VAT code on account | Invalid VAT code blocks account save |
| RH108R | (internal) | Checks konto is specified | *in38 if konto blank |
| RH610R | AUSRPF lookup | Check department restriction | Protects dept fields if ABBAVD='J' |
| RH500R | (none – read-only) | Account inquiry for callers | Returns selected account; no blocking |
| RA104R → RH500R | RH500R | Fixed account GL validation | Account not in RHOVPF blocks fixed account save |

---

## 8. Reference Tables

| Table | Description | Key Fields Used |
|-------|------------|-----------------|
| `RHOVPF` | Chart of accounts (active) | `RHFIRM`, `RHKONT`, `RHPROJ`, `RHAVDK`, `RHREFN` |
| `RHSTPF` | Standard chart of accounts (reference) | `SHFIRM`, `SHKONT` |
| `RFORPF` | GL journal postings | `RKFIRM`, `RKBILAG`, `RKKONT` |
| `RASTPF` | System accounting parameters | `RA1AAR` (current accounting year), `RA1HKT`, `RA1HRS` |
| `RA04PF` | Fixed accounts | `RAFIRM`, `RAKODE` |
| `AUSRPF` | User authorizations / department restriction | `AUFIRM`, `AUUSER`, `ABBAVD` |
