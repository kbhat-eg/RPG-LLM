# Conditions and Terms — Business Rules

## Introduction

The Conditions and Terms module (module prefix **JP**) manages supplier price conditions, discount agreements, freight terms, and NOBB contract imports. Condition records define price elements (percentage discounts, fixed deductions, net prices) that are applied to purchase orders. The module supports:

- Supplier condition headers and elements (`JBETPF`, `JRAEPF`)
- Freight condition records
- NOBB (Norwegian building products) contract imports from flat files or XML
- A discount factor calculation engine used at order entry time

---

## Prerequisites and Master Data Requirements

| Requirement | Table | Key Fields | Used By |
|---|---|---|---|
| Supplier must exist in supplier register | `JLEVPF` | firm + supplier number | JP101R, JP600R |
| File group must exist in archive parameter header | `ACPHPF` | firm + file group | JP700R |
| Supplier must be non-zero | — | — | JP101R |
| Condition line number is auto-assigned; no manual entry | `JBETPF` | line number from JP710R | JP710R |
| Activated conditions require `jpeakt = 1` | `JRAEPF.JPEAKT` | — | JP720R |

---

## Validation Rules

### VR-01 — Supplier Must Be Non-Zero (JP101R)

```
if supplier = 0 → *in31 = *on
```

Entering zero as the supplier number is rejected immediately.

### VR-02 — Supplier Must Exist in JLEVPF (JP101R)

```
chain to JLEVPF; if not %found → *in32 = *on
```

After the non-zero check, the supplier number is validated against `JLEVPF`. If no record is found, `*in32` is set and the screen displays an error.

*Effect*: Conditions can only be created for suppliers that have a master record in `JLEVPF`.

### VR-03 — Supplier Validation for Report (JP600R)

When entering report parameters:
- Supplier number is validated against `JLEVPF`. If not found → `*in31 = *on`.
- Date must be valid (internal date check) → `*in32 = *on` if invalid.
- Value flags (`valr`, `vala`, `valu`) must each be `0` or `1`:
  - `valr` out of range → `*in33 = *on`
  - `vala` out of range → `*in34 = *on`
  - `valu` out of range → `*in35 = *on`

### VR-04 — File Group Must Exist in ACPHPF for NOBB Import (JP700R)

When initiating a NOBB contract import:

```
chain to ACPHPF using file group key; if not %found → *in31 = *on
```

If the file group code does not exist in `ACPHPF`, `*in31` is set and the import is blocked.

### VR-05 — Non-Activated Discount Elements Are Skipped in Calculation (JP720R)

During discount factor calculation:

```
if b_akti = *on and jpeakt <> 1 → skip element
```

Where `b_akti` is the "check activation flag" switch. When this switch is on, only elements with `JPEAKT = 1` (activated) are included in the calculation. Non-activated elements are silently skipped.

*Effect*: A condition line that has not been activated will not contribute to the discount calculation, resulting in a higher net price than expected.

### VR-06 — Discount Factor Cap at 999.99999 (JP720R, v6.10)

The accumulated discount factor is capped at 999.99999. If the factor would exceed this value, it is set to 999.99999. This prevents arithmetic overflow in downstream price calculations.

### VR-07 — Registration Type Determines Calculation Method (JP720R)

The `JRAEPF` registration type field determines how each element is applied:

| Registration type (`REGTYP`) | Calculation method |
|---|---|
| `'1'` | Percentage discount: `price × (1 - jperab/100)` |
| `'2'` | Fixed monetary deduction: `price - jpefra` |
| `'N'` | Net price: overrides all previous calculations |

If a net price element (`'N'`) is encountered, all preceding percentage or fixed deductions are discarded and the net price is used directly.

---

## Configuration and Authorization Rules

### CA-01 — NOBB Import Routing via System Switch (JP700R)

The system switch `NOBB_KONTR_XML` is read via `CO402R`. Depending on the switch value:

- If `NOBB_KONTR_XML = '1'` (active): import routes to `JP705R` (XML-format NOBB import).
- If `NOBB_KONTR_XML = '0'` or blank: import routes to `JP700C` / `JP701R` / `JP702C` (flat-file NOBB import).

*Effect*: Changing this switch changes the file format expected for NOBB contract imports. If the switch value does not match the available import file format, the import will fail.

### CA-02 — Default NOBB Folder Construction (JP700R)

