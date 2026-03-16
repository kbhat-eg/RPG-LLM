# Business Rules: Status and Setup Codes (RA Module)

**System:** ASOFAK
**Module Prefix:** RA
**Programs Analyzed:** RA100R, RA101R, RA102R, RA103R, RA104R, RA110R, RA113R, RA120R, RA130R, RA200R, RA501R
**Date:** 2026-03-16

---

## 1. Prerequisites / Master Data Requirements

- All code maintenance programs (RA101R–RA137R) are dispatched from the RA100R main menu. Access to the menu requires the user to have a valid session with the LDA (Local Data Area) populated with firm number (l_sfir) and user ID (l_user).
- Code types accepted by RA100R are integer values 1–34 and 37. Any value outside this range causes `*in31` (invalid code type indicator) and the dispatch is blocked.
- GL account validation (RA104R fixed accounts) depends on RH500R being available as a callable subprogram. GL account codes referenced in fixed account setup must exist in `RHOVPF` (chart of accounts).
- The number series table `ANUMPF` must contain an entry for any document code (RA101R) that references a number series (`c2binu`). If the number series is specified but not found in `ANUMPF`, indicator `*in34` is set and the save is blocked.
- For price group codes (RA130R), the `RKOVPF` table stores codes under reduction code field `REDKOD`. New price group codes are saved to this table; duplicate codes must be confirmed before overwriting.
- For warehouse codes (RA110R), postal code validation is performed via RS730R. Department validation is performed via RS707R. Both must return valid results for the record to be saved.
- Distribution key percentages (RA113R) are stored in `RA13PF`. The program reads and writes monthly percentage fields `ramp01` through `ramp13`; these fields must be numeric.
- The Adra bank import (RA200R) reads from `RADRPF` (raw bank file) and writes to `RE0TPF` (bank transaction staging). Both files must be accessible and `RADRPF` must contain records matching the expected fixed-width format.

---

## 2. Validation Rules

### RA100R – Main Menu Dispatch
- Valid code type values: 1–34, 37. If `a2valg` (user-entered code type) is outside this range, `*in31` is set and dispatch is blocked.
- When `w_status = 'Sperret'` (locked), access control program AD005R is called to verify the user has elevated permission. If AD005R denies access, the maintenance screen is not presented.

### RA101R – Document Codes
- Description field must not be blank. A blank description sets `*in31` and blocks save.
- If `c2binu` (number series reference) is non-blank, `ANUMPF` must contain a record for that series. If not found, `*in34` is set and save is blocked.
- If a code already exists in `RA01PF` (duplicate), a confirmation dialog is displayed. The user must explicitly confirm to overwrite.

### RA102R – Payment Methods
- Description must not be blank → `*in31` blocks save.
- If the system switch `OKO_TILGANGSKONTROLL` is active, AD005R is called before any change is committed. If access is denied, the change is blocked.
- Duplicate code → `*in32` is set with an informational message; overwrite requires user confirmation.

### RA103R – Payment Terms
- Description must not be blank → `*in31` blocks save.
- AD005R is called for access control when the module-level lock is active.
- Duplicate code → `*in32` message displayed; explicit user confirmation required to proceed.

### RA104R – Fixed Accounts
- Every GL account number entered in the fixed account table must be validated against `RHOVPF` via RH500R. An account not found in the chart of accounts blocks the save.
- Multiple account lookup subprograms are called for different account categories (input accounts, output accounts, VAT accounts); each must resolve successfully.

### RA110R – Warehouse Codes
- Description must not be blank → `*in31` blocks save.
- Postal code is validated via RS730R. Invalid postal code → `*in32` blocks save.
- Department code is validated via RS707R. Invalid department → `*in33` blocks save.
- Duplicate warehouse code → confirmation dialog; user must confirm to overwrite.
- Region coupling (via `RA91PF`) is informational only; it does not block the save.

