# Order Line Processing Business Rules

## Introduction

This document describes the business rules enforced by the ASOFAK Order Line Processing module (FD prefix programs). This is the core order entry engine for the ERP system, controlling registration and maintenance of order lines for quotations and sales orders. The module handles item validation, credit limit checking, deposit/advance payment requirements, scarce item control, POS shop order validation, delivery type management, price calculation, warehouse splitting, and integration with downstream systems (bestilling/picking, ByggDok, Cobuilder). All rules described below represent conditions that **block or prevent** an operation from completing. Fields use DB2 TABLE.COLUMN notation where tables are referenced by their physical-file name.

Note: FD100R.MBR exists in both NXCLOUD/rpgsrc and NXKORR/rpgsrc. The NXKORR version contains additional change entries through version 8.10 (2025-05-16) including deposit handling improvements (7.f series), quick-wins functionality (Q series), and various bug fixes. The business rules documented here reflect the combined logic of both versions.

---

## Prerequisites and Master Data Requirements

| Master Data | Table | Key Fields | Used By |
|---|---|---|---|
| Order header | FOHEPF | FOFIRM + FONUMM + FOSUFF | FD100R, FD105R |
| Order lines | FODTPF | FDFIRM + FDNUMM + FDSUFF + FDLINE | FD100R, FD105R |
| Customer master | RKUNPF | RKFIRM + RKKUND | FD100R |
| Memo balance | RKMEL (own file) | RKFIRM + RKKUND | FD100R (v6.22+) |
| Credit limit | RUKPPF | RUFIRM + RUKUND | FD100R |
| Credit card status | RUKSPF | RUFIRM + RUKUND | FD100R |
| Item master | VVARPF | VVFIRM + VVVARE | FD100R, FD105R |
| Scarce items | JVARPF | JVFIRM + JVVARE | FD100R, FD105R |
| Delivery types | VLTPPF | VYFIRM + VYLETY | FD100R, FD105R |
| Order types | VOTYPF | VAOFIRM + VAOTYP | FD100R |
| Line types | VLTYPF | VALFIRM + VALTYP | FD105R |
| Order status | FSTSPF | FSFIRM + FSSTTS | FD100R |
| Order status v2 | FST2PF | Firm + status | FD105R (v7.fb+) |
| Deposits | SDEPPF | SDFIRM + SDNUMM + SDSUFF | FD100R, FD105R |
| Warehouse status | LSTSPF | LSFIRM + LSLAG | FD100R |
| User rights | FUSRPF | FUFIRM + FUUSER | FD100R |
| Customer project | FKPRPF | FKFIRM + FKPRNR | FD100R |
| Length specifications | FVLSPF | Firm + order + line | FD100R |
| Transport item groups | RA17PF | RAFIRM + RAGRP | FD100R |
| System parameters | AFPSPF | Firm + parameter key | FD100R |
| Module activation | AMODPF | Firm + module | FD100R |
| Department validation | AVALPF | Firm + dept | FD105R (v6.3f) |
| Warehouse register | VLAGPF | VLFIRM + VLLAG | FD105R |
| Purchase orders | LOHEPF | LOFIRM + LONORD | FD100R |
| Purchase order lines | LODTPF | LDFIRM + LDNORD + LDLINE | FD100R |
| Order misc | FODMIPF | Firm + order | FD105R (v7.10) |
| Electronic order detail | EDDTPF | Firm + EDI order | FD100R |
| VAT | VMVAPF | VMFIRM + VMVGRP | FD105R |
| POS store settings | BSTSPF | BAFIRM + BATERM | FD100R |
| Supplier | RLEVPF | RLFIRM + RLLEV | FD100R |
| Return fee register | FGEBPF | Firm + return code | FD105R (v7.04) |
| Order additional data | FOD2PF | Firm + order + line | FD105R (v7.04) |
| Price templates | RA50PF | Firm + template | FD105R (v7.04) |

---

## Validation Rules

### Customer Validation (FD100R)

| Condition | Effect |
|---|---|
| Customer (RKUNPF) not found for the order header customer number | Order line entry is blocked; no lines can be added to an order with an invalid customer |
| Customer is blocked (RKUNPF.RKSPRK ≠ `' '`) | Order line entry is blocked; a blocked customer cannot receive new order lines |

### Item Validation (FD100R, FD105R)

| Condition | Effect |
|---|---|
| Item not found in VVARPF (and not found in scarce item register JVARPF) | Item entry rejected; error displayed |
| Item found in VVARPF but deleted (flagged as deleted) | Item is displayed with `** SLETTET VARE **` in the description; the user may see the line but the system warns of a deleted item |
| Item is the SHOP standard item number (BSTSPF.BASNR, v6.3d) | Item entry blocked; standard SHOP item numbers cannot be registered as sales lines |
| Item in scarce-item register (JVARPF.FAALBE): `= 0` | No restriction |
| Item in scarce-item register (JVARPF.FAALBE): `= 1` | Warning displayed; user can continue |
| Item in scarce-item register (JVARPF.FAALBE): `= 2` or `= 3` | Item entry blocked; user cannot proceed |

