# Statistics Parameters — Business Rules

## Introduction

The Statistics Parameters module (module prefix **SP**) maintains the parameter file `SKPMPF`, which controls filtering, grouping, and reporting dimensions for the statistics module. Parameters are stored with a type/code/value structure, where each record associates a specific dimension value (a group code, customer category, period range, item range, or order type) with a named parameter definition.

All selection sub-programs (`SP111R` through `SP151R`) validate entered values against their respective master tables before writing to `SKPMPF`. The main maintenance screen is `SP100R`.

---

## Prerequisites and Master Data Requirements

| Requirement | Table | Key Fields | Used By |
|---|---|---|---|
| Main group must exist in VAHGPF before assignment | `VAHGPF` | firm + main-group code | SP111R |
| Sub-group must exist in VAUGPF before assignment | `VAUGPF` | firm + hgrp + ugrp | SP112R |
| Top group must exist in VOGRL1 (logical on VOGRPF) | `VOGRPF` | firm + top-group code | SP114R |
| Main group and top group both required for SP115R | `VHGRL1` (logical on `VHGRPF`) | firm + top-group + main-group | SP115R |
| Customer category must exist in RA11PF | `RA11PF` | firm + category | SP121R |
| Discount category must exist in RA06PF | `RA06PF` | firm + discount category | SP122R |
| Order type must exist in VOTYPPF | `VOTYPPF` | firm + order type | SP151R |
| Both hgrp and ugrp must be supplied together for SP112R and SP115R | — | — | SP112R, SP115R |

---

## Validation Rules

### VR-01 — Main Group (Alt.) Must Exist in VAHGPF (SP111R)

Parameter type/code = 11 (alternative main group):

```
chain to VAHGPF; if not %found → *in32 = *on
```

If the entered main-group code does not exist in `VAHGPF`, `*in32` is set and the record is rejected.

*Effect*: Only main groups defined in `VAHGPF` can be assigned as alternative main-group parameters.

### VR-02 — Duplicate Check for Alt. Main Group (SP111R)

Before writing, the program chains to `SKPMLR` (logical view of `SKPMPF`) using the parameter composite key. If a record already exists for this type/code/value combination, a duplicate error is shown and the record is not written again.

### VR-03 — Sub-Group Requires Both hgrp and ugrp (SP112R)

Parameter type/code = 12 (alternative sub-group):

Both the main group code (`hgrp`) and the sub-group code (`ugrp`) must be entered. The composite value is packed into a 5-byte structure `WPVALG` before the lookup and storage. If either field is blank, the chain will not find a valid record in `VAUGPF`.

```
chain to VAUGPF; if not %found → *in32 = *on
```

The sub-group must exist in `VAUGPF` with the matching hgrp + ugrp combination. A sub-group code entered without its parent main group will fail validation.

### VR-04 — Top Group Must Exist in VOGRL1 (SP114R)

Parameter type/code = 14 (top group):

```
chain to VOGRL1; if not %found → *in32 = *on
```

The top-group code must exist in `VOGRPF` (accessed via logical `VOGRL1`). Missing top groups are rejected.

Firm break logic applies: if the parameter record on screen has a different firm, parameter code, or parameter type than the current session values, the screen resets.

### VR-05 — Main Group Requires Both hogr and hhgr (SP115R)

Parameter type/code = 15 (main group with top-group parent):

Both the top-group code (`hogr`) and the main-group code (`hhgr`) must be entered. The composite key is packed into `WPVALG` before the `VHGRL1` lookup:

```
chain to VHGRL1; if not %found → *in32 = *on
```

The main group must exist in `VHGRPF` under the specified top group. Entering a main group code without its parent top group will fail the lookup.

### VR-06 — Customer Category Must Exist in RA11PF (SP121R)

Parameter type/code = 21 (customer category):

```
chain to RA11PF; if not %found → *in32 = *on
```

The customer category code must exist in `RA11PF`. An F4 key is available to open `RA511R` for category selection.

### VR-07 — Discount Category Must Exist in RA06PF (SP122R)

Parameter type/code = 22 (discount category):

```
chain to RA06PF; if not %found → *in32 = *on
```

The discount category code must exist in `RA06PF`. An F4 key opens `RA506R` for discount category selection.

### VR-08 — Item Range From Must Not Exceed To (SP131R)

Parameter type/code = 31 (item number from/to ranges):

For each of the 10 from/to pairs in the `WWREST` structure:

```
if b2fvaX > b2tvaX → goto b2taga (skip this pair, show error)
```

The from-item value must be ≤ the to-item value. Pairs where from > to are rejected individually; other pairs in the same screen are not affected.

Additionally, a record is only written to `SKPMPF` when the to-value is non-zero. Pairs where `b2tvaX = 0` are not saved.

An F4 key is available to call `VV500R` for item number lookup.

