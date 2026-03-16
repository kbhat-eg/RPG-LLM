# Weekly Forecasting — Business Rules

## Introduction

The Weekly Forecasting module (module prefix **MM**) manages item-level purchase forecasts on a weekly basis. It allows users to enter, view, and simulate forecast parameters per item, warehouse, and product-group hierarchy. A batch calculation program (`MM732R`) runs the forecast algorithm and writes results to `MPRUPF` (weekly forecast file) and `MPROPF` (purchase parameter file). A secondary batch program (`MM740R`) propagates purchase parameters back to the stock master (`VLAGPF`). The module also maintains a deviation register (`MPRAPF`) recording items where the calculated reorder quantity differs significantly from the existing value.

---

## Prerequisites and Master Data Requirements

| Requirement | Table | Key Fields | Used By |
|---|---|---|---|
| Forecast week record (MPRUPF) must use a valid year/week | `MPRUPF` | firm + vsor + ogrp + hgrp + ugrp + ldor + vark + crdo + year + week | MM102R |
| Item must exist in item master before forecast parameters can be saved | `VVARPF` | firm + item number | MM511R, MM740R |
| Warehouse must exist before stock parameter update | `VLAGPF` | firm + warehouse + item | MM740R |
| Purchase parameter record must exist in MPROPF before stock update | `MPROPF` | composite key | MM740R |
| Forecast parameters require a valid item ABC code | `VLAGPF.VVLABC` | — | MM501R |
| Cost price lookup requires VP750R service program | — | — | MM511R |
| Price group lookup requires VL712R service program | — | — | MM511R |
| Week-number calculation requires AX711R | — | — | MM501R |

---

## Validation Rules

### VR-01 — Year Range Validation (MM102R)

When entering or editing a weekly forecast record:

- If `c1aar < 2005` → indicator `*in31` is set → screen error, record rejected.
- If `c1aar > 2050` → indicator `*in31` is set → screen error, record rejected.

*Effect*: Forecast years outside 2005–2050 cannot be saved.

### VR-02 — Week Number Range Validation (MM102R)

- If `c1uke < 1` → indicator `*in32` is set → screen error.
- If `c1uke > 52` → indicator `*in32` is set → screen error.

*Effect*: Week numbers outside 1–52 cannot be saved. Week 53 is not supported.

### VR-03 — Duplicate Forecast Record Check (MM102R)

Before creating a new weekly forecast record, the program chains to `MPRULR` (logical view of `MPRUPF`) using the full composite key (firm + vsor + ogrp + hgrp + ugrp + ldor + vark + crdo + year + week). If the record already exists, the create is blocked with a duplicate error.

### VR-04 — Stock Lock-Date Blocks Parameter Update (MM740R)

When propagating purchase parameters from `MPROPF` to `VLAGPF`:

```
if vlstda > w_udat → goto neste
```

If the stock record's lock date (`VLSTDA`) is later than the current system date (`w_udat`), the item is skipped entirely — no purchase parameters are written to the stock master for that item.

*Effect*: Items with a future stock-lock date are protected from automated forecast updates.

### VR-05 — Non-Locked Items Excluded from Forecast List (MM501R)

In the interactive forecast item list:

```
if h1barl <> 0 and vvlabc <> 1 → skip item
```

Items where the warehouse buffer flag (`h1barl`) is non-zero AND the ABC code is not 1 are excluded from the forecast item selection list.

*Effect*: Only locked items (or items with ABC code 1) appear in the forecast maintenance screen.

### VR-06 — ABC Code Filter (MM501R)

The list screen supports filtering by ABC code. Items whose ABC code does not match the selected filter value are not displayed. This is a display filter, not a blocking rule for saving, but it limits what can be located for maintenance.

### VR-07 — Deviation Greater than 100% Written to Deviation Register (MM740R)

If the newly calculated reorder quantity differs from the existing value by more than 100%, the deviation is recorded to `MPRAPF` (deviation register) rather than being silently applied. The record in `VLAGPF` is still updated; the deviation register is informational only.

---

## Configuration and Authorization Rules

### CA-01 — Firm Break on All Screens

All interactive programs check that the record being processed belongs to the session firm (`l_firm` from LDA, positions 944–946). Records from other firms are never displayed or updated.

### CA-02 — Filter Parameters: vsor, ogrp, hgrp, ugrp, vare, warehouse, ABC code (MM511R)

The purchase parameter detail screen supports the following filters, all of which restrict which items are displayed and editable:

- `vsor` — sales channel
- `ogrp` — top product group
- `hgrp` — main product group
- `ugrp` — sub-product group
- `vare` — item number
- `h1lage` — warehouse
- `h1kabc` — ABC code

When a filter is set, only items meeting all filter criteria are shown. Saving a parameter record is only possible for displayed items.