### Delivery Type Validation (FD105R)

| Condition | Version | Effect |
|---|---|---|
| Delivery type changed after a purchase order or purchase order suggestion has been created for the line (LOHEPF/LODTPF records exist with LFDTL6 flag) | v6.2g | Delivery type change blocked; message displayed |
| Delivery type changed when accumulation type (VAOAKK) >= 2 | v6.2i | Warning displayed; user must confirm |

### Department Validation (FD105R, v7.2B)

If department validation is enabled (AVALPF records exist for the firm), the department code entered on the order line must be found in AVALPF. If not found, the save is blocked.

### Line Type Restrictions (FD105R, v6.3f)

If the item's line type is `'L'` (length goods) or `'S'` (special), the item text on the line cannot be edited by the user. The text is protected and locked to the item description from VVARPF. This restriction is controlled by a flag checked against AVALPF.

### Warehouse Validation (FD105R, v6.2c)

If a warehouse change is requested and inventory checking is active (FAALBE parameter), the new warehouse must exist in VLAGPF. If the warehouse is not found in VLAGPF, the warehouse change is blocked.

---

## Configuration and Authorization Rules

### Credit Limit Checking (FD100R)

The system performs credit limit checking based on:
1. The customer's credit limit (RUKPPF.RUGRNS)
2. The memo balance (RKMEL) — the outstanding balance of all open orders and invoices for the customer
3. VAT-inclusive memo balance (from v5.53, to improve accuracy)

**Credit limit lock (v6.13):** If the credit limit is locked (RUKPPF field for credit lock), no new order lines can be added regardless of the actual balance. The lock overrides all other credit checks.

**Memo balance days (v7.13):** From version 7.13, the memo balance is calculated using a rolling number of days (fetched from AFPSPF) rather than a fixed total. This "flytende memosaldo" (floating memo balance) is parameterised.

**Credit limit override at quick-close (Q.081, Q.082, Q.083):** When using quick-close actions (quick wins), credit limit is also checked before the order is confirmed. The order total must be calculated and added to the memo balance before the credit check runs (v Q.083). If the credit limit is exceeded and the user lacks override rights, the quick-close is blocked.

### Commons/Almenning Discount Limit (FD100R, v5.60)

A limit is enforced on commons discount amounts (almenningsrabatter). If an order line would exceed the configured commons limit, a warning is issued. From v5.64, a blocking warning with an indicator is shown to the user when the limit is exceeded.

### User Rights (FUSRPF)

User rights control which operations the user can perform on order lines:
- Right to edit packing slip text (pakkseddel): controlled by FUSRPF. Users without this right cannot edit packing-slip text unless it is a plain text line (`v7.2l`).
- Right to override selling price / discount: controlled by FUSRPF. Users without the appropriate right cannot manually override prices.
- Right to proceed to bestilling (purchase order): controlled by FUSRPF.

### Order Split by Warehouse (FD100R, v6.23)

From version 6.23, an order can be split by warehouse when set to "ready for picking" status. This split is parameter-controlled (AFPSPF) and must be explicitly enabled. When splitting is active and the order contains lines from multiple warehouses, the system creates child orders per warehouse. Splitting is blocked if the order type does not support it or if the parameter is not set.

### Deposit/Advance Payment Requirements (FD100R, FD105R, v6.3f / 7.fb)

If a deposit is required for the customer or order (SDEPPF records), and the deposit has not been paid:
- From version 6.3f: a warning message is shown if the deposit has not been paid.
- From version 7.fb: the deposit check is enforced more strictly; the system checks whether the deposit for this order (SDEPPF keyed by firm + order number + suffix) has been fully paid (100% of required amount). If not, the order line operation is blocked.
- The deposit check is configurable (v7.04): a switch in AFPSPF controls whether the check is active. Default is `*on` (active).

**Deposit recalculation (v7.f2):** When exiting the order, if the logging code is not 1 (order not yet confirmed for fulfillment), the deposit is recalculated. If the recalculated deposit would exceed configured limits, the exit is blocked.

**Prepaid orders (v7.f4):** If the order is marked as prepaid (advance payment), adding new item lines is blocked. Only text lines can be added to a fully prepaid order.

### Scarce Item Warehouse Update (FD100R, v6.39)

If an item exists in the EVR (skafferegister / scarce item register, JVARPF), the warehouse inventory is updated even when the delivery type flag FDLVR = 1 (direct delivery). This overrides the normal direct-delivery warehouse exclusion for EVR items.