If no explicit folder is provided, the default NOBB import folder is constructed as:

```
default_folder = l_fgrp + '/NOBB'
```

Where `l_fgrp` is the file group code from the LDA (positions 931–933). If `l_fgrp` is blank, the default folder will be `/NOBB`, which may not be a valid path.

### CA-03 — Existing Conditions Marked on Supplier List (JP100R)

The supplier list screen (`JP100R`) performs a lookup in `JBETPF` for each displayed supplier. If a condition record exists, the supplier row is marked with `'X'` in the conditions column. This is a display indicator; it does not block navigation.

---

## Financial and Transactional Rules

### FT-01 — Line Number Auto-Assignment (JP710R)

Sub-program `JP710R` finds the next available line number in `JBETPF` for a given composite key (firm + ldor + prgr + ogrp + hgrp + ugrp + modn + vare + vtyp + lety + rsts). The line number is returned as `jpline + 1` (maximum existing line number plus one). If no existing lines exist, returns 1.

*Effect*: Line numbers are sequential and non-configurable. Manual assignment of line numbers is not supported.

### FT-02 — Discount Factor Calculation is Read-Only (JP720R)

`JP720R` is a pure calculation sub-program. It reads `JRAEPF` records and returns the accumulated discount factor and net price. It does not update any tables. The calling program is responsible for applying and storing the result.

### FT-03 — Net Price Overrides All Discounts (JP720R)

If any element in the condition chain has registration type `'N'` (net price), the net price from that element is returned and all accumulated percentage/fixed discounts are discarded. The element with type `'N'` must be the terminal element of the chain for predictable results.

---

## Status and Lifecycle Rules

### SL-01 — Condition Activation Status (JRAEPF.JPEAKT)

Each condition element in `JRAEPF` has an activation flag (`JPEAKT`):
- `JPEAKT = 1`: element is active and included in calculations.
- `JPEAKT ≠ 1`: element is inactive and excluded from calculations (when `b_akti` check is enabled).

There is no workflow to transition elements between active and inactive states visible in the reviewed programs. The flag is set directly on the record.

### SL-02 — NOBB Import Does Not Mark Processed Records

The NOBB import programs (`JP700C`, `JP701R`, `JP702C`, `JP705R`) read import files and create/update condition records. There is no processed/unprocessed flag written back to the import file; re-running the import will process the same file again.

---

## Special Conditions

### SC-01 — Full-Text Search on Supplier List (JP100R)

The supplier list screen performs a SQL full-text search across supplier name and address fields in `JLEVPF`. The search string is used in a dynamic SQL `LIKE` condition. A blank search returns all suppliers for the firm.

### SC-02 — Freight Conditions Are Separate from Price Conditions (JP120R)

Freight conditions are maintained separately in `JP120R` and are not included in the discount factor calculation in `JP720R`. Freight charges are applied by a different mechanism at order entry time.

### SC-03 — Condition Selection for Orders (JP505R)

`JP505R` provides a condition selection popup used at purchase order entry time. The conditions returned by this screen are subsequently passed to `JP720R` for price calculation. If the user closes the popup without selecting a condition, the order line will use the default price without any condition discount.

---

## Subprogram Calls Affecting Logic

| Program | Called Sub-Program | Purpose | Failure Effect |
|---|---|---|---|
| JP100R | `JP102R` | Condition element maintenance | Navigation fails |
| JP100R | `JP120R` | Freight condition maintenance | Navigation fails |
| JP700R | `CO402R` | Read NOBB import format switch | Defaults to flat-file import |
| JP700R | `JP705R` | XML-format NOBB import | XML import unavailable |
| JP700R | `JP700C` / `JP701R` / `JP702C` | Flat-file NOBB import | Flat-file import unavailable |
| JP720R | (none) | Pure calculation, no sub-calls | — |
| JP710R | (none) | Line number lookup, no sub-calls | — |

---

## Reference Tables

| Table | Description | Key Fields |
|---|---|---|
| `JBETPF` | Condition header | firm + ldor + prgr + ogrp + hgrp + ugrp + modn + vare + vtyp + lety + rsts + line |
| `JRAEPF` | Condition discount elements | firm + condition key + sequence |
| `JLEVPF` | Supplier register | firm + supplier number |
| `ACPHPF` | Archive parameter headers (file groups) | firm + file group |
| `VOTYPPF` | Order type master | firm + order type |