### CA-03 — Multi-Sequence Positioning (MM501R, MM511R)

Both programs use SQL cursors with multiple positioning sequences (O = item, L = warehouse, A = ABC code for MM501R; C through G for MM511R). The selected sequence determines the sort order of the subfile. Changing the sequence does not block saving but changes which items are visible.

---

## Financial and Transactional Rules

### FT-01 — Unit Rounding Rules for Reorder Quantities (MM740R)

The unit of measure code from the purchase parameter (`vvenhl`) determines how reorder quantities are rounded before being written to `VLAGPF`:

| Unit of measure (`vvenhl`) | Rounding |
|---|---|
| `LM`, `PAL`, `M`, `LTR`, `KG`, `M2`, `M3` | 1 decimal place |
| All other units | Integer (no decimals) |

*Effect*: Items sold/purchased in bulk units (litres, kilograms, metres) allow fractional reorder quantities; discrete-unit items are rounded to whole numbers.

### FT-02 — Deviation Register Entry Threshold (MM740R)

A deviation of more than 100% between the old and new reorder quantity triggers a write to `MPRAPF`. The formula is:

```
deviation% = ABS(new_qty - old_qty) / old_qty * 100
```

If `old_qty = 0`, the deviation check is bypassed (division by zero avoided) and the value is written directly without a deviation record.

### FT-03 — Cost Price Lookup via VP750R (MM511R)

When displaying purchase parameter detail, the current cost price is fetched by calling `VP750R`. If `VP750R` returns no price, the cost-price field is displayed as zero. The forecast calculation itself is not blocked by a missing cost price, but the displayed margin may be misleading.

---

## Status and Lifecycle Rules

### SL-01 — Forecast Result Deletion (MM401R)

`MM401R` deletes all `MPRUPF` records matching the parameter key (firm + vsor + ogrp + hgrp + ugrp + ldor + vark + crdo). If `d_vare = 0`, then `mprul4_vark = *blank`, which broadens the delete to all item variants within the group. This operation is irreversible.

### SL-02 — Simulated Purchase Parameter Deletion (MM402R)

`MM402R` deletes all records from the `MPRWLU` logical view of `MPROPF` for the session firm. This removes the entire simulated forecast for the firm. This operation is irreversible.

### SL-03 — Deviation Register Deletion (MM403R)

`MM403R` deletes all records from `MPRALU` (logical view of `MPRAPF`) for the session firm. Once deleted, the deviation history cannot be recovered. This operation is irreversible.

### SL-04 — Stock Master Update is Non-Transactional (MM740R)

`MM740R` updates `VLAGPF` record by record without explicit transaction boundaries. If the program is interrupted mid-run, some stock records will have been updated and others will not. Re-running the program is safe because it processes all items again from the beginning.

---

## Special Conditions

### SC-01 — Option 8 from Deviation Screen Calls Stock Maintenance

**Program**: MM120R

From the deviation enquiry screen (MPRAPF view), selecting option `8` on a row calls `LL101R` (stock maintenance). This navigates away from the forecasting module. No blocking; it is a navigation aid.

### SC-02 — Warehouse Selection Popup Returns Up to 10 Warehouses (MM801R)

The warehouse selection popup (`RA10PF`) returns a maximum of 10 selected warehouses. If more than 10 warehouses need to be included in a forecast run, they must be selected in multiple passes.

### SC-03 — Week 53 Not Supported

Week number validation in `MM102R` caps at 52 (`c1uke > 52 → error`). Years with ISO week 53 cannot have forecast data entered for that week.

---

## Subprogram Calls Affecting Logic

| Program | Called Sub-Program | Purpose | Failure Effect |
|---|---|---|---|
| MM501R | `AX711R` | Week number calculation | Week display may be incorrect |
| MM501R | `LL101R` | Stock maintenance (option 8) | Navigation fails |
| MM501R | `LL502R` | Additional stock detail | Display only |
| MM501R | `MM502R` | Weekly forecast detail view | Navigation fails |
| MM511R | `VP750R` | Cost price lookup | Cost price shown as zero |
| MM511R | `VL712R` | Price group lookup | Price group field blank |
| MM511R | `MM733C` | Update stock from forecast params | Stock not updated on save |

---

## Reference Tables

| Table | Description | Key Fields |
|---|---|---|
| `MPRUPF` | Weekly forecast results | firm + vsor + ogrp + hgrp + ugrp + ldor + vark + crdo + year + week |
| `MPROPF` | Purchase parameters (simulated) | composite group + item key |
| `MPRAPF` | Deviation register | firm + item + warehouse |
| `VLAGPF` | Stock master (warehouse–item) | firm + warehouse + item |
| `VVARPF` | Item master | firm + item number |
| `RA10PF` | Warehouse register | firm + warehouse code |