### Credit Card Module (FD100R, v6.32)

If the credit card module is not installed (AMODPF check), credit-card related fields and operations are hidden. If the module is installed, the card code is validated against the warehouse (RUKRPF/RUKSPF) and falls back to warehouse 0 if the card code is not found for the specific warehouse (v6.36).

---

## Financial and Transactional Rules

### Price Calculation (FD105R via VP900R)

Price calculation for each order line is performed by calling VP900R through the parameter record:
- Input: 120-byte record containing firm, discount category, customer, project, item, delivery type, unit, date, time, order number, price code, margin, tax-exempt flag, private flag, campaign flag, scarce flag, supplier, sale price, cost price, price group (2 chars from v8.01/v8.08)
- Output: 80-byte record containing delivery type, price code, list price, sale price, cost price, discount1, discount2, margin, spread %, campaign name, commons amount, usage-right type, discounts, VAT code, price group

The price calculation is triggered:
- At item entry (new line)
- On unit change (the price is recalculated for the new unit, v6.2f)
- On price group change
- On explicit F1 recalculate (full subfile reload from v7.01)
- Via quick-wins action for rekalkulering (v7.2i)

**DG calculation (v7.17):** From v7.17, prices are calculated via a dedicated sub-process when registering item lines, replacing some ad-hoc price fetch calls.

**Net sale price overflow prevention:** The system checks for overflow before calculating net sale price × VAT to prevent numeric field overflow on very large orders or quantities (v6.17).

### Margin Display (FD100R, v5.60/6.10)

The F1 key shows DG (dekningsgrad / margin) including customer bonus. From v7.11, the DG display including customer bonus was removed. From v7.16, the bonus text indicator is permanently suppressed.

### Order Total Calculation

The order total is maintained in FOHEPF. When adding, changing, or deleting lines, the system updates the order total. If the order total is not calculated before a credit check, the new lines may not be included (addressed in Q.083).

### Deposit Amount Calculation (FD105R, v7.fb/7.f2)

Deposit requirements are stored in SDEPPF and calculated based on the order total. The deposit percentage and minimum amount are taken from order type configuration. When the order is changed, the deposit is recalculated:
- If the new required deposit exceeds the paid deposit amount, a warning/block is generated.
- If `p_flag` = '1' (item line replacement in progress, v7.f1), the deposit check is deferred until replacement is complete.

### Return Fees (FD105R, v7.04–8.04)

From v7.04, a return fee percentage (FGEBPF) can be applied to return/reclamation order lines (line type 4). The fee is calculated as a percentage of the net amount. The return fee requires:
- FGEBPF record to exist for the return fee code.
- Customer code in RKUNPF to have the return fee flag set (v8.04, RKUNPF field).

If the return fee is not configured (AFPSPF switch, v7.06), it is not applied.

### Logging of Deleted Lines (FD100R, v5.63)

When an order line is deleted, the deletion is logged in FLOGPF with the deletion reason. From v7.08/K000, the return fee handling also logs the deletion reason when a line is created for return purposes.

---

## Status and Lifecycle Rules

### Order Status Transitions

Order lines can only be added or changed when the order is in an editable status (FSTSPF / FST2PF). The following status-based restrictions apply:

| Status Condition | Effect |
|---|---|
| Order status indicates "ready for picking" (plukking) | Adding new main order lines to picking orders is blocked (v7.14/7.15) |
| Order is of a type with delivery type 1 (direct delivery) and "ready for picking" is shown | "Set to picking" option is not displayed (v7.21) |
| Order status indicates "locked" or "sperre" | Lines cannot be modified; the order is protected |

### Transition to Bestilling (Purchase Order)

When the user selects "til bestilling" (to purchase order) in the exit screen:
- The system checks that the order type allows this transition (VOTYPF.VAOAKK field must support it).
- If `behandlingskode <> 1` (processing code is not 1), the transition to bestilling is blocked (v6.10 / 2006-03-10 fix).
- The "til bestilling" option is not shown in the exit screen if processing code is not 1.

### Order Line Replacement (FD105R, v7.26)

From v7.26, an order line's item number can be replaced. This is used primarily for electronic order confirmations where line tracking is important. During replacement (`p_flag='1'` and `p_erst` > 0), deposit checks are deferred to prevent false blocks while the replacement is in progress.

---

## Special Conditions

### ByggDok Integration (FD100R, v6.25/6.26)

If the order or customer project (FKPRPF.FKBDOK) has a ByggDok collection code, the system can trigger ByggDok data collection at the order line level. The ByggDok code is taken from the customer project. Email addresses for ByggDok/Cobuilder notifications are taken from the customer project if available, otherwise from the customer master (v6.3e).

### Cobuilder Integration (FD100R, v6.34/6.3c)

