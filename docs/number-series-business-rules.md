# Number Series – Business Rules

**Module prefix:** AS
**System:** ASPGPL
**Focus:** What blocks or prevents allocation of numbers from a series, and what governs suffix, week-date, and user-validation utilities

---

## Prerequisites / Master Data Requirements

- A number series record must exist in `ANUMPF` (number series register) before any number can be allocated. AS100R performs a cascading 3-level lookup:
  1. Key: firm + type (`ANFELL`) + fixed concept (`ANFAST`) — most specific
  2. Key: firm + type only (`ANFAST = *blank`)
  3. Key: firm only (both type and fixed concept blank)
  If none of the three lookups finds a record, `p_kode = '9'` is returned (series not found) and the caller must treat this as a fatal blocking condition.
- For automatic number allocation (`p_kode = '0'`), the `ANAUTO` field must be `'A'` or blank. If `ANAUTO = 'M'` (manual only) the series cannot be used for automatic allocation.
- For manual number validation (`p_kode = '2'`), the `ANAUTO` field must be `'M'`. A series marked automatic cannot be used for manual validation.
- User validation (AS750R): The username must exist in `AUSRPF` (logical view `AUSRL1`, keyed by firm + username). If not found → `p_retk = 'N'` is returned to the caller, blocking access.
- User validation (AS750R): If the user record is found but `ABLEVL = 0` (blocked user), `p_retk = 'S'` is returned, blocking access.

---

## Validation Rules

### AS100R – Number Series Allocation

| Condition | `p_kode` Returned | Effect |
|-----------|-------------------|--------|
| No `ANUMPF` record found at any of 3 levels | `'9'` | Series not found; caller must handle |
| `ANAUTO = 'M'` when automatic allocation requested (`p_kode = '0'`) | `'9'` (series mismatch) | Auto allocation blocked |
| `ANAUTO = 'A'` or blank when manual validation requested (`p_kode = '2'`) | `'9'` (series mismatch) | Manual validation blocked |
| Next number (`ansist + 1`) > `ANNTOM` (end of range) AND `ANWRAP <> '1'` | `'8'` | Series exhausted; no number allocated |
| Requested manual number (`p_numm`) < `ANNFOM` (start of range) | `'7'` | Number below series range; rejected |
| Requested manual number (`p_numm`) > `ANNTOM` (end of range) | `'7'` | Number above series range; rejected |
| Wrap condition: next > `ANNTOM` AND `ANWRAP = '1'` | wraps to `ANNFOM + 1` | Wrap allowed; number allocated from start of range |
| All conditions pass | `'0'` (success) | Number allocated; `ANSIST` updated in `ANUMPF` |

### AS110R / AS115R – Suffix Allocation (Sales / Purchase Orders)

| Condition | Effect |
|-----------|--------|
| Suffix 01–99 all in use (found in order header, history, or deleted-order file) | Loop continues; suffix counter wraps at 99 back to 01 |
| Suffix found in `FOHEPF` (open sales orders) or `SOHEPF` (historical invoices) or `FDELPF` (deleted orders) | Suffix rejected; next suffix tried |
| Suffix found in `LOHEPF` (open purchase orders) or `SIHEPF` (historical purchase invoices) or `LDELPF` (deleted purchase orders) | Suffix rejected (AS115R); next suffix tried |

### AS710R – Week Date Utility

| Condition | `p_stat` Returned | Effect |
|-----------|-------------------|--------|
| `p_week < 1` OR `p_week > 53` | `'1'` | Invalid week; no dates returned |
| `p_year < 1` | `'1'` | Invalid year; no dates returned |

### AS711R – Week Number for Date

| Condition | `p_stat` Returned | Effect |
|-----------|-------------------|--------|
| Date not found in any week of the year (after iterating all weeks via AS710R) | `'1'` | Week number not found |

### AS750R – User Validation

