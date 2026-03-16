# Customer Bonus — Business Rules

## Introduction

The Customer Bonus module (module prefix **EK**) manages volume-based bonus agreements with customers. A bonus agreement is structured in three layers:

- **Bonus type** (`EKBTST`): defines which bonus category applies to a customer.
- **Bonus header** (`EKBOST`): links a customer to a bonus group for a specific year and period range.
- **Bonus elements** (`EKBEST`): individual percentage bands or value thresholds within a bonus header.

The module also manages bonus credit parameters (`EBKRPF`) that control how and when bonus credits are issued, and bonus group templates (`EPPSPF`) for mass-applying percentage grids across customers.

---

## Prerequisites and Master Data Requirements

| Requirement | Table | Key Fields | Used By |
|---|---|---|---|
| Bonus type must exist in bonus type master | `VBOTPF` | firm + bonus type code | EK104R, EK130R, EK600R |
| Order type must exist in order type master (for credit parameters) | `VOTYPPF` | firm + order type | EK130R |
| Item must exist in item master (for credit parameters) | `VVARPF` | firm + item number | EK130R |
| Customer category must be resolvable for report filters | `EKBTI3` (logical on `EKBTST`) | category + customer | EK601R |
| Period year must be ≥ 2000 for bonus group maintenance | — | — | EK120R |
| Bonus group master must exist for mass-update | `EPPSPF` | firm + group + year | EK120R |

---

## Validation Rules

### VR-01 — Bonus Type Must Exist in VBOTPF (EK104R)

When creating or editing a bonus type record for a customer:

```
vbotl1_key chain vbotl1
if not %found → *in31 = *on
```

If the entered bonus type code does not exist in `VBOTPF`, indicator `*in31` is set and the record is rejected with an error.

*Effect*: Only bonus types defined in the bonus type master can be assigned to customers.

### VR-02 — Bonus Element Grade Range: From ≤ To (EK110R)

When entering a bonus element grade threshold:

```
if c1ebgf > c1ebgt → *in31 = *on
```

The from-grade (`EBGF`) must not exceed the to-grade (`EBGT`). If it does, `*in31` is set and the record is rejected.

*Effect*: Bonus percentage bands must be entered with the lower threshold first.

### VR-03 — Bonus Percentage or Remark Required (EK110R)

When saving a bonus element:

```
if c2ebpr = 0 and c2emrk = *blank → *in31 = *on
```

Either a bonus percentage (`EBPR`) must be non-zero, or a remark (`EMRK`) must be present. A record with both fields empty is rejected.

*Effect*: Every bonus element must specify either a percentage or a textual remark.

### VR-04 — Year Validation for Bonus Group (EK120R)

```
if h1aarr < 2000 → *in32 = *on
```

The year for a bonus group must be 2000 or later. Earlier years are rejected.

### VR-05 — Year Validation for Bonus Header (EK100R)

```
if o1paar < 2000 → *in31 = *on
```

The year for a bonus header record must be 2000 or later.

### VR-06 — Period Range Validation: 1–12 and From ≤ To (EK100R)

- Period values must be in the range 1–12. Values outside this range are rejected.
- The from-period must not exceed the to-period. If `from_period > to_period`, the record is rejected.

### VR-07 — Bonus Credit Parameter Validations (EK130R)

The following validations apply when saving a bonus credit parameter record:

| Field | Rule | Indicator |
|---|---|---|
| Bonus type | Must exist in `VBOTPF` | `*in31` |
| Order type | Must exist in `VOTYPPF` | `*in32` |
| Period | Must not be zero | `*in33` |
| Period interval | Must not be zero | `*in33` |
| From-period | Must be ≤ to-period | `*in35` |
| Item number | Must exist in `VVARPF` | `*in34` |

Any violation sets the corresponding indicator and rejects the save.

### VR-08 — Report Filter Validations (EK600R)

When entering parameters for the bonus report:

| Field | Rule | Indicator |
|---|---|---|
| Year | If > current year, confirmation prompt is shown | (prompt, not hard block) |
| Customer category from/to | From must be ≤ to | — |
| Customer number from/to | From must be ≤ to | `*in40` |
| Bonus type | Must exist in `VBOTPF` | `*in32` |
| Detail level | Must be blank or `'1'` | `*in33` |
| Customer letter flag | Must be blank or `'1'` | `*in41` |

### VR-09 — Bonus Base Lines with Zero Base Are Excluded from Report (EK601R, v6.30)

```
if ekebyg = 0 → skip line
```

Bonus element lines where the bonus base amount (`EBYG`) is zero are excluded from the printed report. This prevents zero-value rows from inflating the report output.

---

## Configuration and Authorization Rules

### CA-01 — Firm Break on All Screens