Cobuilder integration is triggered for certain item types and order contexts. A correction (v6.3c) addressed errors in the initial Cobuilder integration. No additional blocking conditions are documented beyond the standard item and order type checks.

### Trelast (Timber/Lumber) Sortiment (FD100R/FD105R, v6.3l/6.3e)

From versions 6.3l and 6.3e, items with a timber/lumber sortiment code are handled with extended logic for length specifications (FVLSPF). These items may require length specification entry (handled by the length specification module) before the line can be saved.

### NOBB Item Number Search (FD100R, v6.30/6.35)

From version 6.30, if an exact item number is not found in VVARPF, the system performs a NOBB number search. From v6.35, replaced NOBB numbers are shown to the user. The search uses VV501R (v6.3b, replacing VV500R) to prevent parameter conflicts.

### Quick Wins Interface (FD100R, Q series, v7.00+)

Quick-win actions (single-key options for common operations) were added from version Q.01 onwards:
- Q.01: Print, preview, email
- Q.02: Print
- Q.03: Set ready for picking
- Q.04: Change customer
- Q.05: Print parcel labels
- Q.06: Add transport item
- Q.07: Add item to purchase order
- Q.08: Save and exit
- Q.09: Send SMS
- Q.10: Price confirmation packing slip
- Q.11: Mark as delivered

Each quick-win action that involves a status change or commitment performs its own credit limit check and deposit check before executing. If any check fails, the quick-win action is blocked.

---

## Subprogram Calls

| Calling Program | Called Program | Purpose |
|---|---|---|
| FD100R | FD105R | Order line detail maintenance (called per line) |
| FD105R | VP900R | Price and discount calculation |
| FD105R | VL001R | Warehouse/inventory lookup |
| FD105R | VL720R | Retrieve item expiry date from warehouse |
| FD100R | FO788R (via sub, v7.12) | Called in dedicated subroutine; direct call removed |
| FD100R | AS100R | (via BU980R pattern) Order number generation for POS |
| FD100R | VV501R | NOBB item number search (v6.3b) |
| FD100R | FD106R | Item line logging |
| FD100R | FD107R | Additional order line operations |
| FD100R | FD110R | Order line transition handling |
| FD100R | FD115R | Extended order line functions |
| FD100R | FD505R | Order line inquiry/status |
| FD100R | FD510R | Order line reporting |
| FD100R | CO402R | Read memo balance configuration |

---

## Reference Tables

| Table | Description | Key Fields |
|---|---|---|
| FOHEPF | Order header | FOFIRM + FONUMM + FOSUFF |
| FODTPF | Order lines | FDFIRM + FDNUMM + FDSUFF + FDLINE |
| FODMIPF | Order misc (price log) | FDFIRM + FDNUMM + FDSUFF + FDLINE |
| FOD2PF | Order additional data (return fee) | FDFIRM + FDNUMM + FDSUFF + FDLINE |
| SDEPPF | Deposit/advance payment register | SDFIRM + SDNUMM + SDSUFF |
| RKUNPF | Customer master | RKFIRM + RKKUND |
| RKMEL | Memo balance | RKFIRM + RKKUND |
| RUKPPF | Credit limit register | RUFIRM + RUKUND |
| RUKRPF | Credit card register | RUFIRM + RUKUND |
| RUKSPF | Card status register | RUFIRM + RUKUND |
| VVARPF | Item master | VVFIRM + VVVARE |
| JVARPF | Scarce item register (EVR) | JVFIRM + JVVARE |
| VLTPPF | Delivery types | VYFIRM + VYLETY |
| VOTYPF | Order types | VAOFIRM + VAOTYP |
| VLTYPF | Line types | VALFIRM + VALTYP |
| FSTSPF | Order status | FSFIRM + FSSTTS |
| FST2PF | Order status v2 | Firm + status |
| FUSRPF | User rights | FUFIRM + FUUSER |
| FKPRPF | Customer projects | FKFIRM + FKPRNR |
| FVLSPF | Length specifications | Firm + order + line |
| LSTSPF | Warehouse status | LSFIRM + LSLAG |
| VLAGPF | Warehouse register | VLFIRM + VLLAG |
| LOHEPF | Purchase order header | LOFIRM + LONORD |
| LODTPF | Purchase order lines | LDFIRM + LDNORD + LDLINE |
| BSTSPF | POS store settings | BAFIRM + BATERM |
| AFPSPF | System parameters | Firm + parameter key |
| AMODPF | Module activation | Firm + module |
| AVALPF | Department validation | Firm + department |
| FGEBPF | Return fee register | Firm + return code |
| FLOGPF | Order line log | Firm + order + line + sequence |
| VMVAPF | VAT register | VMFIRM + VMVGRP |
| RA17PF | Transport item groups | RAFIRM + RAGRP |