| Condition | `p_retk` Returned | Effect |
|-----------|-------------------|--------|
| Username not found in `AUSRPF` | `'N'` | User unknown; calling program must deny access |
| `ABLEVL = 0` (user blocked) | `'S'` | User blocked; calling program must deny access |
| User valid and active | `' '` (blank) | Access granted; `p_navn` (user's name) populated |

---

## Configuration and Authorization Rules

- AS100R reads the firm number from its call parameter (`p_firm`), not from the LDA. This makes it usable across firms within the same job.
- The three-level ANUMPF lookup is designed so that a generic series for the firm serves as a fallback when no type-specific or concept-specific series exists. Callers can use this to share a single sequence across many transaction types without maintaining individual series for each.
- `ANWRAP = '1'` enables circular/ring number series. When enabled, after reaching `ANNTOM` the next allocated number wraps back to `ANNFOM + 1`. This is typically used for reference numbers that recycle annually. The wrap does not check whether wrapped numbers are already in use — it is the caller's responsibility to detect duplicate numbers if wrapping is enabled.
- AS110R checks the current year's suffix files first, then the previous year's files. This two-year window prevents suffix collisions for orders that span a year boundary.
- AS700R/AS701R/AS702R (search string splitters) apply different wildcard patterns:
  - AS700R: first term gets `'%term%...%'`; used for suffix-anchored search.
  - AS701R: all non-blank terms get `'%term%...%'`; blank terms get `'%'` (match all).
  - AS702R: first term gets `'term%...'` (starts-with pattern); others trimmed or blank.
  The choice of version affects how SQL `WHERE` clauses are built by calling programs — wrong version selection can produce overly broad or overly narrow results.

---

## Financial / Transactional Rules

- AS100R updates `ANUMPF.ANSIST` (last used number) atomically when a number is successfully allocated. The update uses a `CHAIN` + `UPDATE` cycle; if the record cannot be found for update (concurrent access scenario), the allocation result would be undefined — the program does not implement explicit record locking beyond the standard IBM i file-access lock.
- For manual validation (`p_kode = '2'`), AS100R does **not** update `ANSIST`. It only validates that the requested number falls within the allowed range. The caller is responsible for tracking and storing manually assigned numbers.
- The suffix-search programs (AS110R, AS115R) do not allocate numbers — they only find the next available suffix. The actual order record must be created by the calling program using the returned suffix.

---

## Status and Lifecycle Rules

- A number series is considered **exhausted** (`p_kode = '8'`) when `ansist + 1 > anntom` and wrap is not enabled. The series must either be extended (increasing `ANNTOM`) or a new series created before further numbers can be allocated.
- A number series is considered **missing** (`p_kode = '9'`) when no matching record exists at any of the three ANUMPF key levels. This is a configuration error that must be corrected in the number series setup program.
- AS750R returns the user's display name in `p_navn` when the user is valid. The name field is populated only on successful validation — callers must not use `p_navn` when `p_retk = 'N'` or `'S'`.
- Week utilities AS710R and AS711R are stateless — they compute dates from the ISO week algorithm and return status `'1'` for any out-of-range input. They maintain no state between calls.

---

## Special Conditions

- **AS110R/AS115R suffix wrapping at 99:** The suffix loops from 01 to 99 and wraps back to 01 if all suffixes are in use. A theoretical infinite loop could occur if all 99 suffixes are genuinely in use across both the current and previous year. In practice this is prevented by the volume of orders per year being well below 99 per order number.
- **AS700R sort by length descending:** After splitting the search string into up to 5 terms, the terms are sorted longest-first before building the SQL pattern. This prioritises the most selective (longest) search terms in the resulting SQL `WHERE` clause, which improves query performance on large item or customer tables.
- **AS750R blocked user code `'S'`:** The `'S'` return code (for `ABLEVL = 0`) is distinct from `'N'` (user not found). Calling programs can use this distinction to display a specific "account suspended" message versus a generic "user not found" message.
- **ANUMPF three-level fallback:** The cascade means that changing the generic firm-level series affects all transaction types that do not have their own type-specific series. Care must be taken when modifying firm-level (`ANFELL = *blank`) series entries.

---

## Subprogram Calls Affecting Logic

| Caller | Called Program | Purpose | Blocking Effect |
|--------|---------------|---------|-----------------|
| AS711R | AS710R | Get Monday/Sunday for a week | `p_stat = '1'` from AS710R terminates the week search |
| Any transaction program | AS100R | Allocate next number | `p_kode = '9'` = no series; `p_kode = '8'` = exhausted; both block the transaction |
| Any login/access program | AS750R | Validate user | `p_retk = 'N'` or `'S'` blocks access |
| Sales order entry | AS110R | Find next available suffix | Suffix search result used for order key construction |
| Purchase order entry | AS115R | Find next available suffix | Suffix search result used for order key construction |

---

## Reference Tables

| Table | Logical View | Key Fields | Usage |
|-------|-------------|-----------|-------|
| ANUMPF | — | firm + ANFELL (type) + ANFAST (concept) | Number series register; 3-level cascade lookup in AS100R |
| AUSRPF | AUSRL1 | firm + username | User register; existence and block-status check in AS750R |
| FOHEPF | — | firm + order number + suffix | Open sales order headers; suffix conflict check in AS110R |
| SOHEPF | — | firm + year + order number + suffix | Historical sales invoice headers; suffix conflict check in AS110R |
| FDELPF | — | firm + order number + suffix | Deleted sales order headers; suffix conflict check in AS110R |
| LOHEPF | — | firm + order number + suffix | Open purchase order headers; suffix conflict check in AS115R |
| SIHEPF | — | firm + year + order number + suffix | Historical purchase invoice headers; suffix conflict check in AS115R |
| LDELPF | — | firm + order number + suffix | Deleted purchase order headers; suffix conflict check in AS115R |