### RA113R – Budget/Distribution Keys
- Description must not be blank → `*in31` blocks save.
- Monthly percentages `ramp01` through `ramp13` must sum to exactly 100. Any sum other than 100 sets `*in33` and blocks save. This is a hard constraint with no override.
- In copy mode (copying an existing key to a new code), duplicate code → `*in32` blocks the copy without user confirmation option.

### RA120R – Account Groups
- Description must not be blank → `*in31` blocks save.
- Duplicate code → informational message; user can confirm to overwrite.

### RA130R – Price Group Codes
- If code is blank → silently exits without saving (no error indicator, no message).
- If the code is non-blank but its first character is a space (not left-justified) → `*in31` blocks save. Price group codes must be left-justified; no leading spaces are permitted.
- Description must not be blank → `*in31` blocks save.
- Duplicate code stored in `RKOVPF.REDKOD` → confirmation dialog before overwrite.

### RA200R – Adra Bank Import
- Records with amount = 0 are skipped entirely; they are not written to `RE0TPF`.
- Transaction type `wdsbty = 'K'` is treated as a credit (debit amount = 0, credit amount = transaction amount). All other transaction types are treated as debits.
- Sign handling: amounts with a negative sign are inverted. The program maps raw fixed-width data to the staging table fields using positional substring extraction; malformed records with non-numeric amounts produce conversion errors at runtime (no pre-validation).

---

## 3. Configuration and Authorization Rules

- The switch `OKO_TILGANGSKONTROLL` (finance/accounting access control) is checked in RA102R (payment methods). When active, every change to payment method codes requires elevated access via AD005R.
- AD005R is the central access control subprogram used across RA100R (menu dispatch when status = 'Sperret'), RA102R, and RA103R. It receives the firm, user, and program context and returns an allow/deny decision.
- RA501R (document code inquiry) is a read-only subprogram used by other programs to select a valid document code from `RA01PF`. It does not allow any modification and does not require access control beyond the calling program's context.
- Warehouse code region coupling (`RA91PF`) is display-only in RA110R. The region association is maintained in `RA91PF` and is shown as informational context when viewing a warehouse code, but creating or modifying the region coupling is handled by a separate program not analyzed here.

---

## 4. Financial / Transactional Rules

### Distribution Key Percentages (RA113R)
- Exactly 13 monthly percentage fields (`ramp01`–`ramp13`) are maintained per distribution key. Field 13 (`ramp13`) typically represents an adjustment or year-end period.
- The sum must equal exactly 100 (integer comparison). Fractional percentages are permitted as long as they sum to 100.
- Distribution keys are used by the GL journal distribution program (RF102R) to allocate amounts across accounting periods. An incorrect sum would produce unbalanced period allocations, which is why the 100% rule is a hard block.

### Fixed Accounts (RA104R)
- Fixed accounts in `RA04PF` define the default GL accounts used for automated postings (VAT, rounding, discounts, etc.). These accounts are referenced throughout the GL and order processing modules.
- Each fixed account entry links a logical account role (e.g., "input VAT account," "purchase rounding") to a specific chart-of-accounts number in `RHOVPF`. The account must exist at time of entry.

### Bank Transaction Import (RA200R)
- The import maps `RADRPF` (Adra bank format) records to `RE0TPF` (bank reconciliation staging). The amount sign determines whether the amount is posted as a debit or credit in the staging table.
- Records with amount = 0 produce no staging record. This prevents phantom zero-value bank entries from appearing in reconciliation.

---

## 5. Status and Lifecycle Rules