All programs in this module read records only for the session firm (`l_firm`, LDA positions 944–946). Records from other firms are not displayed or processed.

### CA-02 — Bonus Type Filter in Report (EK601R)

If a specific bonus type (`ptbot`) is specified in the report parameters, only `EKBTST` records matching that type are included in the report run. If blank, all bonus types are included.

### CA-03 — Customer Category vs. Customer Number Routing (EK601R)

The report reads `EKBTST` using either:
- `EKBTI3` (logical by customer category) if a category filter is set, or
- `EKBTI1` (logical by customer number) if a customer number range is set.

The two paths are mutually exclusive. Specifying both a category and a customer number range will apply only one path; the other is ignored.

---

## Financial and Transactional Rules

### FT-01 — Copy Bonus Clears Target Customer's Existing Records (EK100R)

The copy-bonus function clears all existing `EKBOST`, `EKBEST`, and `EKBTST` records for the target customer before writing the copied data. This is a destructive operation: any existing bonus agreement for the target customer is permanently deleted before the copy is applied.

*Effect*: Copying a bonus to a customer that already has a bonus agreement will replace the entire agreement without warning beyond the confirmation prompt.

### FT-02 — Delete Bonus Type Cascades to All Sub-records (EK104R)

Deleting a bonus type record from `EKBTST` triggers cascade deletes through `EKBOST` (bonus headers) and `EKBEST` (bonus elements) for that bonus type and customer. All dependent records are removed.

### FT-03 — Mass-Update from Bonus Group Applies Grid Percentages (EK120R)

The mass-update function in `EK120R` reads all `EKBOST` and `EKBEST` records and applies the group percentage grid from `EPPSPF` via an array LOOKUP. Items that do not match a grid entry are left unchanged. There is no preview step; the update is applied immediately on confirmation.

### FT-04 — Year Filter Removed in v6.10 (EK601R)

Prior to version 6.10, the bonus report filtered by year. From v6.10 onwards, the year filter is no longer applied; all bonus type records are included regardless of year. The bonus type (`ptbot`) remains the primary filter.

---

## Status and Lifecycle Rules

### SL-01 — Calculation Trigger via EK710C / EK715R

Bonus calculation is triggered by calling `EK710C` (batch calculation) or `EK715R` (interactive recalculation) from within `EK100R` and `EK104R`. Calculation is not automatic on save; it must be explicitly requested. If the calculation sub-program is not called, the bonus amounts in `EKBOST` will not reflect the latest element configuration.

### SL-02 — No Approval Workflow

There is no approval or activation status on bonus agreements. Records saved to `EKBTST`, `EKBOST`, or `EKBEST` are immediately active for calculation and reporting.

---

## Special Conditions

### SC-01 — Bonus Element Composite Key Includes Full Dimension Set

The composite key for `EKBEST` (EK110R) includes all of:

- firm + bkun + rkat + kun + kpro + ogrp + hgrp + ugrp + ldor + modn + vare + enhe + lety + boty + fdat + tdat + ebgf + ebgt

Changing any dimension field creates a new record rather than updating the existing one. Users must delete the old record manually if they need to modify a dimension.

### SC-02 — F4 Inquiry Functions

Several selection screens provide F4 lookup for master data:
- `EK130R` F4 on item → `VV500R` item selection popup
- `EK600R` F4 on bonus type → `VBOTPF` lookup

If the F4 sub-program is not available, the user must enter codes manually without validation feedback until the save attempt.

---

## Subprogram Calls Affecting Logic

| Program | Called Sub-Program | Purpose | Failure Effect |
|---|---|---|---|
| EK100R | `EK120R` | Bonus group maintenance | Navigation fails |
| EK100R | `EK104R` | Bonus type detail | Navigation fails |
| EK100R | `EK710C` | Batch bonus calculation | Bonus amounts not recalculated |
| EK100R | `EK715R` | Interactive bonus recalculation | Bonus amounts not recalculated |
| EK104R | `EK101R` | Bonus element list | Navigation fails |
| EK104R | `EK710C` | Batch bonus calculation | Bonus amounts not recalculated |

---

## Reference Tables

| Table | Description | Key Fields |
|---|---|---|
| `EKBTST` | Bonus type per customer | firm + customer + bonus type |
| `EKBOST` | Bonus header | firm + customer + year + period range |
| `EKBEST` | Bonus elements (grade thresholds) | composite 18-field key |
| `EBKRPF` | Bonus credit parameters | firm + bonus type + order type |
| `EPPSPF` | Bonus group percentage grid | firm + group + year |
| `VBOTPF` | Bonus type master | firm + bonus type code |
| `VOTYPPF` | Order type master | firm + order type |
| `VVARPF` | Item master | firm + item number |
