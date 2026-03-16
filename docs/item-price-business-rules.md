# Item Price Business Rules

## Introduction

This document describes the business rules enforced by the ASOFAK Item Price module (VP5xx–VP9xx programs). The module manages item prices in VVPRPF, campaign prices in VTILPF, price book printing and reporting, batch price adjustments, and the core price/discount calculation engine VP900R. All rules described below represent conditions that **block or prevent** an operation from completing. Fields use DB2 TABLE.COLUMN notation where tables are referenced by their physical-file name.

---

## Prerequisites and Master Data Requirements

| Master Data | Table | Key Fields | Used By |
|---|---|---|---|
| Item master | VVARPF.VVVARE | VVFIRM + VVVARE | VP510R, VP520R, VP600R, VP601R |
| Item price | VVPRPF.VPVARE | VPFIRM + VPVARE + VPLDOR + VPPRGR + VPENHE + VPLETY + VPPDAT | VP510R, VP700R, VP800R, VP900R |
| Campaign price | VTILPF | VTFIRM + VTVARE | VP700R, VP900R |
| Supplier | RLEVPF.RLLEV | RLFIRM + RLLEV | VP600R |
| Delivery type | VLTPPF.VYLETY | VYFIRM + VYLETY | VP600R |
| Sortiment | VSHEPF | VSFIRM + VSSNR | VP600R |
| Overgroup | VOGRPF.VGOOGR | VGFIRM + VGOOGR | VP600R |
| Main group | VHGRPF.VGHHGR | VGFIRM + VGHHGR | VP600R |
| Subgroup | VUGRPF.VGUUGR | VGFIRM + VGUUGR | VP600R |
| Price group | RA30PF.RAPRGR | RAFIRM + RAPRGR | VP510R (v5.70+) |
| Discount category | RA06PF.RARABL | RAFIRM + RARABL | VP610R |
| Customer | RKUNPF.RKKUND | RKFIRM + RKKUND | VP610R |
| Customer project | FKPRPF.FKPRNR | FKFIRM + FKPRNR | VP610R |
| Warehouse | (via RS710R) | FIRM + WAREHOUSE | VP600R |
| Usage rights | VBRKPF | VBFIRM + VBKUND | VP900R |

---

## Validation Rules

### VP600R — Price Book Print Parameters

Validations are performed before the report program VP600C is called. Any failure blocks report generation.

| Condition | Indicator | Effect |
|---|---|---|
| Sortiment and item-group/supplier filter both specified | *in31 | Mutually exclusive filters; blocked |
| Sortiment not found in VSHEPF | *in32 | Unknown sortiment; blocked |
| Overgroup not found in VOGRPF | *in33 | Unknown overgroup; blocked |
| Main group not found in VHGRPF | *in34 | Unknown main group; blocked |
| Subgroup not found in VUGRPF | *in35 | Unknown subgroup; blocked |
| Supplier not found in RLEVPF | *in36 | Unknown supplier; blocked |
| Delivery type not found in VLTPPF | *in37 | Unknown delivery type; blocked |
| Invalid date format | *in38 | Date cannot be parsed; blocked |
| Warehouse invalid (RS710R returns failure) | *in39 | Invalid warehouse; blocked |

### VP610R — Customer Price Book Parameters

Validations mirror VP600R with added customer-specific checks:

| Condition | Effect |
|---|---|
| Discount category not found in RA06PF | Blocked |
| Customer not found in RKUNPF | Blocked |
| Customer project not found in FKPRPF | Blocked |
| Item group range invalid (FROM > TO) | Blocked |
| Supplier not found in RLEVPF | Blocked |
| Delivery type not found in VLTPPF | Blocked |
| Warehouse invalid via RS710R | Blocked |
| Invalid date format | Blocked |

### VP520R — Items with Outdated Price Date

This program uses an SQL cursor joining VVARPF and VVPRPF against JVPRPF to find items whose last EVR goods-receipt date (JXGDAT in JVPRPF) is newer than their stored price date (VVPRPF.VPPDAT).

**Blocking condition:** If `VVPRPF.VPPDAT >= JXGDAT`, the item is skipped (not shown in the outdated-price list). Only items where the goods-receipt date is strictly newer than the price date are presented.

**Additional skip condition:** Items that have an expiry date in the warehouse (retrieved via VL720R) are excluded from the outdated-price list. Expiry-managed items are handled by a separate process.

---

## Configuration and Authorization Rules

### Supplier Selection for Price Maintenance (VP505R)

VP505R is invoked when a user opens item price maintenance for a given item. The program counts the distinct supplier records in VVPRPF for that item:

- If **only one supplier exists**, the program bypasses the selection subfile, calls VP110R directly with that supplier, and exits. The user never sees the supplier list.
- If **more than one supplier exists**, the subfile is displayed and the user must choose.

This means item price maintenance is blocked from opening in multi-supplier mode unless a supplier selection is made.

### Supplier Selection for Price Provider (VP507R)

VP507R always displays the subfile, even when only one supplier is present. It is used in contexts where the price provider name (`p_lnav`) must be returned to the caller. The distinction from VP505R is that VP507R never bypasses the selection step.

---

## Financial and Transactional Rules

### Price Hierarchy and Fallback (VP700R, VP601R, BU725R)

When retrieving a sale price for an item, the system applies a three-level supplier fallback:

1. **Warehouse supplier** — price for the supplier linked to the specific warehouse (retrieved via VL721R)
2. **Main supplier** — the item's primary supplier (VVPRPF.VPLDOR = main supplier)
3. **Default record** — VVPRPF.VPLDOR = 0 (supplier-independent fallback)

If a price is not found at level 1, the system tries level 2, then level 3. If no price is found at any level, no price is returned and the calling program must handle the absence.

### Campaign Price Conditions (VP700R, VP900R)

A campaign price from VTILPF is applied only when **all** of the following are true:
- The current date falls within VTILPF.VTFDAT (from date) and VTILPF.VTTDAT (to date), inclusive.
- The current time falls within VTILPF.VTFTIM (from time) and VTILPF.VTTTIM (to time), inclusive.

If either the date or time window is not met, the campaign price is not applied and the standard price hierarchy is used.

### Margin Calculation (VP510R)

The margin (dekningsgrad) for each price line is calculated as:

```
b1dekn = 100 - (b1kopr * 100 / b1sapr)
```

If cost price (`b1kopr`) is zero, the system falls back to:
1. The item default margin (`VVPRPF.VVDEKN`) if available.
2. The subgroup default margin (`VUGRPF.VGUDEK`) if item default is also zero.

This fallback ensures a margin figure is always displayed, but the underlying price record is not blocked — it is displayed with the fallback margin.

### Batch Price Adjustment (VP800R)

VP800R applies percentage adjustments to VVPRPF records for a given item and firm. Adjustable price types:

| Parameter | Price Field Adjusted | Condition to Apply |
|---|---|---|
| `p_spro` | VVPRPF.VPSAPR (sale price) | `p_spro <> 0` |
| `p_kpro` | VVPRPF.VPKOPR (cost price) | `p_kpro <> 0` |
| `p_ipro` | VVPRPF.VPINPR (internal price) | `p_ipro <> 0` |
| `p_bpro` | VVPRPF.VPBUPR (budget/base price) | `p_bpro <> 0` |
| `p_dekn` | VVPRPF.VPSAPR (derived from margin) | `p_dekn <> 0 AND VPKOPR <> 0` |

**Blocking condition — no-change skip:** If after applying all adjustments the new prices are identical to the existing prices (`w_sapr = vpsapr AND w_kopr = vpkopr AND w_inpr = vpinpr AND w_bupr = vpbupr`), the update is skipped entirely via GOTO end_oppd. No record write or update occurs.

**Blocking condition — zero cost for margin-based calculation:** The margin-based formula `w_sapr = (100 * vpkopr) / (100 - p_dekn)` is only applied when `p_dekn <> 0` AND `VVPRPF.VPKOPR <> 0`. If cost price is zero, margin-based recalculation is suppressed to avoid division-by-zero.

**Record creation vs. update:** VP800R chains VVPRPF by firm + item + supplier (VPLDOR) + price group (VPPRGR) + unit + delivery type + price date (`p_dato`). If no record exists at `p_dato`, a new record is written. If a record exists, it is updated. The audit fields VPEUSR (last user from LDA position 911–920), VPEDAT, VPETIM are always set on update or creation.

### Price Group Extension (v8.01, VP800R, VP510R)

From version 8.01, price group VPPRGR was extended from 1 to 2 characters. The price group is included in the composite key for both reading (vvprl1_key via VPFIRM + VPVARE) and updating (vvprlu_key via VPFIRM + VPVARE + VPLDOR + VPPRGR + VPENHE + VPLETY + VPPDAT). Any price record without a valid price group code defaults to a blank/zero value, which remains supported.

### Price Tracking via vpodat/vpotim (VP800R v6.10+)

When a **new** price record is created (indicator *in92 is on), the original creation date and time are set:
- VVPRPF.VPODAT — original creation date (`TIME` opcode)
- VVPRPF.VPOTIM — original creation time
- VVPRPF.VPOUSR — original user (from LDA)