### VR-09 — Period From Must Not Exceed To, and Must Be Valid Format (SP141R)

Parameter type/code = 41 (period from/to):

```
if b2fper > b2tper → goto b2taga
if b2fper < 195000 → goto b2taga
```

Two blocking conditions:
1. From-period must be ≤ to-period.
2. From-period must be ≥ 195000 (i.e., year ≥ 1950 in YYYYMM format). This prevents entry of invalid or implausibly early periods.

A record is only written when both `b2fper ≠ 0` and `b2tper ≠ 0`.

### VR-10 — Order Type Must Exist in VOTYPPF (SP151R)

Parameter type/code = 51 (order type):

```
chain to VOTYPPF; if not %found → *in32 = *on
```

The order type code must exist in `VOTYPPF`. An F4 key opens `VA550R` for order type selection.

---

## Configuration and Authorization Rules

### CA-01 — Firm Break on Parameter Screens (SP114R and others)

Parameter screens check that the records being displayed match the session firm, parameter code, and parameter type. If any of these do not match the current session context, the screen resets to the first record for the current context. This prevents editing parameters belonging to a different firm or parameter group.

### CA-02 — Type/Code Structure for SKPMPF

All parameters are stored in `SKPMPF` using a type/code/value structure:

| SP Program | Type | Code | Dimension |
|---|---|---|---|
| SP111R | configured | 11 | Alternative main group |
| SP112R | configured | 12 | Alternative sub-group (hgrp + ugrp packed) |
| SP114R | configured | 14 | Top group |
| SP115R | configured | 15 | Main group + top group packed |
| SP121R | configured | 21 | Customer category |
| SP122R | configured | 22 | Discount category |
| SP131R | configured | 31 | Item from/to ranges (up to 10 pairs) |
| SP141R | configured | 41 | Period from/to |
| SP151R | configured | 51 | Order type |

The type code determines which master table is used for validation and which display screen manages the records.

### CA-03 — Multi-Value Storage for Ranges (SP131R, SP141R)

Up to 10 from/to pairs can be stored for item ranges (SP131R) in a single parameter entry. Each pair occupies its own `SKPMPF` record. The screen array `WWREST` is built by reading all existing records for the current parameter key before display.

---

## Financial and Transactional Rules

There are no monetary or pricing calculations in this module. The module manages configuration parameters only.

---

## Status and Lifecycle Rules

### SL-01 — Parameter Records Are Replaced, Not Versioned

Saving a parameter record with the same type/code/value key overwrites the existing record (via update or delete/re-insert depending on the program). There is no version history or audit trail for parameter changes.

### SL-02 — F4 Inquiry Programs

Each selection screen provides an F4 lookup to a selection popup:

| Program | F4 Target | Purpose |
|---|---|---|
| SP121R | `RA511R` | Customer category selection |
| SP122R | `RA506R` | Discount category selection |
| SP131R | `VV500R` | Item number selection |
| SP151R | `VA550R` | Order type selection |

If a lookup sub-program is not available, the user must enter codes manually without interactive lookup support.

---

## Special Conditions

### SC-01 — Composite Key Packing for Sub-Group and Main-Group Pairs

For SP112R (sub-group, type 12) and SP115R (main group, type 15), the two component codes are packed into a single 5-byte value `WPVALG` before lookup and storage. The packing format is a binary/decimal overlay. Incorrect field lengths in the overlay definition will produce incorrect key values silently.

### SC-02 — No Deletion Validation in SP114R or SP121R

The reviewed programs do not check whether a parameter value is currently in use by any active statistics report or calculation before deletion. Deleting a parameter value that is referenced by a running statistics report may cause that report to produce incomplete results.

---

## Subprogram Calls Affecting Logic

| Program | Called Sub-Program | Purpose | Failure Effect |
|---|---|---|---|
| SP121R | `RA511R` | Customer category F4 lookup | Manual entry only; no validation popup |
| SP122R | `RA506R` | Discount category F4 lookup | Manual entry only; no validation popup |
| SP131R | `VV500R` | Item number F4 lookup | Manual entry only; no validation popup |
| SP151R | `VA550R` | Order type F4 lookup | Manual entry only; no validation popup |

---

## Reference Tables

| Table | Description | Key Fields |
|---|---|---|
| `SKPMPF` | Statistics parameter file | firm + type + code + value |
| `VAHGPF` | Alternative main group | firm + group code |
| `VAUGPF` | Alternative sub-group | firm + hgrp + ugrp |
| `VOGRPF` | Top group | firm + top-group code |
| `VHGRPF` | Main group | firm + top-group + main-group |
| `RA11PF` | Customer category register | firm + category code |
| `RA06PF` | Discount category register | firm + discount category code |
| `VOTYPPF` | Order type master | firm + order type code |
| `VVARPF` | Item master | firm + item number |