- Codes in `RA01PF`, `RA02PF`, `RA03PF`, `RA04PF`, `RA10PF`, `RA13PF`, `RA20PF`, and `RKOVPF` do not have an explicit active/inactive status field managed by these programs. Deletion is the primary lifecycle operation; no soft-delete or status flag is maintained in the RA code tables.
- The `w_status = 'Sperret'` state in RA100R is a session-level lock indicator, not a record-level status. It triggers the AD005R access check when a user attempts to maintain locked code types.
- Price group codes (RA130R) in `RKOVPF` are referenced by item price records in `VVPRPF`. Deleting a price group code that is still referenced by price records is not blocked by RA130R itself; referential integrity must be enforced at the application layer by the calling program or by database constraints.
- Number series entries in `ANUMPF` must be created before document codes can reference them. The dependency is one-way: RA101R validates that the number series exists, but does not create it. Number series are managed by a separate program.

---

## 6. Special Conditions

- **Distribution key field `ramp13`:** The thirteenth period field is an extension to the standard 12-month calendar. Some configurations use it for a 13th accounting period (e.g., year-end adjustments). The 100% sum rule applies to all 13 fields combined.
- **Price group left-justification rule (RA130R):** The first-character-is-space check is specifically designed to prevent codes that look identical when displayed but differ by leading spaces. This prevents lookup ambiguity in price group resolution (VL711R–VL713R).
- **Document code number series (RA101R):** Number series (`ANUMPF`) control auto-numbering of documents (orders, invoices, etc.). A document code without a number series can still exist but will not auto-number documents. The `*in34` block only applies when `c2binu` is non-blank (i.e., number series is explicitly requested but not found).
- **RA200R format dependency:** The Adra bank import uses hard-coded positional substrings to parse `RADRPF`. Any change in the Adra file format would require program modification. There is no format version check in the program.
- **Warehouse code used as inventory key:** `RA10PF.RALAGE` is the warehouse code used as a key in `VLAGPF`, `VHISPF`, and all transaction line tables. Creating a warehouse code in RA110R is a prerequisite for any inventory operation on that warehouse.

---

## 7. Subprogram Calls Affecting Logic

| Caller | Subprogram | Purpose | Blocking Effect |
|--------|-----------|---------|-----------------|
| RA100R | AD005R | Access control for locked code types | Blocks maintenance if access denied |
| RA101R | (internal) | Number series lookup in ANUMPF | *in34 if series not found |
| RA102R | AD005R | Finance access control (OKO_TILGANGSKONTROLL) | Blocks save if access denied |
| RA103R | AD005R | Access control | Blocks save if access denied |
| RA104R | RH500R | GL account lookup for fixed accounts | Blocks save if account not in RHOVPF |
| RA100R → RA104R | Multiple account lookup programs | Various account category lookups | Blocks save if any account invalid |
| RA110R | RS730R | Postal code validation | *in32 if postal code invalid |
| RA110R | RS707R | Department code validation | *in33 if department invalid |

---

## 8. Reference Tables

| Table | Description | Key Fields Used |
|-------|------------|-----------------|
| `RA01PF` | Document codes | `RAFIRM`, `RACOD1` |
| `RA02PF` | Payment methods | `RAFIRM`, `RACOD2` |
| `RA03PF` | Payment terms | `RAFIRM`, `RACOD3` |
| `RA04PF` | Fixed accounts | `RAFIRM`, `RAKODE` |
| `RA10PF` | Warehouse codes | `RAFIRM`, `RALAGE` |
| `RA13PF` | Budget/distribution keys | `RAFIRM`, `RAKODE`, `ramp01`–`ramp13` |
| `RA20PF` | Account groups | `RAFIRM`, `RAGRP` |
| `RA91PF` | Warehouse-region coupling | `RAFIRM`, `RALAGE` |
| `RKOVPF` | Reduction/price group codes | `RKFIRM`, `REDKOD` |
| `ANUMPF` | Number series definitions | `ANFIRM`, `ANSER` |
| `RHOVPF` | Chart of accounts (GL) | `RHFIRM`, `RHKONT` |
| `RADRPF` | Adra bank raw import file | (positional, firm-based) |
| `RE0TPF` | Bank transaction staging | `REFIRM`, `REBILAG` |
| `AUSRPF` | User authorizations | `AUFIRM`, `AUUSER` |