These fields are only set on creation and are not updated on subsequent changes.

---

## Status and Lifecycle Rules

### Price Date Precedence (VP800R)

VP800R processes all price records for the given item where `w_udat >= vppdat` (current date is on or after the price effective date). For each unique combination of unit and delivery type, the first encountered record triggers an `oppdater` subroutine call. Subsequent records for the same unit+delivery-type combination are skipped (`b_frst` logic). This means only the most-current price date record per unit+delivery-type is recalculated in a single batch run.

### EVR Date Comparison (VP520R)

The comparison `vppdat >= jxgdat` (price date >= EVR goods-receipt date) determines whether an item is considered up to date. Items satisfying this condition are excluded from the outdated-price list; only items with `vppdat < jxgdat` are presented.

---

## Special Conditions

### VP900R — Core Price/Discount Calculation Engine

VP900R is called by FD105R (order line maintenance), BU920R (POS price calculator), and other programs. It receives a 120-byte input record and returns an 80-byte output record.

Input record layout (key fields):

| Offset | Field | Description |
|---|---|---|
| 1–3 | Firm | Firm code |
| 4–6 | Discount category | RKABL |
| 7–12 | Customer | RKKUND |
| 13–18 | Project | FKPRNR |
| 19–33 | Item | VVVARE |
| 34 | Delivery type | VYLETY |
| 35–37 | Unit | VVENHE |
| 38–47 | Date (ISO) | Order/price date |
| 48–55 | Time (ISO) | Order/price time |
| 56–63 | Order number | FOHORD |
| 64 | Price code | VAPKOD |
| 65–71 | Margin % | Dekningsgrad |
| 72 | Tax-exempt flag | — |
| 73 | Private flag | — |
| 74 | Campaign flag | — |
| 75 | Scarce flag | — |
| 76–81 | Supplier | VPLDOR |
| 82–89 | Sale price | VPSAPR |
| 90–97 | Cost price | VPKOPR |
| 98–99 | Price group | VPPRGR (2 chars from v8.01) |
| 100 | Loyalty club flag | — |

VP900R reads FPRIPF (special prices), FRABL (discount matrix), VTILPF (campaign prices), and VBRKPF (usage rights). No blocking of the calculation occurs within VP900R itself; it is a pure service program that returns the best applicable prices and discounts.

---

## Subprogram Calls

| Calling Program | Called Program | Purpose |
|---|---|---|
| VP505R | VP110R | Open item price maintenance for resolved supplier |
| VP520R | VL720R | Retrieve item expiry date from warehouse |
| VP600R | VP600C | Batch price book report |
| VP600R | RS710R | Validate warehouse |
| VP601R | VL721R | Retrieve warehouse-specific supplier |
| VP610R | VP610C | Batch customer price book report |
| VP700R | VL721R | Retrieve warehouse-specific supplier |
| VP800R | (none) | Direct VVPRPF read/write |
| VP900R | (none) | Pure calculation engine; reads FPRIPF, FRABL, VTILPF, VBRKPF |

---

## Reference Tables

| Table | Description | Key Fields |
|---|---|---|
| VVPRPF | Item price register | VPFIRM + VPVARE + VPLDOR + VPPRGR + VPENHE + VPLETY + VPPDAT |
| VTILPF | Campaign prices | VTFIRM + VTVARE (+ date/time window) |
| FPRIPF | Special price agreements | FPFIRM + FPPRGR + FPRABL + FPKKAT + FPKUND + FPKPRO + FPVARE + FPENHE + FPLETY |
| FRABL | Discount matrix | RBFIRM + RBKKAT + RBVKAT |
| VBRKPF | Usage rights | VBFIRM + VBKUND |
| VVARPF | Item master | VVFIRM + VVVARE |
| VSHEPF | Sortiment register | VSFIRM + VSSNR |
| VOGRPF | Item overgroup | VGFIRM + VGOOGR |
| VHGRPF | Item main group | VGFIRM + VGHHGR |
| VUGRPF | Item subgroup | VGFIRM + VGUUGR |
| RLEVPF | Supplier register | RLFIRM + RLLEV |
| VLTPPF | Delivery types | VYFIRM + VYLETY |
| VSHEPF | Sortiment | VSFIRM + VSSNR |
| RA30PF | Price groups | RAFIRM + RAPRGR |
| RA06PF | Discount categories | RAFIRM + RARABL |
| RKUNPF | Customer master | RKFIRM + RKKUND |
| FKPRPF | Customer projects | FKFIRM + FKPRNR |
| JVPRPF | EVR goods-receipt log | JXFIRM + JXVARE + JXGDAT |
